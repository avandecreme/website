+++
title = "Understanding rust async closures"
draft = true
date = "2026-01-18"
description = "Explorations on what are rust async closures and how they work under the hood."
# taxonomies.tags = [
#     "rust",
# ]
+++

# Async closures

## AsyncFnOnce

By adding the `async` keyword before the closure, we can make it return a `Future`. For example, the following closure will sleep 1 second before returning it last word.

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

So, very similar to our previous `FnOnce` example. However, async closures implement `AsyncFnOnce` instead.

Below is the desugaring however, I won't explain in details the mechanism because it also require explaining async functions desugaring with state machines. If the topic interest you, I recommand two articles:
1. [Futures in Rust: An In-Depth Technical Analysis](https://www.gencmurat.com/en/posts/futures-in-rust/) by Murat Genc.
2. [Understanding Rust futures by going way too deep](https://fasterthanli.me/articles/understanding-rust-futures-by-going-way-too-deep) by Amos (fasterthanlime).

Note that you need to enable `#![feature(async_fn_traits)]` in nightly.

```rust
struct AsyncDropStruct {
    last_word: String,
}

struct AsyncDropStructFuture<'a> {
    last_word: Option<String>,
    sigh: &'a str,
    state: AsyncDropStructFutureState,
}

enum AsyncDropStructFutureState {
    Initial,
    SleepAwaitPoint {
        future: Pin<Box<dyn Future<Output = ()>>>,
    },
}

impl<'a> Future for AsyncDropStructFuture<'a> {
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
                        let last_word = self
                            .last_word
                            .take()
                            .expect("`last_word` should have a value at this point");
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

impl<'a> AsyncFnOnce<(&'a str,)> for AsyncDropStruct {
    type CallOnceFuture = AsyncDropStructFuture<'a>;
    type Output = String;
    extern "rust-call" fn async_call_once(
        self,
        (sigh,): (&'a str,),
    ) -> Self::CallOnceFuture {
        AsyncDropStructFuture {
            last_word: Some(self.last_word),
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

## AsyncFnMut

In the same way we had `FnMut` for non-async closures, we have `AsyncFnMut` for async ones.
For example:

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

With it desugaring:

```rust
struct AsyncPusherStruct<'a> {
    v: &'a mut Vec<i32>,
}

struct AsyncPusherStructFuture<'a> {
    v: &'a mut Vec<i32>,
    arg: i32,
    state: AsyncPusherStructFutureState,
}

enum AsyncPusherStructFutureState {
    Initial,
    SleepAwaitPoint {
        future: Pin<Box<dyn Future<Output = ()>>>,
    },
}

impl<'a> Future for AsyncPusherStructFuture<'a> {
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

impl<'a> AsyncFnOnce<(i32,)> for AsyncPusherStruct<'a> {
    type CallOnceFuture = AsyncPusherStructFuture<'a>;
    type Output = ();
    extern "rust-call" fn async_call_once(self, args: (i32,)) -> Self::CallOnceFuture {
        AsyncPusherStructFuture {
            v: self.v,
            arg: args.0,
            state: AsyncPusherStructFutureState::Initial,
        }
    }
}

impl<'a> AsyncFnMut<(i32,)> for AsyncPusherStruct<'a> {
    type CallRefFuture<'b>
        = AsyncPusherStructFuture<'b>
    where
        Self: 'b;

    extern "rust-call" fn async_call_mut(&mut self, args: (i32,)) -> Self::CallRefFuture<'_> {
        AsyncPusherStructFuture {
            v: self.v,
            arg: args.0,
            state: AsyncPusherStructFutureState::Initial,
        }
    }
}

let mut v = vec![];
let mut async_adder_struct = AsyncPusherStruct { v: &mut v };
async_adder_struct(5).await;
async_adder_struct(4).await;
assert_eq!(vec![5, 4], v);
```

Notice how in the same way `FnMut` requires an `FnOnce` implementation, `AsyncFnMut` requires an `AsyncFnOnce` implementation.

## AsyncFn

Finally, we also have `AsyncFn` closures:

```rust
let last_word = "last word: ".to_owned();
let async_greeter_closure = async |sigh| {
    tokio::time::sleep(Duration::from_secs(1)).await;
    String::new() + &last_word + sigh
};

// We can access `last_word` here:
assert_eq!("last word: ".to_owned(), last_word);

assert_eq!("last word: sigh!", async_greeter_closure("sigh!").await);
// We can call async_greeter_closure again
assert_eq!("last word: sigh!", async_greeter_closure("sigh!").await);

// We can access `last_word` here too
assert_eq!("last word: ".to_owned(), last_word);
```

And the desugaring:

```rust
struct AsyncGreeterStruct<'a> {
    last_word: &'a String,
}

struct AsyncGreeterStructFuture<'a, 'b> {
    last_word: &'a String,
    sigh: &'b str,
    state: AsyncGreeterStructFutureState,
}

enum AsyncGreeterStructFutureState {
    Initial,
    SleepAwaitPoint {
        future: Pin<Box<dyn Future<Output = ()>>>,
    },
}

impl<'a, 'b> Future for AsyncGreeterStructFuture<'a, 'b> {
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
                        let res = String::new() + &self.last_word + self.sigh;
                        Poll::Ready(res)
                    }
                    Poll::Pending => Poll::Pending,
                }
            }
        }
    }
}

impl<'a, 'b> AsyncFnOnce<(&'b str,)> for AsyncGreeterStruct<'a> {
    type CallOnceFuture = AsyncGreeterStructFuture<'a, 'b>;
    type Output = String;
    extern "rust-call" fn async_call_once(
        self,
        (sigh,): (&'b str,),
    ) -> Self::CallOnceFuture {
        AsyncGreeterStructFuture {
            last_word: self.last_word,
            sigh,
            state: AsyncGreeterStructFutureState::Initial,
        }
    }
}


impl<'a, 'b> AsyncFnMut<(&'b str,)> for AsyncGreeterStruct<'a> {
    type CallRefFuture<'c>
        = AsyncGreeterStructFuture<'a, 'b>
    where
        Self: 'c;

    extern "rust-call" fn async_call_mut(
        &mut self,
        (sigh,): (&'b str,),
    ) -> Self::CallRefFuture<'_> {
        AsyncGreeterStructFuture {
            last_word: self.last_word,
            sigh,
            state: AsyncGreeterStructFutureState::Initial,
        }
    }
}

impl<'a, 'b> AsyncFn<(&'b str,)> for AsyncGreeterStruct<'a> {
    extern "rust-call" fn async_call(&self, (sigh,): (&'b str,)) -> Self::CallRefFuture<'_> {
        AsyncGreeterStructFuture {
            last_word: self.last_word,
            sigh,
            state: AsyncGreeterStructFutureState::Initial,
        }
    }
}

let last_word = "last word: ".to_owned();
let async_greeter_struct = AsyncGreeterStruct {
    last_word: &last_word,
};
assert_eq!("last word: sigh!", async_greeter_struct("sigh!").await);
assert_eq!("last word: sigh!", async_greeter_struct("sigh!").await);
```

# Last word

~~Sigh~~

That ended up being a long article and there are still things I didn't mention.
If you want to read more on the subject I recommend:
1. The [closure chapter in the rust book](https://doc.rust-lang.org/book/ch13-01-closures.html)
2. The [closure chapter in the rust reference](https://doc.rust-lang.org/reference/types/closure.html)
3. The [article from the baby steps blog about adding an explicit capture clause](https://smallcultfollowing.com/babysteps/blog/2025/10/22/explicit-capture-clauses/) 

That last article is actually the one which made me realized that there was a lot I didn't know about closures and made me write this article. I hope you learned a few things as well.
