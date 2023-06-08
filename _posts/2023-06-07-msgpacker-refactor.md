---
layout: post
title: "MsgPacker: Enhancing performance and security"
---

[Rust](https://www.rust-lang.org/) is renowned for its performance and safety features, making it an ideal choice for systems programming. This blog post focuses on improving the efficiency of Rust code through a performance-driven refactor of the [msgpacker](https://crates.io/crates/msgpacker) library. By implementing a set of design decisions, significant performance gains and enhanced security were achieved.

[msgpacker](https://crates.io/crates/msgpacker) is a [MessagePack](https://msgpack.org/) implementation focused on providing a simple API without compromising functionality.

## Benchmarks

The benchmarks conducted on an `Intel(R) Core(TM) i9-9900X CPU @ 3.50GHz` showcased the superior performance of [msgpacker](https://crates.io/crates/msgpacker) compared to the well-known [rmp-serde](https://crates.io/crates/rmp-serde) library. The tests involved the serialization and deserialization of 1,000 entries of a complex structure. [msgpacker](https://crates.io/crates/msgpacker) outperformed rmp-serde by a substantial margin, exceeding 10x speed improvements.

The target structure uses randomized, standard distribution instances, defined as:

```rust
pub struct Value {
    pub t00: Option<u8>,
    pub t01: Option<u16>,
    pub t02: Option<u32>,
    pub t03: Option<u64>,
    pub t04: Option<usize>,
    pub t05: Option<i8>,
    pub t06: Option<i16>,
    pub t07: Option<i32>,
    pub t08: Option<i64>,
    pub t09: Option<isize>,
    pub t10: (),
    pub t11: PhantomData<String>,
    pub t12: Option<bool>,
    pub t13: Option<Vec<u8>>,
    pub t14: Option<String>,
}
```

The `Vec<u8>` and `String` will contain non-negligible amounts of data, ranging from 0 bytes to ~65Kb, uniformly distributed.

#### Serialization of 1.000 entries

![pdf](https://github.com/codx-dev/msgpacker/assets/8730839/d317a9c4-31ab-40f5-b96a-157ed0354cbb)
![pdf](https://github.com/codx-dev/msgpacker/assets/8730839/757fb98f-de38-4ecd-ac9d-5733a263a46b)

#### Deserialization of 1.000 entries

![pdf](https://github.com/codx-dev/msgpacker/assets/8730839/f47f2c45-b6aa-4014-8d32-9472691a6286)
![pdf](https://github.com/codx-dev/msgpacker/assets/8730839/c32809f7-6883-4c68-ab27-2fbce87dd688)

## Design decisions

The previous version of [msgpacker](https://crates.io/crates/msgpacker) [employed an intermediate structure](https://github.com/codx-dev/msgpacker/blob/078e2833ec829bd231e47951d96f9df1fb2c5e7e/msgpacker/src/message_ref.rs#L48-L75) for parsing bytes from the protocol, allowing conversion to arbitrary types. However, this approach suffered from performance issues and potential vulnerabilities due to recursive type definitions.

The previous implementation avoids unnecessary copy by using references to the original slices. This can be achieved with the temporary buffers created by [BufRead](https://doc.rust-lang.org/std/io/trait.BufRead.html) implementations. However, this will still incur overhead, as the deserialization will first be performed on this enum, and then finally translated to the target type.

This approach is somewhat naive, as it will not only underperform, but also create potential vulnerabilities as recursive types will be defined by the message. If a malicious user sends multiple, sequential recursive types (i.e. `Vec<Vec<Vec<Vec...`), each of these will recurse, and will at some point shutdown the service with a stack overflow.

To address these shortcomings, the core decision in the refactor was to eliminate the intermediate enum and shift the responsibility to the Rust concrete type representation. By aligning deserialization with Rust's static type definitions, infinite recursion attacks were rendered infeasible, and performance was significantly improved. This is a win-win decision, as production environments will never have such enums, as services expects concrete, well-formed types.

If the user has a type like this:

```rust
pub struct Foo {
    bar: Box<Self>,
}
```

It means he will have to allocate `bar` somewhere before instantiating `Foo`, and he will be constrained by the resources of the host system to do so. Therefore, a recursion attack on this type would require an unlimited amount of bytes to create such scenario, and not only a malicious message construction - deeming the attack unfeasible.

However, such intermediate type might be useful for some scenarios where the user will receive arbitrary data from I/O, and we have [rmpv](https://crates.io/crates/rmpv) that will cover such case.

## Lazy Functions for efficient serialization

Rust structures consist of atomic types, and by covering the most atomic types, complex dependent types can be covered automatically. To minimize boilerplate code, a derive procedural macro called `MsgPacker` is provided. Users can implement encoding traits for types that contain attributes implementing encoding and decoding. The `#[msgpacker(map)]` and `#[msgpacker(array)]` attributes signal that a field is a map or an array, respectively.

This is somewhat similar to serde, but far simpler. serde attempts to create a general framework to signal types that can be serializable, and how the serialization engine show interface with such types. In [msgpacker](https://crates.io/crates/msgpacker), we attempt a much simpler approach to optimize for ad-hoc serialization strategy. Often, general serialization greatly increases the complexity of the implementation, as the implementer will have to waste both abstraction & cycles with design traits that aren't part of the domain of the specific problem he is solving. One example of this phenomena happens [here](https://github.com/3Hren/msgpack-rust/blob/f4ad0d0257edfea460eb176cdb7d11ddfe97ba3b/rmp-serde/src/config.rs#L99-L101), where the implementer needs to specify boilerplate configuration on whether or not the format is human readable - and MessagePack is not! This example can be augmented to other parts of the code that, at high level of confidence, is the responsible for the performance difference between the two implementations.

## Raw types vs Collections

MessagePack deals in very simple terms - it will encode efficiently numbers, strings, arrays, and maps. Numbers and strings are trivial to be represented as lazy load, on-demand functions, but arrays and maps will require some minor tweaks.

To handle collections, [msgpacker](https://crates.io/crates/msgpacker) leverages Rust's iterator-based representation. Collections implementing the map-like or array-like iterator interface can be encoded and decoded seamlessly. The library provides functions such as `pack_map` and `unpack_map` that accept and return any type adhering to the map-like interface. Additionally, a derive macro attribute simplifies the implementation for types containing map or array attributes.

Rust represents collections as iterators. All the commonly used collections implements iterators of items for array-like implementations, and iterators of key-value pairs for map-like ones. We can take advantage of this fact and abuse this as generic, so we can cover any type that follows this pattern.

We can see an example with [pack_map](https://github.com/codx-dev/msgpacker/blob/65a0ab07f835d765d9b07cfe91e4b9207ef339ff/msgpacker/src/pack/collections.rs#L34-L42), that accepts anything that provides such map-like interface. Its counterpart, [unpack_map](https://github.com/codx-dev/msgpacker/blob/65a0ab07f835d765d9b07cfe91e4b9207ef339ff/msgpacker/src/unpack/collections.rs#L67-L74), will return anything that can be constructed `FromIterator<(K, V)>`. We can see this pattern in any reputable map implementation, such as [hashbrown](https://docs.rs/hashbrown/0.14.0/hashbrown/struct.HashMap.html#impl-FromIterator%3C(K,+V)%3E-for-HashMap%3CK,+V,+S,+A%3E).

However, it is very inconvenient to the user to have to learn that there is a function that will perform such conversion. Instead, a derive macro attribute will help to avoid such case. Consider the following struct:

```rust
#[derive(MsgPacker)]
pub struct City {
    name: String,
    #[msgpacker(map)]
    inhabitants_per_street: HashMap<String, u64>,
    #[msgpacker(array)]
    zones: Vec<String>,
}
```

The `#[msgpacker(map)]` will signal to `#[derive(MsgPacker)]` that this is a map; hence, it must be encoded via the appropriate function. The same applies to `array`.

We cannot treat `Vec<_>` as atomic and implement the traits automatically because the protocol has a special case for `Vec<u8>`: it can be a [binary](https://github.com/msgpack/msgpack/blob/8aa09e2a6a9180a49fc62ecfefe149f063cc5e4b/spec.md?plain=1#L56-L58) format. Since `u8` is `Packable`, there is an ambiguity on how the serialization strategy should be performed. We leave this decision to the user, and provide mechanisms for the user to easily pick one or the other.

The following will work automatically, as `Vec<u8>` is part of the canonical protocol:

```rust
#[derive(MsgPacker)]
pub struct City {
    name: String,
    #[msgpacker(map)]
    inhabitants_per_street: HashMap<String, u64>,
    #[msgpacker(array)]
    zones: Vec<String>,
    payload:Vec<u8>,
}
```

Or, if the user wants this payload to be a message pack array, he can:

```rust
#[derive(MsgPacker)]
pub struct City {
    name: String,
    #[msgpacker(map)]
    inhabitants_per_street: HashMap<String, u64>,
    #[msgpacker(array)]
    zones: Vec<String>,
    #[msgpacker(array)]
    payload:Vec<u8>,
}
```

## Motivation and Conclusion

The motivation behind [msgpacker](https://crates.io/crates/msgpacker)'s development stemmed from a desire to create a GPU-accelerated frontend for [NeoVim](https://neovim.io/). While existing MessagePack implementations were functional, they were perceived as unnecessarily complex. Msgpacker's initial version demonstrated satisfactory performance but had vulnerabilities such as [recursion attacks](https://github.com/codx-dev/msgpacker/issues/4) and [eager allocation](https://github.com/codx-dev/msgpacker/issues/5).

The refactor of [msgpacker](https://crates.io/crates/msgpacker) aimed to enhance performance and security while simplifying the implementation. By eliminating the intermediate enum and utilizing lazy functions, significant performance gains were achieved, surpassing other popular libraries. The resulting code provides a more efficient and secure solution for MessagePack serialization in Rust.
