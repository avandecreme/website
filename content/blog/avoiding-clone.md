+++
title = "Tips and tricks to avoid cloning"
date = "2026-04-04"
description = "Some tricks to avoid calling clone in Rust"
draft = false
# taxonomies.tags = [
#     "rust",
# ]
+++

In Rust, it is often better to avoid cloning if possible to avoid unnecessary allocation.  
Here is a list of techniques I use most often to do that.
All titles are links to Rust playgrounds if you want to play with the examples.

# [Implement the Copy trait](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=affc900742f8d23783ba03e1d8067d44)

For small data types, cloning might actually be the cheapest option.  
There is a special trait for that named [Copy](https://doc.rust-lang.org/std/marker/trait.Copy.html).

Whenever possible, you should implement that trait on your data types. It makes the code a lot simpler because you don't have to worry about ownership at all. The borrow checker will let you pass your types implementing `Copy` around freely, for example:

```rust
#[derive(Copy, Clone)]
struct Point {
   x: i32,
   y: i32,
}

fn draw_point(Point {x, y}: Point) {
    println!("Drawing {x}, {y}");
}

let point = Point { x: 1, y: -4 };

// We can pass point by value indefinitely.
// To make the compiler unhappy, try removing Copy from the derive annotation above.
draw_point(point);
draw_point(point);
draw_point(point);
```

See [the doc](https://doc.rust-lang.org/std/marker/trait.Copy.html) to know when you can, should and can't implement `Copy` for your type.

In practice, the most common usage I have for it is for [newtypes](https://doc.rust-lang.org/rust-by-example/generics/new_types.html) and enums:
```rust
#[derive(Copy, Clone)]
struct NewType(i32);

#[derive(Copy, Clone)]
enum SomeEnum {
    Variant1,
    Variant2(i32),
}
```

# [Take parameters by reference](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=51d201ccd1417ff3c05ba227aad6877f)

Oftentimes, a function doesn't need to take ownership of its parameters and is able to work with a reference, for example:

```rust
fn print_by_val(value: String) {
    println!("Printing {value}");
}

fn print_by_ref(value: &String) {
    println!("Printing {value}");
}

let money = String::from("money");
// To print infinite money with print_by_val, we must expensively clone repeatedly
print_by_val(money.clone());
print_by_val(money.clone());
print_by_val(money.clone());

// But we get free money with print_by_ref!
print_by_ref(&money);
print_by_ref(&money);
print_by_ref(&money);
```

Note that `print_by_ref` should ideally take a `&str` instead of a `&String`, see this [clippy lint](https://rust-lang.github.io/rust-clippy/master/index.html#ptr_arg). I use `&String` here to make the difference between `print_by_val` and `print_by_ref` clearer. 

# [Use the proper iterator](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=85ead1464b1bdb6107d93cb9bc3d04f9)

Depending on whether you only need a reference or ownership of the values of an iterable, you should use `.iter()` or `.into_iter()` (or `.iter_mut()` for a mutable reference).

```rust
struct NewType(String);

let data = vec![String::from("foo"), String::from("bar"), String::from("baz")];

// If you don't need `data` afterward, avoid
let new_types = data.iter().map(|elem| NewType(elem.clone())).collect::<Vec<_>>();

// Prefer
let new_types = data.into_iter().map(NewType).collect::<Vec<_>>();
```

Similarly, by default a `for` loop is similar to `.into_iter()`, so call `.iter()` first if you don't want to drop your data and only need a reference:

```rust
let data = vec![String::from("foo"), String::from("bar"), String::from("baz")];

// If you need `data` afterward, avoid
for elem in data.clone() {
    println!("{elem}");
}

// Prefer
for elem in data.iter() {
    println!("{elem}");
}
```

# [Have closures capturing by value also return the value](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=b9b649b220cc5cf8308b086458caf505)

Sometimes you are forced to have a closure capture by value, for example because the closure is sent to another thread. However, the closure doesn't really need to consume the value and you need to use it after the closure call.
In that situation, the trick is to have the captured variable be returned by the closure:

```rust
let world = String::from("world");

// Avoid
let cloned_world = world.clone();
let computed = std::thread::spawn(move || format!("Hello {cloned_world}!"))
    .join()
    .unwrap();
println!("{computed}");

// Prefer
let (world, computed) = std::thread::spawn(|| {
    let computed = format!("Hello {world}!");
    (world, computed)
})
.join()
.unwrap();
println!("{computed}");

// We still have our world here
println!("Hello {world}!");
```

Note that if you are not clear about the use of the `move` keyword here, you might want to read [my article on closures](../rust-closures).


# Conclusion

This is not an exhaustive list but those are the techniques I use the most often.
If you would like to discuss them or share your own, you can do so on [Reddit](https://https://www.reddit.com) or [Hacker News](https://news.ycombinator.com/).
