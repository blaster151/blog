---
title: "Hello World"
date: 2026-03-26
draft: false
tags: ["meta", "intro"]
summary: "First post — kicking the tires on Hugo + PaperMod."
ShowToc: true
---

## Welcome

This is the inaugural post on **This Is A Blog Post About**. Testing out code formatting below.

### Python Example

```python
def fibonacci(n: int) -> list[int]:
    """Generate the first n Fibonacci numbers."""
    seq = []
    a, b = 0, 1
    for _ in range(n):
        seq.append(a)
        a, b = b, a + b
    return seq

print(fibonacci(10))
```

### Rust Example

```rust
fn main() {
    let nums: Vec<u64> = (0..10)
        .scan((0u64, 1u64), |state, _| {
            let val = state.0;
            *state = (state.1, state.0 + state.1);
            Some(val)
        })
        .collect();

    println!("{:?}", nums);
}
```

### What's Next

More technical deep-dives coming soon. Stay tuned!
