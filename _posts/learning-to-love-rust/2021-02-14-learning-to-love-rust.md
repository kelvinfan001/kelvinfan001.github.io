---
title: Learning to Love the Most Loved Programming Language
date: 2021-02-14
tags: []
description: Comparing Vec<T>, &[T], [T], [T;n], Box<[T]>
tag:
  - Rust
  - programming
  - strings
image: /learning-to-love-rust/does_not_compile.svg
---

<img width="200px" src="/assets/img/learning-to-love-rust/does_not_compile.svg" alt="Confused Ferris"/>

# Becoming a Rustacean
For five years in a row, Rust has been the most loved programming language according to the [StackOverflow Developer Survey](https://stackoverflow.blog/2020/06/05/why-the-developers-who-use-rust-love-it-so-much/). As a tech enthusiast and student of computer science, of course I had to give this hip new successor to C a shot (the fact that my job required it might have played a role, as well).

Despite the trend of many species in nature to eventually evolve into crabs, or so the internet tells me, I find the natural process of carcinization a bit too slow for my needs. But speeding up the process of becoming a Rustacean has not been easy. With a compiler stricter than my fifth grade Chinese teacher, I can confidently say that Rust was one of the most difficult, if not _the_ most difficult, languages that I've learned, and, yes, that includes natural languages, too.

A huge part of what made Rust so difficult to pick up for me was how different it is compared to other languages. From ownership to lifetimes, there are a bunch of new concepts I'd love to talk about, but I'll start with one of the most basic and commonly used types in Rust.

# What is a String _slice_??
The last time I was so confused by strings, I was still in my high school orchestra. To be fair, the actual confusion stems from the more general vectors, slices, and arrays; but I think it is helpful to start with string-ish types as an example, since contiguous characters is the earliest manifestation of these types that I encountered. 

####  The `String` type
At first sight, the `String` type doesn't seem so special.

{% highlight rust %}
let my_string: String = String::from("I love Rust");
{% endhighlight %}

`my_string` is a `String` type, it is made of three components: a pointer to some bytes, a length, and a capacity.

{% highlight rust %}
use std::mem::size_of_val;

// The `size_of_val()` function returns the size of the 
// pointed-to value in bytes.
println!("{}", size_of_val(&my_string)); // 24
{% endhighlight %}

The size of `my_string` is 24 bytes, or 3 words (assuming 64-bit architecture). These 3 words are used to store the pointer, length, and capacity of the string. The pointer points to a buffer on the heap, the length indicates the number of bytes on the buffer (not null-terminated), and the capacity is used to facilitate re-allocating memory in the buffer (e.g. appending to strings, etc.).

#### The `&String` type
This is a reference to a `String`. A reference is like a pointer from C (which represents a memory location), but references are never invalid (Null) and you can't do pointer arithmetic on them.

{% highlight rust %}
let reference_to_my_string: &String = &my_string;
println!("{}", size_of_val(&reference_to_my_string)); // 8
{% endhighlight %}

Loosely speaking, `reference_to_my_string` is just a pointer that points to a pointer (and some metadata) that points to a buffer on the heap. Since it's just a pointer that points to some memory location, the size of a `&String` is just 1 word.

So far so good...

#### The `&str` type
This is the type that was totally new to me, but used all the time in Rust.
As we can see from the ampersand before it, this type is a _reference_ to something. That something is a string slice (`str`). But before digging into slices, let's look at `&str`'s size.

{% highlight rust %}
// The `as_str()` method of the `String` type extracts a string slice
// containing the entire String and returns a reference to it.
let reference_to_my_string_slice: &str = my_string.as_str();
println!("{}", size_of_val(&reference_to_my_string_slice)); // 16
{% endhighlight %}

Hmm, isn't `reference_to_my_string_slice` just a pointer that points to some memory location? The size of `reference_to_my_string` is 1 word, how come a reference to the string slice version of `my_string` is double the size? `reference_to_my_string_slice` is a "fat pointer". Normally, I wouldn't concern myself with the BMI of pointers (I believe that it's really a personal matter), but size does matter here. It not only contains a pointer to the string slice's memory location, but also the _length_ of the string slice. So why do we need this extra bit of data? We'll need to examine the type that `&str` points to. This is where things got weird for me.

#### The `str` type
The _string slice_ (AKA `str`) type is a _dynamically sized_ type. Dynamically sized types do not have a size at compile time. `String` is a _normal_ type, the compiler always know its size --- it's always 3 words. A `str` is some number of instances of `u8` (unsigned 8-bit integer) laid out sequentially in memory, but the exact number if unknown. The compiler cannot compute the size of a `str` because it has no length. Because of this, we can't put a `str` on the stack; in other words, we can never have a variable that is a `str`. So if we try to compile the following,

{% highlight rust %}
// Note that we try to derefence with `*` to get the actual `str`.
let my_string_slice: str = *my_string.as_str();
{% endhighlight %}

The compiler complains:
```
  |     let my_string_slice: str = *my_string.as_str();
  |         ^^^^^^^^^^^^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `Sized` is not implemented for `str`
  = note: all local variables must have a statically known size
```

Basically, I like to think that `*my_string.as_str()` _is_ the data in `my_string`'s buffer. There's nothing encoded in the buffer on the heap that tells us where this contiguous block of `u8`s end, so all we know about `*my_string.as_str()` is that there is a block of `u8`s, and they're _guaranteed_ to be `u8`s, lying next to each other somewhere. Importantly, this contiguous sequence doesn't necessarily need to be on the heap.

#### The string literal
{% highlight rust %}
let my_string_literal: &str = "I literally love Rust";
println!("{}", size_of_val(&my_string_literal)); // 16
{% endhighlight %}

String literals are really just `&str`s. The `my_string_literal` variable has a size of 2 words; it's a "fat pointer", just as we expected. `my_string_literal` is a reference that points to a contiguous sequence of `u8`s; but unlike the `str` that `reference_to_my_string_slice` points to, the contiguous sequence of `u8`s that `my_string_literal` points to is stored inside the binary itself!

Now, `*string_literal` actually has a known size, and the compiler could probably know it (of course, it's _literally_ hardcoded), but treating it as a `str` is already sufficient to do everything we need. Since a `str` is just _any_ sequence of characters, we don't care where it is, be it on the heap, as with `reference_to_my_string_slice`, or in the binary itself, as with `my_string_literal`. So because we have this `str` type, whenever we have hardcoded "strings" in a program, we can just view it as a `str` and not have to worry about copying it onto the heap or the stack.
For example, if we want to compare a variable like so:

{% highlight rust %}
if foo == "potentially very long string that is very long" {
    // do stuff with `foo`
}
{% endhighlight %}

Since `"potentially very long string that is very long"` is really a slice, we don't need to allocate any space for it on the heap (and then free it immediately afterwards), or even put it on the stack. All we have is a pointer, albeit a fat one (memory location and size).

#### What about arrays?
A string slice is _basically_ a slice of `u8`s, or `[u8]` (there's actually a slight difference in that `str`s are guaranteed to be valid UTF-8 text), and a `String` is analogous to a vector of `u8`s, or `Vec<u8>`. Rust has a more general slice type, the `[T]`, which just means a contiguous sequence of type `T`s _somewhere_ of _some_ length. 

There is also the array type, `[T; n]`, which is a contiguous sequence of type `T`'s _somewhere_ of length `n`. Sounds almost like a slice, but with arrays, the compiler knows the length. So we can put an array of four `i64` types on the stack like this:

{% highlight rust %}
let my_array: [i32; 4] = [1, 2, 3, 4];
println!("{}", size_of_val(&my_array)); // 16
{% endhighlight %}

Arrays sound pretty nice, they aren't some weird dynamic, semi-mythical type like `[T]` or `str`, we can actually put them on the stack (or the heap); they have a _size_. But I think the fact that they have a size is exactly why they are so rarely seen in Rust. Because the size is a part of the type, they are not so convenient to pass around. For example:

{% highlight rust %}
let my_big_array: [i32; 5] = [1, 2, 3, 4, 5];
let my_small_array: [i32; 3] = [1, 2, 3];
{% endhighlight %}

`my_big_array` and `my_small_array` are different types, so if we want to have a function that takes in an array and prints out each of its values in a new line, what would the signature of the function be? This is where slices are better.
We can get a slice view of `my_array` (but we have to put the slices behind references):

{% highlight rust %}
let reference_to_slice_of_my_big_array: &[i32] = &my_big_array[..];
let reference_to_slice_of_my_small_array: &[i32] = &my_small_array[..];
{% endhighlight %}

This way, the type of our function's argument could just be `&[i32]`. 
Also note that both `reference_to_slice_of_my_big_array` and `reference_to_slice_of_my_small_array` are fat pointers that point to `str`s on the stack!

#### Conclusion

From these examples, we see how flexible slices could be: whenever we have a function that would like to interact with a contiguous sequence of some type, we could just pass it a `&[T]` (or `&str`). We don't need to worry about whether that sequence is an array or a vector, or whether it's in the binary, stack, or heap, a slice always offers us a nice view into that block of data.

#### Extra: Slices on the heap
References aren't the only way to interact with slices; for example, it is also possible to interact with slices through _container types_ like "boxes". The `Box` type is a pointer type for heap allocation. Still using `my_string` from way above:

{% highlight rust %}
let my_string_in_a_box: Box<str> = my_string.into_boxed_str();
println!("{}", size_of_val(&my_string_in_a_box)); // 16
{% endhighlight %}

`my_string_in_a_box` is of type `Box<str>` and is only 2 words because all it stores is a pointer and length. `Box<str>` is sort of like `&str`, except the `str` that `my_string_in_a_box` points to is guaranteed to be on the heap. `Box<str>` is also sort of like `String`, except it doesn't have the capacity field, so we cannot resize `my_string_in_a_box`. There is also the `Vector` equivalent method of `into_boxed_slice()` for the more general vectors.

As far as I can tell, boxed slices aren't too common, since `String`s and `Vector`s can do everything that a `Box<str>` or `Box<[T]>` can, but I think it's helpful to illustrate that slices don't _have_ to be behind references, they can be behind anything, so long as it has some way to point to where the slice is, and a size.

# References
- https://stackoverflow.com/questions/61151041/i-dont-understand-the-difference-between-a-slice-and-reference-rust
- http://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/second-edition/ch04-03-slices.html
- http://smallcultfollowing.com/babysteps/blog/2014/01/05/dst-take-5/
- https://www.reddit.com/r/rust/comments/9jnp7u/why_is_a_slice_a_dst/e6t5qm8/
- https://stackoverflow.com/questions/24158114/what-are-the-differences-between-rusts-string-and-str
- https://users.rust-lang.org/t/use-case-for-box-str-and-string/8295/3
- https://users.rust-lang.org/t/when-would-you-want-to-use-a-boxed-array/46658/2
- https://doc.rust-lang.org/std/string/struct.String.html
- https://doc.rust-lang.org/book/ch04-03-slices.html
- https://stackoverflow.com/questions/57754901/what-is-a-fat-pointer
- https://stackoverflow.com/questions/63572130/why-are-string-literals-str-instead-of-string-in-rust
- https://stevedonovan.github.io/rust-gentle-intro/1-basics.html
