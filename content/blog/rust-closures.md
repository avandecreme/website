+++
title = "Understanding rust closures"
date = "2026-01-24"
description = "Explorations on what are rust closures and how they work under the hood."
# taxonomies.tags = [
#     "rust",
# ]
+++

While reading the [Explicit capture clauses](https://smallcultfollowing.com/babysteps/blog/2025/10/22/explicit-capture-clauses/)
blog post, I realized that my understanding of rust closures was very superficial.
This article is an attempt at explaining what I learned while reading and experimenting on the subject.
It starts from the very basics and then explore more complex topics.
Note that each title is a link to a rust playground where you can experiment
with the code in the section.

# [Closures basics](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=b0ad9054f991999ae106c4941113a1b5)

You probably already know that a closure in rust is a function written with the following syntax:

```rust
let double_closure = |x| x * 2;
assert_eq!(4, double_closure(2));
```

Written as a regular function it looks like:

```rust
fn double_function(x: u32) -> u32 {
    x * 2
}
assert_eq!(4, double_function(2));
```

Very similar. There is actually a small difference between the two, the `double_function` parameter and return type are `u32`.
On the other hand, because we did not specify any type in `double_closure`, the
[default integer type](https://github.com/rust-lang/rfcs/blob/master/text/0212-restore-int-fallback.md) has been picked, namely `i32`.

We can fix that like this:

```rust
let double_typed_closure = |x: u32| -> u32 { x * 2 };
assert_eq!(4, double_typed_closure(2));
assert_eq!(4, double_typed_closure(2u32));
// assert_eq!(4, double_typed_closure(2u16)); // This would be an error.
```

And for a classic example usage of closures, we can use the `Option::map` method:

```rust
assert_eq!(Some(4), Some(2).map(|x| x * 2));
assert_eq!(Some(4), Some(2).map(double_closure)); // double_closure from above
assert_eq!(Some(4), Some(2).map(double_function)); // Passing double_function works too!
```

So, it seems closures are just a shorter syntax for functions with type inference.

# [Capture](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=5545b031dafe8cdb6d7b8c192848bc8a)

The main difference between closures and functions is that closures can capture
variables from their environment while functions can't:

```rust
let hello = "Hello ";
let greeter_closure = |x| String::new() + hello + x;

assert_eq!("Hello world", greeter_closure("world"));
assert_eq!(
    Some("Hello world".to_owned()),
    Some("world").map(greeter_closure)
);
```

Notice how the `hello` variable is used within the body of the `greeter_closure`.
Let's try that with a function:

```rust
let hello = "Hello ";

fn greeter_function(x: &str) -> String {
    String::new() + hello + x
}
```

```
error[E0434]: can't capture dynamic environment in a fn item
 --> src/main.rs:7:25
  |
7 |         String::new() + hello + x
  |                         ^^^^^
  |
  = help: use the `|| { ... }` closure form instead
```

This does not work and the compiler helpfully suggest to use a closure instead.

## [Capture by shared reference](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=0276070dfd19827a12a77cf31beae73e)

In the `greeter_closure` example above, the `hello` variable was captured by
shared reference because the variable is only read.
As shown below, we can still use that variable after the closure declaration and usage:

```rust
let hello = "Hello ";
let greeter_closure = |x| String::new() + hello + x;

// We can still use the `hello` variable here
assert_eq!("Hello ", hello);

assert_eq!("Hello world", greeter_closure("world"));

// And here
assert_eq!("Hello ", hello);
```

## [Capture by mutable reference](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=76889d80d2162747eb4a341123e4ba86)

It is also possible to capture by mutable reference so that the closure can alter
the value of the captured variable. See this naive way to compute the sum of
integers from 1 to 10:

```rust
let mut total = 0;
let add_mut_closure = |x| total += x;

// We can't access total here:
// assert_eq!(0, total);
// error[E0502]: cannot borrow `total` as immutable because it is also borrowed as mutable

(1..=10).for_each(add_mut_closure);

// But we can access total here, now that `add_mut_closure` is out of scope.
assert_eq!(55, total);
```

## [Capture by value](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=b3237a395181972495dc9e4c68aa12cb)

Finally, one can capture by value:

```rust
let last_word = "last word: ".to_owned();
let drop_closure = |sigh| {
    let res = String::new() + &last_word + sigh;
    drop(last_word); // Forcing the capture by value
    res
};

// We can't access `last_word` here:
// assert_eq!("last word: ".to_owned(), last_word);
// error[E0382]: borrow of moved value: `last_word`

assert_eq!("last word: sigh!", drop_closure("sigh!"));

// We can't access `last_word` here either
// assert_eq!("last word: ".to_owned(), last_word);
// error[E0382]: borrow of moved value: `last_word`

// And we can't call drop_closure again
// assert_eq!("last word: sigh!", drop_closure("sigh!"));
// error[E0382]: use of moved value: `drop_closure`
```

# [FnOnce trait](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2024&gist=10ea8cfe65c1d2f4e1bd89a6d0674bd0)

In the previous example, notice the last error when trying to call `drop_closure` twice.
Here is the full error:

```
error[E0382]: use of moved value: `drop_closure`
  --> src/main.rs:18:32
   |
12 | assert_eq!("last word: sigh!", drop_closure("sigh!"));
   |                                --------------------- `drop_closure` moved due to this call
...
18 | assert_eq!("last word: sigh!", drop_closure("sigh!"));
   |                                ^^^^^^^^^^^^ value used here after move
   |
note: closure cannot be invoked more than once because it moves the variable `last_word` out of its environment
  --> src/main.rs:5:10
   |
 5 |     drop(last_word);
   |          ^^^^^^^^^
note: this value implements `FnOnce`, which causes it to be moved when called
  --> src/main.rs:12:32
   |
12 | assert_eq!("last word: sigh!", drop_closure("sigh!"));
   |                                ^^^^^^^^^^^^
```

The interesting note is:
```
note: this value implements `FnOnce`, which causes it to be moved when called
```

What is that `FnOnce` implementation the compiler is talking about?

It is a trait automatically implemented by the compiler which state that the closure can be called at least once. 

That trait is a bit special because it cannot be implemented manually in stable rust.  
However, if we switch to unstable and enable some features, we can play with it and try to desugar how closures are actually implemented by the compiler.

Let's try to desugar the `drop_closure` above.

First, make sure to [switch to the nightly channel](https://doc.rust-lang.org/book/appendix-07-nightly-rust.html#rustup-and-the-role-of-rust-nightly) and to enable
the following features (for example by putting them at the top of your `main.rs`):

```rust
#![feature(fn_traits)]
#![feature(unboxed_closures)]
```

Next, we need to define a struct having the captured variables as fields:

```rust
struct DropStruct {
    last_word: String,
}
```

Simple enough, we are capturing only one variable so our struct has one field.

Now the `FnOnce` implementation:

```rust
impl FnOnce<(&str,)> for DropStruct {
    type Output = String;
    extern "rust-call" fn call_once(self, (sigh,): (&str,)) -> Self::Output {
        let res = String::new() + &self.last_word + sigh;
        drop(self.last_word);
        res
    }
}
```

That is some weird trait!

Let's go step by step.  
`impl FnOnce<(&str,)>` means that we are implementing a closure which takes one parameter which is a `&str`.  
If the closure took two arguments of type `i32` and `i64` we would have `impl FnOnce<(i32, i64)>`. `(&str,)` is the definition of a tuple of one element.
See the reference on [tuple types](https://doc.rust-lang.org/reference/types/tuple.html) for details. 

`for DropStruct` should not be too surprising.

`type Output = String` specifies that our closure returns a `String`.

`extern "rust-call"` is some magic which I won't explain mostly because I don't know exactly why it is required.

The rest of the implementation should be self explanatory. We just took the content
of the closure and replaced `last_word` by `self.last_word`.

Let's try it:

```rust
let last_word = "last word: ".to_owned();
let drop_struct = DropStruct { last_word };

// We could call `call_once`:
// assert_eq!("last word: sigh!", drop_struct.call_once(("sigh!",)));

// But more simply, we can use the function call syntax:
assert_eq!("last word: sigh!", drop_struct("sigh!"));

// And we still can't call it twice
// assert_eq!("last word: sigh!", drop_struct("sigh!"));
// error[E0382]: use of moved value: `drop_struct`
```

# [FnMut trait](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2024&gist=fff4a9ff6d11986ea977ac7cd1c80806)

What about our `add_mut_closure` from before? We were able to call it
multiple times and even mutate the capture variables.

That kind of closure implements the `FnMut` trait.

Let's try to desugar the following closure which push elements in a vector:

```rust
let mut v = vec![];
let push_closure = |x| v.push(x);

(1..=5).for_each(push_closure);
assert_eq!(vec![1, 2, 3, 4, 5], v);
```

First we need to define a struct:

```rust
struct PusherStruct<'a> {
    v: &'a mut Vec<i32>,
}
```

Because we are capturing by reference, we need to introduce a lifetime.

Now the `FnMut` implementation:

```rust
impl<'a> FnMut<(i32,)> for PusherStruct<'a> {
    extern "rust-call" fn call_mut(&mut self, (x,): (i32,)) -> Self::Output {
        self.v.push(x)
    }
}
```

It is very similar to the `FnOnce` trait except that the function is called `call_mut` instead of `call_once` and that it takes `&mut self` instead of `self`.

Let's try to compile that:

```
error[E0277]: expected a `FnOnce(i32)` closure, found `PusherStruct<'a>`
 --> src/main.rs:8:5
  |
8 |     extern "rust-call" fn call_mut(&mut self, args: (i32,)) -> Self::Output {
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected an `FnOnce(i32)` closure, found `PusherStruct<'a>`
  |
help: the trait `FnOnce(i32)` is not implemented for `PusherStruct<'a>`
```

Turns out we need to implement `FnOnce` too. Remember that `FnOnce` defines functions which can be called **at least** once.
In the example above, we called our closure 5 times, so it can definitely be called at least once.

Let's implement it:

```rust
impl<'a> FnOnce<(i32,)> for PusherStruct<'a> {
    type Output = ();
    extern "rust-call" fn call_once(mut self, args: (i32,)) -> Self::Output {
        self.call_mut(args)
    }
}
```

Our closure does not return anything so the `Output` is the unit.  
As for the `call_once` implementation, we can just call `call_mut` to avoid repetition.

This should compile and we can now use it like so:
```rust
let mut v = vec![];
let pusher_struct = PusherStruct { v: &mut v };

(1..=5).for_each(pusher_struct);
assert_eq!(vec![1, 2, 3, 4, 5], v);
```

# [Fn trait](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2024&gist=d62def8acd356148ce5a4aa4ea4b8b09)

Finally, there is a third trait implemented by closures which can be called multiple times and don't need a mutable reference; the [Fn trait](https://doc.rust-lang.org/std/ops/trait.Fn.html).

To see that let's try to desugar the `greeter_closure` from before:

```rust
let hello = "Hello ";
let greeter_closure = |x| String::new() + hello + x;

assert_eq!("Hello world", greeter_closure("world"));
assert_eq!("Hello rust", greeter_closure("rust")); // Can be called multiple times
```

As usual, we need to define our struct:

```rust
struct GreeterStruct<'a> {
    hello: &'a str,
}
```

Let's not make the same mistake as before, and remember to implement `FnOnce` and `FnMut` first. The same way an `FnMut` closures are also `FnOnce` because they can be called **at least** once. `Fn` closures are also `FnMut` because if given a mutable reference, they can still perform their work which does not mutate the reference.

```rust
impl<'a, 'b> FnOnce<(&'b str,)> for GreeterStruct<'a> {
    type Output = String;
    extern "rust-call" fn call_once(self, args: (&'b str,)) -> Self::Output {
        self.call(args)
    }
}

impl<'a, 'b> FnMut<(&'b str,)> for GreeterStruct<'a> {
    extern "rust-call" fn call_mut(&mut self, args: (&'b str,)) -> Self::Output {
        self.call(args)
    }
}
```

This should be pretty straightforward. `call_once` and `call_mut` are just calling `call` which is defined in `Fn`:

```rust
impl<'a, 'b> Fn<(&'b str,)> for GreeterStruct<'b> {
    extern "rust-call" fn call(&self, (x,): (&'b str,)) -> Self::Output {
        String::new() + &self.hello + &x
    }
}
```

And we can use it like this:

```rust
let hello = "Hello";
let greeter_struct = GreeterStruct {
    hello,
};

assert_eq!("Hello world", greeter_struct("world"));
assert_eq!("Hello rust", greeter_struct("rust")); // Can be called multiple times
```

# [The move keyword](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2024&gist=86050d3163ea1ea4477e9626488a79e8)

You may already know that one can add the `move` keyword in front of a closure to force the closure to take ownership of the capture variables even if the closure only need a reference to it.  
For example:

```rust
let hello = "Hello ".to_owned();
let greeter_closure = move |x| String::new() + &hello + x;

// We can't access `hello` here
// assert_eq!("Hello ", hello);
// error[E0382]: borrow of moved value: `hello`

assert_eq!("Hello world", greeter_closure("world"));

// Nor here
// assert_eq!("Hello ", hello);
// error[E0382]: borrow of moved value: `hello`
```

In order to clearly understand what we can do depending on whether the closure needs a shared reference, a mutable reference or a value and if there is `move` keyword or not, let's introduce those small dummy functions:

```rust
fn by_ref(_data: &String) {}

fn by_mut(_data: &mut String) {}

fn by_value(_data: String) {}
```

Now, let's see what we can do with different combination of move / not move and by_ref / by_mut / by_value:

```rust
let data = "by_ref".to_owned();
let by_ref_closure = || by_ref(&data);

// Access data while the closure is still in scope
assert_eq!("by_ref", data);

// Call the closure once
by_ref_closure();

// Call the closure multiple times
by_ref_closure();

// Access data once the closure is out of scope
assert_eq!("by_ref", data);
```

Now with move:

```rust
let data = "move_by_ref".to_owned();
let move_by_ref_closure = move || by_ref(&data);

// Access data while the closure is still in scope
// assert_eq!("move_by_ref", data);
// error[E0382]: borrow of moved value: `data`

// Call the closure once
move_by_ref_closure();

// Call the closure multiple times
move_by_ref_closure();

// Access data once the closure is out of scope
// assert_eq!("move_by_ref", data);
// error[E0382]: borrow of moved value: `data`
```

This makes sense, since the closure took ownership of `data` we can't access it anymore from outside.

Similarly we can define the following closures:

```rust
let mut data = "by_mut".to_owned();
let by_mut_closure = || by_mut(&mut data);

let mut data = "move_by_mut".to_owned();
let move_by_mut_closure = move || by_mut(&mut data);

let data = "by_value".to_owned();
let by_value_closure = || by_value(data);

let data = "move_by_value".to_owned();
let move_by_value_closure = move || by_value(data);
```

I will let you play with them, here what you should see:


<table class="closures-table">
    <thead>
        <tr>
            <th></th>
            <th>by_ref</th>
            <th>by_mut</th>
            <th>by_value</th>
            <th>move by_ref</th>
            <th>move by_mut</th>
            <th>move by_value</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <th scope="row">Access when in scope</th>
            <td>✅</td>
            <td>❌</td>
            <td>❌</td>
            <td>❌</td>
            <td>❌</td>
            <td>❌</td>
        </tr>
        <tr>
            <th scope="row">Call once</th>
            <td>✅</td>
            <td>✅</td>
            <td>✅</td>
            <td>✅</td>
            <td>✅</td>
            <td>✅</td>
        </tr>
        <tr>
            <th scope="row">Call multiple times</th>
            <td>✅</td>
            <td>✅</td>
            <td>❌</td>
            <td>✅</td>
            <td>✅</td>
            <td>❌</td>
        </tr>
        <tr>
            <th scope="row">Access when out of scope</th>
            <td>✅</td>
            <td>✅</td>
            <td>❌</td>
            <td>❌</td>
            <td>❌</td>
            <td>❌</td>
        </tr>
    </tbody>
</table>

And the trait implemented by each closures:

<table class="closures-table">
    <thead>
        <tr>
            <th></th>
            <th>by_ref</th>
            <th>by_mut</th>
            <th>by_value</th>
            <th>move by_ref</th>
            <th>move by_mut</th>
            <th>move by_value</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <th scope="row">FnOnce</th>
            <td>✅</td>
            <td>✅</td>
            <td>✅</td>
            <td>✅</td>
            <td>✅</td>
            <td>✅</td>
        </tr>
        <tr>
            <th scope="row">FnMut</th>
            <td>✅</td>
            <td>✅</td>
            <td>❌</td>
            <td>✅</td>
            <td>✅</td>
            <td>❌</td>
        </tr>
        <tr>
            <th scope="row">Fn</th>
            <td>✅</td>
            <td>❌</td>
            <td>❌</td>
            <td>✅</td>
            <td>❌</td>
            <td>❌</td>
        </tr>
    </tbody>
</table>

We can see that the `move` keyword has no impact on the implemented trait. It only changes the capture to be from reference to value.

For example, the desugaring of `by_ref_closure` is:


```rust
struct ByRefStruct<'a> {
    data: &'a String,
}

impl<'a> FnOnce<()> for ByRefStruct<'a> {
    type Output = ();
    extern "rust-call" fn call_once(self, args: ()) -> Self::Output {
        self.call(args)
    }
}

impl<'a> FnMut<()> for ByRefStruct<'a> {
    extern "rust-call" fn call_mut(&mut self, args: ()) -> Self::Output {
        self.call(args)
    }
}

impl<'a> Fn<()> for ByRefStruct<'a> {
    extern "rust-call" fn call(&self, (): ()) -> Self::Output {
        by_ref(self.data)
    }
}
```

whereas for `move_by_ref_closure`:

```rust
struct MoveByRefStruct {
    data: String,
}

impl FnOnce<()> for MoveByRefStruct {
    type Output = ();
    extern "rust-call" fn call_once(self, args: ()) -> Self::Output {
        self.call(args)
    }
}

impl FnMut<()> for MoveByRefStruct {
    extern "rust-call" fn call_mut(&mut self, args: ()) -> Self::Output {
        self.call(args)
    }
}

impl Fn<()> for MoveByRefStruct {
    extern "rust-call" fn call(&self, (): ()) -> Self::Output {
        by_ref(&self.data)
    }
}
```

Notice how the `data` field changed from `&'a String` to `String` and the call to `by_ref` from `self.data` to `&self.data` eventhough in the closure forms we had `by_ref(&data)` in both cases.


So we now hopefully understand what the `move` keyword does but you might wonder why that can be useful? After all, the first table above shows that we only removed flexbility.

Spawning a thread:
```rust
let data = "by_ref".to_owned();
std::thread::spawn(|| by_ref(&data)).join().unwrap();
```

Without `move`, we get the following compiler error which helpfully suggest adding `move`:

```
error[E0373]: closure may outlive the current function, but it borrows `data`, which is owned by the current function
 --> src/main.rs:9:20
  |
9 | std::thread::spawn(|| by_ref(&data)).join().unwrap();
  |                    ^^         ---- `data` is borrowed here
  |                    |
  |                    may outlive borrowed value `data`
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:9:1
  |
9 | std::thread::spawn(|| by_ref(&data)).join().unwrap();
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
help: to force the closure to take ownership of `data` (and any other referenced variables), use the `move` keyword
  |
9 | std::thread::spawn(move || by_ref(&data)).join().unwrap();
  |                    ++++
```

Creating a function returning a closure:

```rust
fn make_greeter(greeter: &str) -> impl Fn(&str) -> String {
    move |name| format!("{greeter} {name}")
}

let hello_greeter = make_greeter("Hello");
let hi_greeter = make_greeter("Hi");

assert_eq!(hello_greeter("rust"), "Hello rust");
assert_eq!(hi_greeter("rust"), "Hi rust");
```

Here too we need `move` otherwise we get the same borrow checker error.


# Last word

~~Sigh~~

This article is long enough as is, so I am stopping here for now.
I plan to publish a follow up article for async closures later.
If you want to read more on the subject I recommend:
1. The [closure chapter in the rust book](https://doc.rust-lang.org/book/ch13-01-closures.html).
2. The [closure chapter in the rust reference](https://doc.rust-lang.org/reference/types/closure.html).
3. [Finding Closure in Rust](https://huonw.github.io/blog/2015/05/finding-closure-in-rust/) by Huon Wilson.
4. The [article from the baby steps blog about adding an explicit capture clause](https://smallcultfollowing.com/babysteps/blog/2025/10/22/explicit-capture-clauses/).
5. The Rust Unstable book on [fn_traits](https://doc.rust-lang.org/beta/unstable-book/library-features/fn-traits.html) and [unboxed_closures](https://doc.rust-lang.org/beta/unstable-book/language-features/unboxed-closures.html).

And to discuss this article, you can head over to the [Hacker News thread](https://news.ycombinator.com/item?id=46746266) or the [reddit thread](https://www.reddit.com/r/rust/comments/1qluyre/understanding_rust_closures/).
