+++
title = "Understanding rust async closures"
date = "2026-02-22"
description = "Explorations on what are rust async closures and how they work under the hood."
# taxonomies.tags = [
#     "rust",
# ]
+++

In the [previous article](rust-closures), we saw how "normal" closures are desugared. The goal of this article is to see how it is done for async closures.

However, we first need to understand how an async function is desugared.
As in the first article, all titles are links to rust playgrounds where you can experiment with the code in the section.

# [Async fn desugaring](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=da3a9118851df1a40e87486e672bb1ca)

Let's consider this simple async function which sleeps 1 second before returning
a `String`:
```rust
async fn last_word(sigh: &str) -> String {
    tokio::time::sleep(Duration::from_secs(1)).await;
    "last word: ".to_owned() + sigh
}

assert_eq!("last word: sigh!", last_word("sigh!").await);
```

As you might already know, an `async` function returning a `String` returns a value implementing `Future<Output = String>`.

Here is the definition of the `Future` trait:
```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

Let's try to implement that trait for a struct having a `sigh` field corresponding to our `sigh` parameter above:

```rust
struct LastWordFuture<'sigh> {
    sigh: &'sigh str,
    state: LastWordFutureState,
}
```

You probably wonder what is that `state` field. When, desugaring an async function,
the rust compiler builds a state machine with each state corresponding to an await point. In our case:

```rust
enum LastWordFutureState {
    Initial,
    SleepAwaitPoint {
        future: Pin<Box<dyn Future<Output = ()>>>,
    },
}
```
Since we have only one `await`, we just have the `Initial` state and the `SleepAwaitPoint`. If we had more `await` in the function, we would have one
extra variant per `await` point.

We are now ready to implement `Future` for `LastWordFuture`:

```rust
impl<'sigh> Future for LastWordFuture<'sigh> {
    type Output = String;
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        match self.state {
            LastWordFutureState::Initial => {
                let future = Box::pin(tokio::time::sleep(Duration::from_secs(1)));
                self.state = LastWordFutureState::SleepAwaitPoint { future };
                self.poll(cx)
            }
            LastWordFutureState::SleepAwaitPoint { ref mut future } => {
                match future.as_mut().poll(cx) {
                    Poll::Ready(()) => {
                        Poll::Ready("last word: ".to_owned() + self.sigh)
                    }
                    Poll::Pending => Poll::Pending,
                }
            }
        }
    }
}
```

We first set `type Output = String` because our future returns a `String`.
Then, we implement `poll` by matching on `state`. If we are on the `Initial` state, we call `tokio::time::sleep` and store the resulting
future in the new `SleepAwaitPoint` state. We then recursively call `poll` again.
On this new call to `poll` we are on the `SleepAwaitPoint` state and we call `poll` on the `future` stored in the `SleepAwaitPoint`.
If that future is `Ready`, we are able to advance and return the result in a `Ready` variant. Otherwise, we return `Pending` and the
asynchronous runtime is in charge of calling `poll` again until `Ready` is returned.

Here is how we can use our future struct:

```rust
let sigh = "sigh!";
let last_word_future = LastWordFuture {
    sigh,
    state: LastWordFutureState::Initial,
};
assert_eq!("last word: sigh!", last_word_future.await);
```

Notice how we can use `.await` on our `LastWordFuture` struct instance.

# Async closures

## [AsyncFnOnce](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2024&gist=3bc28f4dce6330a2c4093496bff7d982)

Similarly to functions, one can add the `async` keyword before a closure, to make it possible to `await` inside it.
The closure is then returning a `Future`. For example, the following closure will sleep 1 second before returning its last word.

```rust
let last_word = "last word: ".to_owned();
let async_drop_closure = async |sigh| {
    tokio::time::sleep(Duration::from_secs(1)).await;
    let res = String::new() + &last_word + sigh;
    drop(last_word);
    res
};

// We can't access `last_word` here:
// assert_eq!("last word: ".to_owned(), last_word);
// error[E0382]: borrow of moved value: `last_word`

assert_eq!("last word: sigh!", async_drop_closure("sigh!").await);

// We can't access `last_word` here either
// assert_eq!("last word: ".to_owned(), last_word);
// error[E0382]: borrow of moved value: `last_word`

// And we can't call async_drop_closure again
// assert_eq!("last word: sigh!", async_drop_closure("sigh!").await);
// error[E0382]: use of moved value: `async_drop_closure`
```

As you can see, the borrowing rules are very similar to our [previous article `FnOnce` example](rust-closures/#fnonce-trait). However, async closures implement [`AsyncFnOnce`](https://doc.rust-lang.org/std/ops/trait.AsyncFnOnce.html) instead.

Here is the trait definition:
```rust
pub trait AsyncFnOnce<Args>
where
    Args: Tuple,
{
    type CallOnceFuture: Future<Output = Self::Output>;
    type Output;

    // Required method
    extern "rust-call" fn async_call_once(
        self,
        args: Args,
    ) -> Self::CallOnceFuture;
}
```

It is very similar to the `FnOnce` trait. The main difference is the `CallOnceFuture` type which is returned by the `async_call_once` method.

Here is how we can desugar the previous closure.

Note that you need to use the nightly channel and enable `#![feature(async_fn_traits)]` and `#![feature(unboxed_closures)]`.

First, similarly to non-async closures, we need to declare a struct with the captured variables:
```rust
struct AsyncDropStruct {
    last_word: String,
}
```

Secondly, similarly to the async function desugaring, we will need to implement a future with a state machine.

```rust
struct AsyncDropStructFuture<'sigh> {
    last_word: String,
    sigh: &'sigh str,
    state: AsyncDropStructFutureState,
}

enum AsyncDropStructFutureState {
    Initial,
    SleepAwaitPoint {
        future: Pin<Box<dyn Future<Output = ()>>>,
    },
}

impl<'sigh> Future for AsyncDropStructFuture<'sigh> {
    type Output = String;
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        match self.state {
            AsyncDropStructFutureState::Initial => {
                let future = Box::pin(tokio::time::sleep(Duration::from_secs(1)));
                self.state = AsyncDropStructFutureState::SleepAwaitPoint { future };
                self.poll(cx)
            }
            AsyncDropStructFutureState::SleepAwaitPoint { ref mut future } => {
                match future.as_mut().poll(cx) {
                    Poll::Ready(()) => {
                        let last_word = std::mem::take(&mut self.last_word);
                        let res = String::new() + &last_word + self.sigh;
                        drop(last_word);
                        Poll::Ready(res)
                    }
                    Poll::Pending => Poll::Pending,
                }
            }
        }
    }
}
```

As you can see it is very similar to the desugaring of the `last_word` function.
The main difference is that we need to store the captured `last_word` variable in our future struct and in the `Poll::Ready` branch we need to extract its value with [`std::mem::take`](https://doc.rust-lang.org/std/mem/fn.take.html) to be able to pass it to `drop`.

Finally, we are ready to implement `AsyncFnOnce`:

```rust
impl<'sigh> AsyncFnOnce<(&'sigh str,)> for AsyncDropStruct {
    type CallOnceFuture = AsyncDropStructFuture<'sigh>;
    type Output = String;
    extern "rust-call" fn async_call_once(
        self,
        (sigh,): (&'sigh str,),
    ) -> Self::CallOnceFuture {
        AsyncDropStructFuture {
            last_word: self.last_word,
            sigh,
            state: AsyncDropStructFutureState::Initial,
        }
    }
}

let async_drop_struct = AsyncDropStruct {
    last_word: "last word: ".to_owned(),
};
assert_eq!("last word: sigh!", async_drop_struct("sigh!").await);
```

Not much done here, we already did most of the work before. `async_call_once` just returns an `AsyncDropStructFuture` instance.
As you can see, we can write `async_drop_struct("sigh!").await`, using both the function call syntax together with `.await` on our `AsyncDropStruct` instance.

## [AsyncFnMut](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2024&gist=d55ad3d42000186ab161e268839ce124)

In the same way we had `FnMut` for non-async closures, we have [`AsyncFnMut`](https://doc.rust-lang.org/std/ops/trait.AsyncFnMut.html) for async ones.

Here is its definition:
```rust
pub trait AsyncFnMut<Args>: AsyncFnOnce<Args>
where
    Args: Tuple,
{
    type CallRefFuture<'a>: Future<Output = Self::Output>
       where Self: 'a;

    // Required method
    extern "rust-call" fn async_call_mut(
        &mut self,
        args: Args,
    ) -> Self::CallRefFuture<'_>;
}
```

As expected, one must first implement `AsyncFnOnce` to be able to implement `AsyncFnMut`.
More surprising is that we need another future `CallRefFuture<'a>` in addition to the `CallOnceFuture` we already have thanks to the implementation of `AsyncFnOnce`. We will see why later. For now, let's try to desugar the following closure:

```rust
let mut v = vec![];
let mut async_pusher_closure = async |x| {
    tokio::time::sleep(Duration::from_secs(1)).await;
    v.push(x);
};

async_pusher_closure(5).await;
async_pusher_closure(2).await;
assert_eq!(vec![5, 2], v);
```

Like before, we need to define the struct together with its future:

```rust
struct AsyncPusherStruct<'v> {
    v: &'v mut Vec<i32>,
}

struct AsyncPusherStructFuture<'v> {
    v: &'v mut Vec<i32>,
    arg: i32,
    state: AsyncPusherStructFutureState,
}

enum AsyncPusherStructFutureState {
    Initial,
    SleepAwaitPoint {
        future: Pin<Box<dyn Future<Output = ()>>>,
    },
}

impl<'v> Future for AsyncPusherStructFuture<'v> {
    type Output = ();
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        match &mut self.state {
            AsyncPusherStructFutureState::Initial => {
                let future = Box::pin(tokio::time::sleep(Duration::from_secs(1)));
                self.state = AsyncPusherStructFutureState::SleepAwaitPoint { future };
                self.poll(cx)
            }
            AsyncPusherStructFutureState::SleepAwaitPoint { future } => {
                match future.as_mut().poll(cx) {
                    Poll::Ready(()) => {
                        let arg = self.arg;
                        self.v.push(arg);
                        Poll::Ready(())
                    }
                    Poll::Pending => Poll::Pending,
                }
            }
        }
    }
}
```

This is very similar to what we did with `AsyncFnOnce`, the main difference is that we need to introduce a lifetime for our captured mutable reference.

And now the `AsyncFnOnce` and `AsyncFnMut` implementations:

```rust
impl<'v> AsyncFnOnce<(i32,)> for AsyncPusherStruct<'v> {
    type CallOnceFuture = AsyncPusherStructFuture<'v>;
    type Output = ();
    extern "rust-call" fn async_call_once(self, (arg,): (i32,)) -> Self::CallOnceFuture {
        AsyncPusherStructFuture {
            v: self.v,
            arg,
            state: AsyncPusherStructFutureState::Initial,
        }
    }
}

impl<'v> AsyncFnMut<(i32,)> for AsyncPusherStruct<'v> {
    type CallRefFuture<'this>
        = AsyncPusherStructFuture<'this>
    where
        Self: 'this;

    extern "rust-call" fn async_call_mut(&mut self, (arg,): (i32,)) -> Self::CallRefFuture<'_> {
        AsyncPusherStructFuture {
            v: self.v,
            arg,
            state: AsyncPusherStructFutureState::Initial,
        }
    }
}

let mut v = vec![];
let mut async_pusher_struct = AsyncPusherStruct { v: &mut v };
async_pusher_struct(5).await;
async_pusher_struct(2).await;
assert_eq!(vec![5, 2], v);
```

You might wonder why we repeat ourselves in `async_call_once` and `async_call_mut` instead of calling one from the other.
We definitely can't call `async_call_once` from `async_call_mut` because `async_call_mut` only has a reference to `self` but `async_call_once` needs a value.
We can try the other way around though:

```rust
extern "rust-call" fn async_call_once(mut self, args: (i32,)) -> Self::CallOnceFuture {
    self.async_call_mut(args)
}
```

If we try to compile, we get:
```
error[E0515]: cannot return value referencing function parameter `self`
  --> src/main.rs:64:5
   |
64 |     self.async_call_mut(args)
   |     ----^^^^^^^^^^^^^^^^^^^^^
   |     |
   |     returns a value referencing data owned by the current function
   |     `self` is borrowed here
```
It doesn't work either! It is because while the two functions return an `AsyncPusherStructFuture<'_>`, they do not have the same lifetime.
The one returned by `async_call_once` has the lifetime of `v` but the one returned by `async_call_mut` has the lifetime of `AsyncPusherStruct`.

## [AsyncFn](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2024&gist=2ed6c9ccfe8a952f44d022d6e1833711)

Finally, we also have `AsyncFn` closures. For this example, we will force the capture by value with the `move` keyword.

```rust
let hello = "Hello ".to_owned();
let async_greeter_closure = async move |world| {
    tokio::time::sleep(Duration::from_secs(1)).await;
    String::new() + &hello + world
};

assert_eq!("Hello world!", async_greeter_closure("world!").await);
assert_eq!("Hello world!", async_greeter_closure("world!").await);
```

Since we are forcing the capture by value, this desugars to the following struct (`hello` is a `String`, not a `&String`):

```rust
struct AsyncGreeterStruct {
    hello: String,
}
```

The future and its implementation:

```rust
struct AsyncGreeterStructValFuture<'world> {
    hello: String,
    world: &'world str,
    state: AsyncGreeterStructFutureState,
}

enum AsyncGreeterStructFutureState {
    Initial,
    SleepAwaitPoint {
        future: Pin<Box<dyn Future<Output = ()>>>,
    },
}

impl<'world> Future for AsyncGreeterStructValFuture<'world> {
    type Output = String;
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        match self.state {
            AsyncGreeterStructFutureState::Initial => {
                let future = Box::pin(tokio::time::sleep(Duration::from_secs(1)));
                self.state = AsyncGreeterStructFutureState::SleepAwaitPoint { future };
                self.poll(cx)
            }
            AsyncGreeterStructFutureState::SleepAwaitPoint { ref mut future } => {
                match future.as_mut().poll(cx) {
                    Poll::Ready(()) => {
                        let res = String::new() + &self.hello + self.world;
                        Poll::Ready(res)
                    }
                    Poll::Pending => Poll::Pending,
                }
            }
        }
    }
}
```

And now we can first implement `AsyncFnOnce`:

```rust
impl<'world> AsyncFnOnce<(&'world str,)> for AsyncGreeterStruct {
    type CallOnceFuture = AsyncGreeterStructValFuture<'world>;
    type Output = String;
    extern "rust-call" fn async_call_once(
        self,
        (world,): (&'world str,),
    ) -> Self::CallOnceFuture {
        AsyncGreeterStructValFuture {
            hello: self.hello,
            world,
            state: AsyncGreeterStructFutureState::Initial,
        }
    }
}
```

Next step is `AsyncFnMut`:

```rust
impl<'world> AsyncFnMut<(&'world str,)> for AsyncGreeterStruct {
    type CallRefFuture<'hello>
        = AsyncGreeterStructValFuture<'world>
    where
        Self: 'hello;

    extern "rust-call" fn async_call_mut(
        &mut self,
        (world,): (&'world str,),
    ) -> Self::CallRefFuture<'_> {
        AsyncGreeterStructValFuture {
            hello: self.hello,
            world,
            state: AsyncGreeterStructFutureState::Initial,
        }
    }
}
```

Oops, the compiler is not happy:
```
error[E0507]: cannot move out of `self.hello` which is behind a mutable reference
    |
116 |                 hello: self.hello,
    |                        ^^^^^^^^^^ move occurs because `self.hello` has type `String`, which does not implement the `Copy` trait
```

Since we only have a mutable reference to `self`, we can only get a reference to `self.hello` but we need an owned value to build `AsyncGreeterStructValFuture`.
In this case, we could `clone` the value with a small performance penalty but should the captured variables not implement `Clone` we could not even do that.
The proper fix is to introduce a new future which takes `hello` by reference:

```rust
struct AsyncGreeterStructRefFuture<'hello, 'world> {
    hello: &'hello String,
    world: &'world str,
    state: AsyncGreeterStructFutureState,
}

impl<'hello, 'world> Future for AsyncGreeterStructRefFuture<'hello, 'world> {
    type Output = String;
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        match self.state {
            AsyncGreeterStructFutureState::Initial => {
                let future = Box::pin(tokio::time::sleep(Duration::from_secs(1)));
                self.state = AsyncGreeterStructFutureState::SleepAwaitPoint { future };
                self.poll(cx)
            }
            AsyncGreeterStructFutureState::SleepAwaitPoint { ref mut future } => {
                match future.as_mut().poll(cx) {
                    Poll::Ready(()) => {
                        let res = String::new() + self.hello + self.world;
                        Poll::Ready(res)
                    }
                    Poll::Pending => Poll::Pending,
                }
            }
        }
    }
}
```

And with that we can implement both `AsyncFnMut` and `AsyncFn`:

```rust
impl<'world> AsyncFnMut<(&'world str,)> for AsyncGreeterStruct {
    type CallRefFuture<'hello>
        = AsyncGreeterStructRefFuture<'hello, 'world>
    where
        Self: 'hello;

    extern "rust-call" fn async_call_mut(
        &mut self,
        args: (&'world str,),
    ) -> Self::CallRefFuture<'_> {
        self.async_call(args)
    }
}

impl<'world> AsyncFn<(&'world str,)> for AsyncGreeterStruct {
    extern "rust-call" fn async_call(
        &self,
        (world,): (&'world str,),
    ) -> Self::CallRefFuture<'_> {
        AsyncGreeterStructRefFuture {
            hello: &self.hello,
            world,
            state: AsyncGreeterStructFutureState::Initial,
        }
    }
}
```

Notice how we can just call `async_call` from `async_call_mut` here. We have the same types and lifetimes and it is not a problem to get `&self` from `&mut self`.
Using this struct is as you would expect:

```rust
let async_greeter_struct = AsyncGreeterStruct {
    hello: "Hello ".to_owned(),
};
assert_eq!("Hello world!", async_greeter_struct("world!").await);
assert_eq!("Hello world!", async_greeter_struct("world!").await);
```


# Further reading

If you want to read more on the subject I recommend:

1. [Futures in Rust: An In-Depth Technical Analysis](https://www.gencmurat.com/en/posts/futures-in-rust/) by Murat Genc.
2. [Understanding Rust futures by going way too deep](https://fasterthanli.me/articles/understanding-rust-futures-by-going-way-too-deep) by Amos (fasterthanlime).
3. [Async Closures](https://hackmd.io/@compiler-errors/async-closures) by Michael Goulet who implemented that feature.
4. [Giving, lending, and async closures](https://smallcultfollowing.com/babysteps/blog/2023/05/09/giving-lending-and-async-closures/) by Niko Matsakis.
5. The [async closure RFC](https://rust-lang.github.io/rfcs/3668-async-closures.html).
