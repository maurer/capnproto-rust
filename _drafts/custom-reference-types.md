---
layout: post
title: custom mutable reference types
author: dwrensha
---

Rust's *mutable references* provide exclusive access to writable locations in memory.
If we have `x : &'a mut T`,
then we know that the referred-to `T`
cannot be read or modified except through dereferencing `x`.
In other words, `x` can have no aliases.
This guarantee is
crucial for memory safety,
as it implies that
any mutations we apply through
`x` have no risk of invalidating references held
elsewhere in our program.


The `Builder` types
of [capnproto-rust](https://github.com/dwrensha/capnproto-rust)
also need to provide an exclusivity guarantee.
Recall that if `Foo` is a struct defined in a Cap'n Proto schema,
then a `foo::Builder<'a>`
provides access to a writable location
in arena-allocated memory that contains
a `Foo` in [Cap'n Proto format](https://kentonv.github.io/capnproto/encoding.html).
To protect access to that memory, a `foo::Builder<'a>` ought to behave
as if it were a `&'a mut Foo`,
even though the `Foo` type
cannot directly exist in Rust
(because Cap'n Proto struct layout
differs from Rust struct layout).

Thus, we need to define `foo::Builder<'a>` as a custom mutable reference.

Built-in mutable references have some special semantics in Rust.

To mimic them, we need to do some things manually.


pass-by-move and reborrowable.
non-`Copy` and reborrowable.

Suppose we declare a Cap'n Proto schema like this

```
struct Foo {
  x @0 : Float32;
  blob @1 : Data;
}
```

When we call `capnp compile -orust foo.capnp`, we will get generated code
containing definitions of `Foo::Reader<'a>` and `Foo::Builder<'a>`

like a `&'a Foo` and `&'a mut Foo`

```
mod foo {
  pub struct Builder<'a> {...}

  impl <'a> Builder<'a> {
    pub fn get_x(self) -> f32 {...}
    pub fn set_x(&mut self, value : f32) {...}
    pub fn get_blob(self) -> ::capnp::data::Builder<'a> {...}
    pub fn set_blob(&mut self, value : ::capnp::data::Reader) {...}
    pub fn init_blob(self, length : u32) -> ::capnp::data::Builder<'a> { ... }
  }
}
```


```
fn init_and_return_slice<'a>(foo : foo::Builder<'a>) -> &'a mut [u8] {
    foo.init_blob(100).slice_mut(5, 10)
}
```

What if we want to call this and then we want to call `set_x()`?
We can't get our foo back once it has been passed by move


```
fn do_some_things_wrong<'a>(mut foo : foo::Builder<'a>) {
   let slice = init_and_return_slice(foo);
   slice[0] = 42;
   foo.set_x(1.23);

}
```


```
main.rs:17:9: 17:12 error: cannot borrow `foo` as mutable more than once at a time
main.rs:17         foo.set_x(1.23);
                   ^~~
```


```
impl <'a> Builder <'a> {
     pub fn borrow<'b>(&'b mut self) -> Builder<'b> { ... }
}
```

```
fn do_some_things_right<'a>(mut foo : foo::Builder<'a>) {
    {
        let slice = init_and_return_slice(foo.borrow());
        slice[0] = 42
    }
    foo.set_x(1.23);
}
```



```
struct Bar {
  oneFoo @0 : Foo;
  manyFoos @1 : List(Foo);
}
```


::capnp::struct_list::Reader<'a, Foo::Reader<'a>>


::capnp::data::Reader<'a> = &'a [u8];
::capnp::data::Builder<'a> = &'a mut [u8];


