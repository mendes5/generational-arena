#### Changes of the fork

This fork fixes the issue [#35](https://github.com/fitzgen/generational-arena/issues/35) of the forked repository.

So that this wont compile anymore:

```rust
// Create 2 arenas of different types
let mut arena1: Arena<i32> = Arena::new();
let mut arena2: Arena<f32> = Arena::new();

// Inserts integers on one and store the indexes
let i_idx0 = arena1.insert(0);
let i_idx1 = arena1.insert(1);
let i_idx2 = arena1.insert(2);

// Inserts floats on one and store the indexes
let f_idx0 = arena2.insert(0.5);
let f_idx1 = arena2.insert(1.5);
let f_idx2 = arena2.insert(2.5);

// Indexes of the float arena can be used on the integer arena
assert_eq!(*arena1.get(f_idx0).unwrap(), 0);
assert_eq!(*arena1.get(f_idx1).unwrap(), 1);
assert_eq!(*arena1.get(f_idx2).unwrap(), 2);

// Indexes of the integer arena can be used on the float arena
assert_eq!(*arena2.get(i_idx0).unwrap(), 0.5);
assert_eq!(*arena2.get(i_idx1).unwrap(), 1.5);
assert_eq!(*arena2.get(i_idx2).unwrap(), 2.5);
```

With this patch, trying to compile this will result in errors like:

```
mismatched types

expected `i32`, found `f32`

note: expected struct `component::generational_arena::Index<i32>`
         found struct `component::generational_arena::Index<f32>`
```

Moving the indexes to the propper arenas produces the expected results:

```rust
// Indexes of the float arena can be only used in float arenas
assert_eq!(*arena1.get(i_idx0).unwrap(), 0);
assert_eq!(*arena1.get(i_idx1).unwrap(), 1);
assert_eq!(*arena1.get(i_idx2).unwrap(), 2);

// Indexes of the integer arena can be used in integer arenas
assert_eq!(*arena2.get(f_idx0).unwrap(), 0.5);
assert_eq!(*arena2.get(f_idx1).unwrap(), 1.5);
assert_eq!(*arena2.get(f_idx2).unwrap(), 2.5);
```

_original README.md:_

# `generational-arena`

[![](https://docs.rs/generational-arena/badge.svg)](https://docs.rs/generational-arena/)
[![](https://img.shields.io/crates/v/generational-arena.svg)](https://crates.io/crates/generational-arena)
[![](https://img.shields.io/crates/d/generational-arena.svg)](https://crates.io/crates/generational-arena)
[![Travis CI Build Status](https://travis-ci.org/fitzgen/generational-arena.svg?branch=master)](https://travis-ci.org/fitzgen/generational-arena)

A safe arena allocator that allows deletion without suffering from [the ABA
problem](https://en.wikipedia.org/wiki/ABA_problem) by using generational
indices.

Inspired by [Catherine West's closing keynote at RustConf
2018](https://www.youtube.com/watch?v=aKLntZcp27M), where these ideas
were presented in the context of an Entity-Component-System for games
programming.

### What? Why?

Imagine you are working with a graph and you want to add and delete individual
nodes at a time, or you are writing a game and its world consists of many
inter-referencing objects with dynamic lifetimes that depend on user
input. These are situations where matching Rust's ownership and lifetime rules
can get tricky.

It doesn't make sense to use shared ownership with interior mutability (i.e.
`Rc<RefCell<T>>` or `Arc<Mutex<T>>`) nor borrowed references (ie `&'a T` or `&'a mut T`) for structures. The cycles rule out reference counted types, and the
required shared mutability rules out borrows. Furthermore, lifetimes are dynamic
and don't follow the borrowed-data-outlives-the-borrower discipline.

In these situations, it is tempting to store objects in a `Vec<T>` and have them
reference each other via their indices. No more borrow checker or ownership
problems! Often, this solution is good enough.

However, now we can't delete individual items from that `Vec<T>` when we no
longer need them, because we end up either

- messing up the indices of every element that follows the deleted one, or

- suffering from the [ABA
  problem](https://en.wikipedia.org/wiki/ABA_problem). To elaborate further, if
  we tried to replace the `Vec<T>` with a `Vec<Option<T>>`, and delete an
  element by setting it to `None`, then we create the possibility for this buggy
  sequence:

  - `obj1` references `obj2` at index `i`

  - someone else deletes `obj2` from index `i`, setting that element to `None`

  - a third thing allocates `obj3`, which ends up at index `i`, because the
    element at that index is `None` and therefore available for allocation

  - `obj1` attempts to get `obj2` at index `i`, but incorrectly is given
    `obj3`, when instead the get should fail.

By introducing a monotonically increasing generation counter to the collection,
associating each element in the collection with the generation when it was
inserted, and getting elements from the collection with the _pair_ of index and
the generation at the time when the element was inserted, then we can solve the
aforementioned ABA problem. When indexing into the collection, if the index
pair's generation does not match the generation of the element at that index,
then the operation fails.

### Features

- Zero `unsafe`
- Well tested, including quickchecks
- `no_std` compatibility
- All the trait implementations you expect: `IntoIterator`, `FromIterator`,
  `Extend`, etc...

### Usage

First, add `generational-arena` to your `Cargo.toml`:

```toml
[dependencies]
generational-arena = "0.2"
```

Then, import the crate and use the
[`generational_arena::Arena`](./struct.Arena.html) type!

```rust
extern crate generational_arena;
use generational_arena::Arena;

let mut arena = Arena::new();

// Insert some elements into the arena.
let rza = arena.insert("Robert Fitzgerald Diggs");
let gza = arena.insert("Gary Grice");
let bill = arena.insert("Bill Gates");

// Inserted elements can be accessed infallibly via indexing (and missing
// entries will panic).
assert_eq!(arena[rza], "Robert Fitzgerald Diggs");

// Alternatively, the `get` and `get_mut` methods provide fallible lookup.
if let Some(genius) = arena.get(gza) {
    println!("The gza gza genius: {}", genius);
}
if let Some(val) = arena.get_mut(bill) {
    *val = "Bill Gates doesn't belong in this set...";
}

// We can remove elements.
arena.remove(bill);

// Insert a new one.
let murray = arena.insert("Bill Murray");

// The arena does not contain `bill` anymore, but it does contain `murray`, even
// though they are almost certainly at the same index within the arena in
// practice. Ambiguities are resolved with an associated generation tag.
assert!(!arena.contains(bill));
assert!(arena.contains(murray));

// Iterate over everything inside the arena.
for (idx, value) in &arena {
    println!("{:?} is at {:?}", value, idx);
}
```

### `no_std`

To enable `no_std` compatibility, disable the on-by-default "std" feature. This
currently requires nightly Rust and `feature(alloc)` to get access to `Vec`.

```toml
[dependencies]
generational-arena = { version = "0.2", default-features = false }
```

#### Serialization and Deserialization with [`serde`](https://crates.io/crates/serde)

To enable serialization/deserialization support, enable the "serde" feature.

```toml
[dependencies]
generational-arena = { version = "0.2", features = ["serde"] }
```
