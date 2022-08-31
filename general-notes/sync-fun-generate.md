## Sync_fun

`generate()` is a function from `test_framework` used to generate blocks. See:

```py
def generate(self, generator, *args, sync_fun=None, **kwargs):
    blocks = generator.generate(*args, invalid_call=False, **kwargs)
    sync_fun() if sync_fun else self.sync_all()
    return blocks
```

If you don't pass anything to `sync_fun`, it calls `sync_all()`.

`sync_all()` will sync blocks and mempools among the nodes. So, to avoid it you pass `self.no_op` in `sync_fun`.