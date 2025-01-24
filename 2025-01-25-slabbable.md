# Slabbable for all your rotating stable slotmaps

![meme slip slop slap](./assets/slip-slop-slap.jpg)

*Note: It's summer down here so I could have just named this as slip-slop-slap instead. j/k*

## So what exactly is [Slabbable](https://docs.rs/slabbable) ?

a trait designed for a very specific use, mainly as the storage layer within my io_uring abstraction involving complicated 3-way ownership and unforgettable lifetimes.

## Why?

I simply wanted a harmonized but yet configurable type that acts a bit like [stable-vec](https://crates.io/crates/stable-vec), [slab](https://crates.io/crates/slab) or [slotmap](https://crates.io/crates/slotmap).

It helps me test my specific use-scenario and validate the exact guarantees I need (rather than want) as well as bench and profile them more easily when the trait and it's implementations are molded in the usage specific scenario -- as I do in the [slabbable-validation](https://github.com/yaws-rs/edifice/tree/main/slabbable-validation) crate within the monorepo holding these things together.

## How do I use it?

You can look at the [validation tests](https://github.com/yaws-rs/edifice/tree/main/slabbable-validation/src/lib.rs) or my [heavily-work-in-progress yaws io_uring abstractions](https://github.com/yaws-rs/io_uring-utils/tree/main/io-uring-epoll/src) how I use the Slabbable (or SelectedSlab) as the storage layer for the extended lifetime and complicated ownership items without simply std::mem::forgetting them.

Or basically after `cargo add slabbable-impl-selector`:

```rust
// You can also directly use impl crate without opting via the selector
// use slabbable_slab::SlabSlab as SelectedSlab;
use slabbable_impl_selector::SelectedSlab;

#[repr(C, packed)]
#[derive(Debug, Clone)]
struct MyCData {
  whatever: u16,
}

let slab = SelectedSlab::<MYCData>::with_fixed_capacity(1);
let first_key = slab.take_next_with(MyCData { whatever: 1 }).expect("We should be able to take one and the only one.");
// assert_eq!(first_key, 0 as usize); // we can't rely on this, each impl ideally specifies the issue order
// slab.take_next_with(MYCData { whatever: 2 }).expect("This would be correct, we were guarded from re-allocating over the bounded fixed capacity.");
let first_ref = slab.slot_get_ref(first_key); // &raw first_ref is now relatively stable(ish)
```

See [the trait docs](https://docs.rs/slabbable/latest/slabbable/trait.Slabbable.html#required-methods) for full intended API spec.

## Guardrails

It's quite easy to leave something undocumented and then realize that it's not acting as one would assume.

My validation consists on ensuring "the commandments" I've [outlined in the trait](https://github.com/yaws-rs/edifice/blob/main/slabbable/src/lib.rs) hold across all the implementations namedly hold stable memory addresses across the fixed capacity and disallow state where the underlying addresses become invalidated.

It's also essential to not only validate but benchmark both timings and memory usage in various usage-specific ways where the trait helps to harmonize this goal.

Currently you can see how I can [validate](https://github.com/yaws-rs/edifice/blob/main/slabbable-validation/src/lib.rs#L24) multiple implementations based on harmonized trait.

And how I can measure virt and phys memory usage under various scenarios in [mem.rs](https://github.com/yaws-rs/edifice/blob/main/slabbable-validation/src/bin/mem.rs) within the validation crate - which I'm planning to move under criterion as measurable -- afaik I don't know anyone who has added trait impl for making convenient memory benchmarks.

## Slabbable Implementations

So far I've finished [hashbrown](https://docs.rs/hashbrown/latest/hashbrown/) - [nohash-hasher](https://crates.io/crates/nohash_hasher) (given I dont't need HashDoS resistance and my key is just usize) abstracted under [slabbable-hash](https://github.com/yaws-rs/edifice/blob/main/slabbable-impls/hash/src/lib.rs#L62).

I've also finished slab via [slabbable-slab](https://github.com/yaws-rs/edifice/blob/main/slabbable-impls/slab/src/lib.rs#L27) and stable-vec via [slabbable-stablevec](https://github.com/yaws-rs/edifice/blob/main/slabbable-impls/stable-vec/src/lib.rs#L37).

## Preferred Slabbable Implementation

I tend to prefer the hashing one as I can easily implement rotating-revolving usize key at usize::MAX instead of re-using previously used keys through slab / stable-vec that typically for example re-use immediately key zero when it has been made vacant instead of incrementing serial counter.

Reason I prefer the serial rotating-revolving counter over re-using index keys revolves mitigating the [ABA problem](https://en.wikipedia.org/wiki/ABA_problem).

Ofcourse performance is also important so being able to more conveniently bench and validate the implementation comes to play e.g. default SelectedSlab (hash) impl:

~/yaws/edifice/slabbable-validation$ cargo bench

Or slab non-default SelectedSlab impl

~/yaws/edifice/slabbable-validation$ env RUSTFLAGS='--cfg slabbable_impl="slab"' cargo bench

1,024,000 linear insert

| impl | timing |
| :--- | :---   |
| hash | [9.5091 ms 9.5891 ms 9.6844 ms] |
| slab | [8.3422 ms 8.3555 ms 8.3685 ms] |
| stable-vec | [9.4680 ms 9.5011 ms 9.5562 ms] |

It could totally be that my benchmark is totally wrong but I'm not seeing much difference for insert.

Real conclusive differences come from chaos & load testing which I will do later when I integrate it into my [yaws](https://github.com/yaws-rs) runtime through my [io-uring abstractions](https://github.com/yaws-rs/io_uring-utils).

However memory usage story is quite different between hash (default), slab and stablevecs.

* ~/yaws/edifice/slabbable-validation$ `cargo run`
* ~/yaws/edifice/slabbable-validation$ `env RUSTFLAGS='--cfg slabbable_impl="slab"' cargo run`
* ~/yaws/edifice/slabbable-validation$ `env RUSTFLAGS='--cfg slabbable_impl="stablevec"' cargo run`

| impl      | 10M initialized baseline        | 10M filled increase           |
| :---      | :---                            | :---                          |
| hash      | phys  65.54 kB virt 285.93 MB   | phys +159.46 MB virt +4.10 kB |
| slab      | phys 516.10 kB virt 161.06 MB   | phys +160.39 MB virt +4.10 kB |
| stablevec | phys 131.07 kB virt  82.13 MB   | phys  +81.57 MB virt +4.10 kB |

Hash implementation at 10M initialized and filled takes more memory - an unsurprising tradeoff.

I need to do more testing but this shows the importance of capacity and bounds planning given environment and use-scenarios that should form part of validation within the top-level binary.

## Configurable Abstraction - What cfg()

As you see above, we're using overrideable custom cfg() options to provide global configuration options.

The idea is to give the top-level binary the power to choose the appropriate abstraction over well known good default e.g. as an override without transient dependencies either muddying or breaking the compilation - or worst miscompilation leading to unspecified behaviour.

Often I see crates using feature flags non-optimally to create mutually exclusive configuration knobs when it would make more sense to use configuration or 'cfg' predicates for mutual exclusivity.

## Configurable Abstration - Why cfg()

If you would like to see more discussion around why using cfg makes more sense, have a look into the issue I keep referring people to: [dalek-cryptography/curve25519-dalek#414](https://github.com/dalek-cryptography/curve25519-dalek/issues/414) where we ended up abstracting backend override with cfg(curve25519_dalek_backend) for the v4 major release as well as cfg(curve25519_dalek_bits) [described](https://github.com/dalek-cryptography/curve25519-dalek/tree/main/curve25519-dalek#bits--word-size) in it's readme.

## Cofiguration Abstraction - Real World cfg()

I also found out nobody used non-default backend in curve25519-dalek because pretty much none of the dependant crates "relayed" the configuration options through features -- which may end up later being helped by the rfc and compiler implementation regarding globally mutually exclusive features.

## Configuration Abstraction - Compiler Mandate

Compiler recently formalized linting the expected configuration predicates vs occured ones so it's much less easy to make errors gating through typos - e.g. [cfg lint in curve25519-dalek](https://github.com/dalek-cryptography/curve25519-dalek/blob/main/curve25519-dalek/Cargo.toml#L75).

## Reduce the cfg() Gates

Gating can be very involved process when one has to change it in several places leading to hideous errors that may be hard to find especially when not properly continuously testing all the possible cfg combinations.

One can also help to reduce "the gating" around as seen in curve25519_dalek by abstracting "selected implementation" through proxy type either as:

```rust
pub type SelectedSlab<Item> = slabbable_hash::HashSlab<Item>
```

Or re-exporting:
```rust
pub use slabbable_hash::HashSlab as SelectedSlab;
```

The minor difference is the compiler will scream any errors referring to the concrete impl instead of proxy type when using pub type whcih has the trade-off having to line-up the containerized generic in the pub type alias.

You can see how I reduce the gating through [slabbable-impl-selector](https://github.com/yaws-rs/edifice/blob/main/slabbable-impl-selector/src/lib.rs) alias-proxying SelectedSlab conveniently for everything that uses it.

As an example how much gating typically needs to change, you can [look at another PR](https://github.com/dalek-cryptography/curve25519-dalek/pull/695) where I changed AVX512-IFMA into non-default when nightly is used.

Having proxy aliased-types can help reducing the amount of gating required in multiple places.

You can also cfg-validation as I do in the slabbable-impl-selector in build.rs like in [curve25519-dalek/build.rs](https://github.com/dalek-cryptography/curve25519-dalek/blob/main/curve25519-dalek/build.rs).

## Tradeoffs

I don't care too much about memory usage at the moment but rather the "worst case" e.g. by creating bounds at the very the maximum capacity where it may be more difficult to free memory in between.

I could probably improve ramp-up / down memory usage but I rather just bound the maximum to optimize my use-scenarios.

In server programing I typically like to make it easily able to calculate total resource usage.

## Obligatory Support my Work Msg

If you would like to help, please feel free to send and issue or a PR.

If you need help relating to my work, please let me know if I can help in any way or if you feel that my work helps you in any way - whether it's capitalism or open source :)

You can find me at [My personal Discord](https://discord.gg/rXVsmzhaZa) or [Another smol Discord](https://discord.gg/pW35BNSBeV) I tend to hang at.
