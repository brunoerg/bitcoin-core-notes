## Full RBF

### What is RBF?

In Bitcoin, RBF stands for Replace-by-Fee. A Bitcoin transaction can be designated as RBF in order to allow the sender to replace this transaction with another similar transaction which pays a higher fee. (River)

### What about fullrbf?

"Full RBF" means the mempool accepts RBF without replaceability signaling inspection.

### How to use it?

Start your node with `-mempoolfullrbf=1`

### Code - #25353

In `src/validation.cpp`, before:

```cpp
if (!SignalsOptInRBF(*ptxConflicting)) {
    return state.Invalid(TxValidationResult::TX_MEMPOOL_POLICY, "txn-mempool-conflict");
}
```

After:

```cpp
if (!m_pool.m_full_rbf && !SignalsOptInRBF(*ptxConflicting)) {
    return state.Invalid(TxValidationResult::TX_MEMPOOL_POLICY, "txn-mempool-conflict");
}
```

