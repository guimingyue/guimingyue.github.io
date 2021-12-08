---
layout: post
title: Rust 安全之道
category: Rust
---

## Rust 语言
Rust 语言是一种系统编程语言。
## Ownership 和 borrow

## Move 和 Copy

```rust

fn test(vec: Vec<i32>) -> Vec<i32> {
  let mut vec = vec;
  vec.push(123);
  vec
}

```

### Clone
