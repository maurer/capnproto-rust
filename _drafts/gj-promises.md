---
layout: post
title: asynchronous I/O with promises
author: dwrensha
---


My Rust [implementation](https://github.com/dwrensha/capnp-rpc-rust)
of the Cap'n Proto remote procedure call protocol
was designed in a bygone era.
Back then, Rust's runtime library
provided thread-like "tasks"
that were backed by [libgreen](https://github.com/alexcrichton/green-rs)
and were therefore "cheap to spawn."
These enabled
[CSP](http://en.wikipedia.org/wiki/Communicating_sequential_processes)-style
programming
with beautifully simple blocking I/O operations
that were, under the hood,
dispatched through [libuv](https://github.com/libuv/libuv).
While the question of whether this model was actually efficient
was a matter of much [discussion](https://github.com/rust-lang/rfcs/pull/219),
I personally enjoyed using it and found it
easy to reason about.


For better or worse, the era of libgreen
[has ended](https://github.com/rust-lang/rfcs/blob/master/text/0230-remove-runtime.md).
Code originally written for libgreen can still work,
but because each "task" is now its own system-level thread,
calling them "lightweight" is more of a stretch than ever.
As I've maintained capnp-rpc-rust over the past year,
its need for a different approach to concurrency
has become increasingly apparent.


## Announcing [GJ](https://github.com/dwrensha/gj)

[GJ](https://github.com/dwrensha/gj) is a new Rust library that provides
abstractions for concurrency and asynchronous I/O.
The main ideas in GJ are taken from
[KJ](https://capnproto.org/cxxrpc.html#kj-concurrency-framework),
the library that forms the foundation of capnproto-c++.
At [Sandstorm](https://sandstorm.io), we have been
successfully using KJ-based concurrency
in our core infrastructure for a while now,
including in a
[bridge](https://github.com/sandstorm-io/sandstorm/blob/3a3e93eb142969125aa8573df4edc6c62efbeebe/src/sandstorm/sandstorm-http-bridge.c++) that translates between
HTTP and [this Cap'n Proto interface](https://github.com/sandstorm-io/sandstorm/blob/3a3e93eb142969125aa8573df4edc6c62efbeebe/src/sandstorm/web-session.capnp),
and in a
[Cap'n Proto driver](https://github.com/sandstorm-io/sandstorm/blob/3a3e93eb142969125aa8573df4edc6c62efbeebe/src/sandstorm/fuse.c++)
to a FUSE filesystem.


{% highlight rust %}
pub fn foo(addr: gj::io::NetwordAddress) -> gj::Promise<()> {
    return addr.connect().then(|stream| {
       stream.write(vec![]).then(|(stream, buf)| {

       });
    });
}
{% endhighlight %}






