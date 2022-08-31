# Lifetimebound

The `Clang` lifetimebound attribute can be used to tell the compiler that a lifetime is bound to an object and potentially see a compile-time warning if the object has a shorter lifetime from the invalid use of a temporary. You can use the attribute by adding a `LIFETIMEBOUND` annotation defined in src/attributes.h

```cpp
#ifndef BITCOIN_ATTRIBUTES_H
#define BITCOIN_ATTRIBUTES_H

#if defined(__clang__)
#  if __has_attribute(lifetimebound)
#    define LIFETIMEBOUND [[clang::lifetimebound]]
#  else
#    define LIFETIMEBOUND
#  endif
#else
#  define LIFETIMEBOUND
#endif

#endif // BITCOIN_ATTRIBUTES_H
```

Clang warns if it is able to detect that an object or reference refers to another object with a shorter lifetime. For example, Clang will warn if a function returns a reference to a local variable, or if a reference is bound to a temporary object whose lifetime is not extended. By using the lifetimebound attribute, this determination can be extended to look through user-declared functions.