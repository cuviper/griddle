[![Crates.io](https://img.shields.io/crates/v/griddle.svg)](https://crates.io/crates/griddle)
[![Documentation](https://docs.rs/griddle/badge.svg)](https://docs.rs/griddle/)
[![Build Status](https://dev.azure.com/jonhoo/jonhoo/_apis/build/status/griddle?branchName=master)](https://dev.azure.com/jonhoo/jonhoo/_build/latest?definitionId=26&branchName=master)
[![Coverage Status](https://codecov.io/gh/jonhoo/griddle/branch/master/graph/badge.svg)](https://codecov.io/gh/jonhoo/griddle)
![Maintenance](https://img.shields.io/badge/maintenance-experimental-blue.svg)

A `HashMap` variant that spreads resize load across inserts.

Most hash table implementations (including [`hashbrown`], the one in
Rust's standard library) must occasionally "resize" the backing memory
for the map as the number of elements grows. This means allocating a new
table (usually of twice the size), and moving all the elements from the
old table to the new one. As your table gets larger, this process takes
longer and longer.

For most applications, this behavior is fine — if some very small number
of inserts take longer than others, the application won't even notice.
And if the map is relatively small anyway, even those "slow" inserts are
quite fast. Similarly, if your map grow for a while, and then _stops_
growing, the "steady state" of your application won't see any resizing
pauses at all.

Where resizing becomes a problem is in applications that use maps to
keep ever-growing state where tail latency is important. At large scale,
it is simply not okay for one map insert to take 30 milliseconds when
most take single-digit **micro**seconds. Worse yet, these resize pauses
can compound to create [significant spikes] in tail latency.

This crate implements a technique referred to as "incremental resizing",
in contrast to the common "all-at-once" approached outlined above. At
its core, the idea is pretty simple: instead of moving all the elements
to the resized map immediately, move a couple each time an insert
happens. This spreads the cost of moving the elements so that _each_
insert becomes a little slower until the resize has finished, instead of
_one_ insert becoming a _lot_ slower.

This approach isn't free, however. While the resize is going on, the old
table must be kept around (so memory isn't reclaimed immediately), and
all reads must check both the old and new map, which makes them slower.
Only once the resize completes is the old table reclaimed and full read
performance restored.

To help you decide whether this implementation is right for you, here's
a handy reference for how this implementation compares to the standard
library map:

 - Inserts all take approximately the same time.
   After a resize, they will be slower for a while, but only by a
   relatively small factor.
 - Memory is not reclaimed immediately upon resize.
 - Reads and removals of **old** or **missing** keys are slower for a
   while after a resize.
 - The incremental map is slightly larger on the stack.

## Benchmarks

There is a silly, but illustrative benchmark in `benches/vroom.rs`. It
just runs lots of inserts back-to-back, and measures how long each one
takes. The problem quickly becomes apparent:

```console
$ cargo bench --bench vroom > data.txt
hashbrown::HashMap max: 25.481194ms, mean: 1.349µs
griddle::HashMap max: 1.700794ms, mean: 1.362µs
```

You can see that the standard library implementation (through
`hashbrown`) has some pretty severe latency spikes. This is more readily
visible through a timeline latency plot (`misc/vroom.plt`):

![latency spikes on resize](https://raw.githubusercontent.com/jonhoo/griddle/master/misc/vroom.png)

Resizes happen less frequently as the map grows, but they also take
longer _when_ they occur. With griddle, those spikes are mostly gone.
There is a small linear component left, which I believe comes from the
work required to find the buckets that hold elements that must be moved.

## Implementation

Griddle uses the
[`hashbrown::raw`](https://docs.rs/hashbrown/0.8/hashbrown/raw/index.html)
API, which allows it to take advantage of all the awesome work that has
gone into making `hashbrown` as fast as it is. The key different parts
of griddle live in `src/raw/mod.rs`.

Griddle aims to stick as closely to `hashbrown` as it can, both in terms
of code and API. `src/map.rs` and `src/set.rs` are virtually identical
to the equivalent files in `hashbrown` (I encourage you to diff them!),
without only some (currently;
[#4](https://github.com/jonhoo/griddle/issues/4)) unsupported API
removed. Like `hashbrown`, griddle exposes a `raw` module, which lets
you build your own map implementation on top of griddle's table variant.

## Why "griddle"?

You can amortize the cost of making hashbrowns by using a griddle..?

[`hashbrown`]: https://crates.io/crates/hashbrown
[significant spikes]: https://twitter.com/jonhoo/status/1277618908355313670

## License

Licensed under either of

 * Apache License, Version 2.0
   ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license
   ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.
