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
To protect access to this memory, a `foo::Builder<'a>` ought to act
like a `&'a mut Foo`.
Note that `Foo` here is a pretend type
that cannot directly exist in Rust,
because Cap'n Proto struct layout
differs from Rust struct layout.


For convenience, mutable references
have some implicit semantics.

exposes some of the implicit semantics
of `& mut T`.

The lifetime `'a` tracks the scope of `x`.


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


```
impl <'a> Builder <'a> {
     pub fn borrow<'b>(&'b mut self) -> Builder<'b> { ... }
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


