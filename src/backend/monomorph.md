# Monomorphization

<!-- toc -->

As you probably know, Rust has a very expressive type system that has extensive
support for generic types. But of course, assembly is not generic, so we need
to figure out the concrete types of all the generics before the code can
execute.

Different languages handle this problem differently. For example, in some
languages, such as Java, we may not know the most precise type of value until
runtime. In the case of Java, this is ok because (almost) all variables are
reference values anyway (i.e. pointers to a heap allocated object). This
flexibility comes at the cost of performance, since all accesses to an object
must dereference a pointer.

Rust takes a different approach: it _monomorphizes_ all generic types. This
means that compiler stamps out a different copy of the code of a generic
function for each concrete type needed. For example, if I use a `Vec<u64>` and
a `Vec<String>` in my code, then the generated binary will have two copies of
the generated code for `Vec`: one for `Vec<u64>` and another for `Vec<String>`.
The result is fast programs, but it comes at the cost of compile time (creating
all those copies can take a while) and binary size (all those copies might take
a lot of space).

Monomorphization is the first step in the backend of the Rust compiler.

## Collection

First, we need to figure out what concrete types we need for all the generic
things in our program. This is called _collection_, and the code that does this
is called the _monomorphization collector_.

Take this example:

```rust
fn banana() {
   peach::<u64>();
}

fn main() {
    banana();
}
```

The monomorphization collector will give you a list of `[main, banana,
peach::<u64>]`. These are the functions that will have machine code generated
for them. Collector will also add things like statics to that list.

See [the collector rustdocs][collect] for more info.

[collect]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_monomorphize/collector/index.html

The monomorphization collector is run just before MIR lowering and codegen.
[`rustc_codegen_ssa::base::codegen_crate`][codegen1] calls the
[`collect_and_partition_mono_items`][mono] query, which does monomorphization
collection and then partitions them into [codegen
units](../appendix/glossary.md#codegen-unit).

## Codegen Unit (CGU) partitioning

For better incremental build times, the CGU partitioner creates two CGU for each source level
modules. One is for "stable" i.e. non-generic code and the other is more volatile code i.e.
monomorphized/specialized instances.

For dependencies, consider Crate A and Crate B, such that Crate B depends on Crate A.
The following table lists different scenarios for a function in Crate A that might be used by one
or more modules in Crate B.

| Crate A function | Behavior |
| - | - |
| Non-generic function | Crate A function doesn't appear in any codegen units of Crate B |
| Non-generic `#[inline]` function |  Crate A function appears within a single CGU  of Crate B, and exists even after post-inlining stage|
| Generic function |  Regardless of inlining, all monomorphized (specialized) functions <br> from Crate A appear within a single codegen unit for Crate B. <br> The codegen unit exists even after the post inlining stage.|
| Generic `#[inline]` function |   - same - |

For more details about the partitioner read the module level [documentation].

[mono]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_monomorphize/partitioning/fn.collect_and_partition_mono_items.html
[codegen1]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_codegen_ssa/base/fn.codegen_crate.html
[documentation]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_monomorphize/partitioning/index.html

## Polymorphization

As mentioned above, monomorphization produces fast code, but it comes at the
cost of compile time and binary size. [MIR optimizations][miropt] can help a
bit with this.

In addition to MIR optimizations, rustc attempts to determine when fewer
copies of functions are necessary and avoid making those copies - known
as "polymorphization". When a function-like item is found during
monomorphization collection, the
[`rustc_mir_monomorphize::polymorphize::unused_generic_params`][polymorph]
query is invoked, which traverses the MIR of the item to determine on which
generic parameters the item might not need duplicated.

Currently, polymorphization only looks for unused generic parameters. These
are relatively rare in functions, but closures inherit the generic
parameters of their parent function and it is common for closures to not
use those inherited parameters. Without polymorphization, a copy of these
closures would be created for each copy of the parent function. By
creating fewer copies, less LLVM IR is generated and needs processed.

`unused_generic_params` returns a `FiniteBitSet<u64>` where a bit is set if
the generic parameter of the corresponding index is unused. Any parameters
after the first sixty-four are considered used.

The results of polymorphization analysis are used in the
[`Instance::polymorphize`][inst_polymorph] function to replace the
[`Instance`][inst]'s substitutions for the unused generic parameters with their
identity substitutions.

Consider the example below:

```rust
fn foo<A, B>() {
    let x: Option<B> = None;
}

fn main() {
    foo::<u16, u32>();
    foo::<u64, u32>();
}
```

During monomorphization collection, `foo` will be collected with the
substitutions `[u16, u32]` and `[u64, u32]` (from its invocations in `main`).
`foo` has the identity substitutions `[A, B]` (or
`[ty::Param(0), ty::Param(1)]`).

Polymorphization will identify `A` as being unused and it will be replaced in
the substitutions with the identity parameter before being added to the set
of collected items - thereby reducing the copies from two (`[u16, u32]` and
`[u64, u32]`) to one (`[A, u32]`).

`unused_generic_params` will also invoked during code generation when the
symbol name for `foo` is being computed for use in the callsites of `foo`
(which have the regular substitutions present, otherwise there would be a
symbol mismatch between the caller and the function).

As a result of polymorphization, items collected during monomorphization
cannot be assumed to be monomorphic.

It is intended that polymorphization be extended to more advanced cases,
such as where only the size/alignment of a generic parameter are required.

More details on polymorphization are available in the
[master's thesis][thesis] associated with polymorphization's initial
implementation.

[miropt]: ../mir/optimizations.md
[polymorph]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_monomorphize/polymorphize/fn.unused_generic_params.html
[inst]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/instance/struct.Instance.html
[inst_polymorph]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/instance/struct.Instance.html#method.polymorphize
[thesis]: https://davidtw.co/media/masters_dissertation.pdf
