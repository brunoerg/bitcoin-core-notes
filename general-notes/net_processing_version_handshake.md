## net_processing: the version handshake

Before having any kind of communication between two peers, it's necessary both peers have exchanged their version.

Let's go to the code to see it in practice. In `net_processing` we have a function to process every message, 
called `PeerManagerImpl::ProcessMessage`. This function has a parameter called `msg_type`, 
we use it to check the message type (obviously). One of the message types is called "version", our node won't send any data until we receives this message. Let's see, in practice, how this function 
processes a "version" message.

As every payload, the "version" one contains lots of informations. So, one of the first things we will do 
is ignoring the `addr_from`, according to [Wiki](https://en.bitcoin.it/wiki/Protocol_documentation#version), 
this field can be ignored - "It is used to be the network address of the node emitting this message, 
but most P2P implementations send 26 dummy bytes. The "services" field of the address would also be 
redundant with the second field of the version message."
```cpp
vRecv.ignore(8); // Ignore the addrMe service bits sent by the peer
```

After ignoring it, now it's time to check whether the connection is `inbound`. 

**Remember**: `inbound` they opened the connection to us, `outbound` we opened the connection to them.

If the connection isn't `inbound`, we have to call `SetServices` to update the 
services related to that address.
```cpp
if (!pfrom.IsInboundConn())
{
    m_addrman.SetServices(pfrom.addr, nServices);
}
```
----------

Now it's very important to check if that peer has the services we're expecting from, so we can know if that peer is relevant for us. If the peer doesn't offer te expected services, we're going to disconnect it.

```cpp
if (pfrom.ExpectServicesFromConn() && !HasAllDesirableServiceFlags(nServices))
{
    LogPrint(BCLog::NET, "peer=%d does not offer the expected services (%08x offered, %08x expected); disconnecting\n", pfrom.GetId(), nServices, GetDesirableServiceFlags(nServices));
    pfrom.fDisconnect = true;
    return;
}
```

`HasAllDesirableServiceFlags` calls `GetDesirableServiceFlags` which we can see
it checks, behind the services, whether the network is limited or not.
```cpp
ServiceFlags GetDesirableServiceFlags(ServiceFlags services) {
    if ((services & NODE_NETWORK_LIMITED) && g_initial_block_download_completed) {
        return ServiceFlags(NODE_NETWORK_LIMITED | NODE_WITNESS);
    }
    return ServiceFlags(NODE_NETWORK | NODE_WITNESS);
}
```

After that, `ProcessMessage` will check if the peer's version is older than the proto version. If so, we're going to disconnect the peer. Versions below 31800 are no longer supported.

```cpp
if (nVersion < MIN_PEER_PROTO_VERSION) {
    // disconnect from peers older than this proto version
    LogPrint(BCLog::NET, "peer=%d using obsolete version %i; disconnecting\n", pfrom.GetId(), nVersion);
    pfrom.fDisconnect = true;
    return;
}
```

```cpp
//! disconnect from peers older than this proto version
static const int MIN_PEER_PROTO_VERSION = 31800;
```

------------------

Next step is to check the `nonce` from the payload. This `nonce` is generated every time a `version` package is sent and we use it to detect whether we're not connecting to ourselves.

```cpp
if (!vRecv.empty()) {
    // The version message includes information about the sending node which we don't use:
    //   - 8 bytes (service bits)
    //   - 16 bytes (ipv6 address)
    //   - 2 bytes (port)
    vRecv.ignore(26);
    vRecv >> nNonce;
}
...

// Disconnect if we connected to ourself
if (pfrom.IsInboundConn() && !m_connman.CheckIncomingNonce(nNonce))
{
    LogPrintf("connected to self at %s, disconnecting\n", pfrom.addr.ToString());
    pfrom.fDisconnect = true;
    return;
}
```

Another important thing, we previously saw when someone opens a connection with us, 
we want to know their services to check if we want to stay connected with him. 
However, it's also important to that peer to know our services. It is basically "ok, you want to open a 
connection with me, let me check your services. Alright, here is mine ones".

```cpp
// Inbound peers send us their version message when they connect.
// We send our version message in response.
if (pfrom.IsInboundConn()) {
    PushNodeVersion(pfrom, *peer);
}
```
------------

Now we need to know our greatest common version, we're going to use it to check support to `BIP155`, `wtxid_relay` and others.

```cpp
// Change version
const int greatest_common_version = std::min(nVersion, PROTOCOL_VERSION);
pfrom.SetCommonVersion(greatest_common_version);
pfrom.nVersion = nVersion;

const CNetMsgMaker msg_maker(greatest_common_version);
...

if (greatest_common_version >= WTXID_RELAY_VERSION) {
    m_connman.PushMessage(&pfrom, msg_maker.Make(NetMsgType::WTXIDRELAY));
}

// Signal ADDRv2 support (BIP155).
if (greatest_common_version >= 70016) {
    // BIP155 defines addrv2 and sendaddrv2 for all protocol versions, but some
    // implementations reject messages they don't know. As a courtesy, don't send
    // it to nodes with a version before 70016, as no software is known to support
    // BIP155 that doesn't announce at least that protocol version number.
    m_connman.PushMessage(&pfrom, msg_maker.Make(NetMsgType::SENDADDRV2));
}
```

----------

Now we are going to determine whether we are going to relay transactions to that peer. Fortunately, the conditions are
well explained in the comments, see:

```cpp
// Only initialize the Peer::TxRelay m_relay_txs data structure if:
// - this isn't an outbound block-relay-only connection, and
// - this isn't an outbound feeler connection, and
// - fRelay=true (the peer wishes to receive transaction announcements)
//   or we're offering NODE_BLOOM to this peer. NODE_BLOOM means that
//   the peer may turn on transaction relay later.
if (!pfrom.IsBlockOnlyConn() &&
    !pfrom.IsFeelerConn() &&
    (fRelay || (peer->m_our_services & NODE_BLOOM))) {
    auto* const tx_relay = peer->SetTxRelay();
    {
        LOCK(tx_relay->m_bloom_filter_mutex);
        tx_relay->m_relay_txs = fRelay; // set to true after we get the first filter* message
    }
    if (fRelay) pfrom.m_relays_txs = true;
}
```

We also check if we support tx reconcilition, if so, we announce it.

```cpp
if (greatest_common_version >= WTXID_RELAY_VERSION && m_txreconciliation) {
    // Per BIP-330, we announce txreconciliation support if:
    // - protocol version per the peer's VERSION message supports WTXID_RELAY;
    // - transaction relay is supported per the peer's VERSION message (see m_relays_txs);
    // - this is not a block-relay-only connection and not a feeler (see m_relays_txs);
    // - this is not an addr fetch connection;
    // - we are not in -blocksonly mode.
    if (pfrom.m_relays_txs && !pfrom.IsAddrFetchConn() && !m_ignore_incoming_txs) {
        const uint64_t recon_salt = m_txreconciliation->PreRegisterPeer(pfrom.GetId());
        m_connman.PushMessage(&pfrom, msg_maker.Make(NetMsgType::SENDTXRCNCL, 
        TXRECONCILIATION_VERSION, recon_salt));
    }
}
```

----

And then, after all this, we finally send the `VERACK` message.

```cpp
m_connman.PushMessage(&pfrom, msg_maker.Make(NetMsgType::VERACK));
```cpp
