- Feature Name: generators
- Start Date: 2016-12-15
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This adds generators to Rust, which have the ability to yield and return back to the caller. Yielding will suspend the generator and the owner can resume it later. Generator are also known as stackless coroutines or semicoroutines.

# Motivation
[motivation]: #motivation

The primary motivation for this is that it allows writing code that looks like code with blocking operations, but which actually suspends and does something else while that operation is happening. In order to do this in Rust currently, we need to either create another thread, which comes at a huge performance cost, create a manual state machine which is hard to reason about, or use future combinators to create the state machine, which is a lot better, but still hard and unfamiliar.

It's useful for writing iterators and high-scale servers where you wouldn't want an expensive thread per concurrent request. So this feature aligns well with the [proposed 2017 roadmap](https://github.com/rust-lang/rfcs/pull/1774).

# Detailed design
[design]: #detailed-design

Generators are functions which can suspend its execution and resume at a later point. Initially they start out suspended and return a value of an anonymous type which implement the `Generator` trait. This return value is wrapped in a `Wrapper` struct. This avoids running into problems with trait coherence. Let's call the anonymous type the frame type for the specific generator. Instead of allocating arguments, local and temporary variables on the stack, we'll allocate storage in the frame type. The compiler can avoid storing the values in the frame type, if possible. We also need to keep track of where in the function we are, so we add an instruction pointer which initally points to the entry point of the function. The Rust compiler is free to optimize this representation as long as it preserves the semantics.

Let's look at the Generator trait and relevant code:
```rust
pub struct Wrapper<T>(T);

pub enum State<Y, R> {
    Yielded(Y),
    Complete(R),
}

pub trait Generator<Args>: ?Move {
    type Yield;
    type Return;
    fn resume(&mut self, args: Args) -> State<Self::Yield, Self::Return>;
}
```

When `resume` is called on a generator we resume the function in the position the instruction pointer indicates and it can return any of the `State` enum variants, giving the reason why the generator suspended. `Yielded` is returned with the yielded value when generators yield. `Complete` is given with the return value when the generator finally finishes, and further resumptions will result in panics.

The associated `Yield` types gives the type of values that can be yielded and `Return` gives the type of return values.

Functions and closures will return a generator if they use the `=>` syntax instead of `->` to give the return type. `yield` statements are only allowed if the body specifies an generator to return. Closures that return a generator will still be able to capture variables.

We have the following implementation which just ensures that our wrapper type implements `Generator`:
```rust
impl<Args, T: Generator<Args>> Generator<Args> for Wrapper<T> {
    type Yield = T::Yield;
    type Return = T::Return;

    fn resume(&mut self, args: Args) -> State<Self::Yield, Self::Return> {
        self.0.resume(args)
    }
}
```

Here is an example of a generator:
```rust
fn count_to_ten() => impl Generator<(), Yield=usize, Return=()> {
    for i in 1...10 {
        yield i;
    }
}
```

## References to local variables

## Accessing the implicit argument

## Suspend points

Let's look at suspend points in detail. `yield` and `return` are all sugar built on top of a suspend operation, which suspends the generator and specifies the return value of the `resume` function. The suspend operation is not accessible for end users and is only an implementation detail.

`return <v>` expands to `suspend State::Complete(<v>); panic!("the generator had already completed")`

`yield <v>` expands to `suspend State::Yielded(<v>)`

## Immovable generators

Support for immovable generator requires immovable types, which is a separate RFC.

We indicate that a generator is immovable by the addition of the `static` keyword in front of the return type.

Immovable generators return an anonymous instance of the `PinGenerator` trait wrapped in the `ImmovableWrapper` struct. It allows you to construct an immovable generator by calling the `pin` method. Since the only state that's initally captured by a generator is the arguments to the function creating it, and the arguments must implement `Move` this constructor type can implement `Move`, which means you can freely move it until you actually start the generator.

```rust
pub struct ImmovableWrapper<T: ?Move>(T);
pub struct ImmovablePinnedWrapper<T: ?Move>(T);

pub trait PinGenerator<Args> {
    type Yield;
    type Return;
    type Pinned: Generator<Args, Yield=Self::Yield, Return=Self::Return> + ?Move;
    fn pin(self) -> ImmovablePinnedWrapper<Pinned>;
}
```

`ImmovableWrapper` implements `PinGenerator`, and `ImmovablePinnedWrapper` implements `Generator` in a similar manner as the above implementation of `Generator` for `Wrapper`.

Here is the above example, this time as an immovable generator:
```rust
fn count_to_ten() => static impl PinGenerator<(), Yield=usize, Return=()> {
    for i in 1...10 {
        yield i;
    }
}
```

## Implementation

### Desugaring to HIR

The suspend point sections gave some expansions for the relevant expressions. These expansions should be applied when generating HIR, with the exception of the `return` expansion, which should stay intact in order to avoid duplicating the panic code.

### Implications for type inference

Type inference for generators would work similarly to type inference for closures. We have some expression which has an anonymous type which implements a trait.

When we do type inference for functions or closures which are generators, we create 3 type variables, one for the type of the executor (which does not appear in the signature), one for the return type (since the type in the signature is the generator to return), and one for the yield type, which defaults to `!`. The anonymous type implements `Generator<E, Return=R, Yield=Y>` where `E` is the type variables created for the executor, `R` is the return type variable and `Y` is the yield type variable. The type of values in the yield expression unifies with the yield type variable.

### Changes to the borrow checker

Immovable generators require no changes to the borrow checker. Movable generator do however, since the stack frame of a generator can move while it is suspended. That could result in all references to data inside being invalidated. Because of this, we require the borrow checker to reject references to local variables, temporary variables, and arguments which lifetimes cross suspension points.

The minimal change would be to reject all references with lifetimes crossing suspension points, but this is quite restrictive, so we would like a more precise solution.

One such a solution would be to use dataflow analysis to detect when variables contains loans to local values and use the result of that to calculate if loans are to local values. Finally we reject only loans which are to local values and cross a suspension point.

### State machine transformation

We do the state machine transformation after borrow checking so that we avoid having to translate errors from the post-transformation program to the pre-transformation program.

After MIR generation of generators, we can run MIR optimizations which are aware of suspend statements. Then we split the function into 2 parts. One which contains the body and is the `resume` function of the implemenatation of the `Generator` trait. The other function corresponds to the declared function and will just constructs an instance of the anonymous generator type, passing in the arguments which are needed after the first suspend operation.

We will move the storage of variables which are live across suspend points into the anonymous type and generated loads and stores as needed.

The anonymous type will also contain a state variable which is initialized as `EntryPoint`. There will be a switch on the top of the function which selects the basic block to go to based on this state variaible. `EntryPoint` goes to what was the previous entry point of the function. Suspend statements will expand to a statement setting the state variable to a new state `S` and a regular return statement returning the value passed to the suspend statement. The point after the suspend statement is turned into a new basic block and `S` will map to it.

Return statements are expanded to a statement setting the state variable to `Complete` and a return statement returning `State::Complete(r)` where `r` is the value passed to the return statement. A single basic block is added per function which `Complete` maps to. It contains the code which panics. This way, there isn't any duplication of panic code.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

The name generator is chosen over stackless coroutines since these generators actually live on the stack, as opposed to C++'s stackless coroutine. Also that is done to avoid confusion with stackful coroutines, for which there exists libraries already.

The concepts of generators is likely to be known to programmers already since a number of other languages have this feature. It should still be introduced as a concept in the book, possibly with the viewpoints of functions being suspended and as a transformation into a state machine.
Some examples for iterators, futures and streams should be added to the book and _Rust by Example_.

# Drawbacks
[drawbacks]: #drawbacks

---No callback support

This adds lot of complexity to the language.

# Alternatives
[alternatives]: #alternatives

An alternative is [this RFC](https://github.com/rust-lang/rfcs/pull/1823). It focuses on minimal language changes at the cost of ergonomics. It lacks the `Blocked` variant from the generator result, which means users dealing with asynchronous operations will have to use an enum on top of that. It also doesn't allow functions to be generators, but require a nested closures.

# Unresolved questions
[unresolved]: #unresolved-questions

Should we let the generator return a new `State::Completed` variant instead of panicking?