
<h1 align="center">OK Redis</h1>
<p align="center">
    <a href="LICENSE"><img src="https://badgen.net/github/license/kristoff-it/zig-heyredis" /></a>
    <a href="https://twitter.com/croloris"><img src="https://badgen.net/badge/twitter/@croloris/1DA1F2?icon&label" /></a>
</p>

<p align="center">
    Zero-allocation client for Redis 6+
</p>

## Handy and Efficient
This client aims to offer an interface with great ergonomics 
without compromising on performance or flexibility: if it 
makes sense, it's going to be straightforward, and if it's 
possible, you're going to be able to do it.


## Zero dynamic allocations, unless explicitly wanted

The client has two main interfaces to send commands: `send` and `sendAlloc`. Following Zig's mantra of making dynamic allocations explicit, only `sendAlloc` can allocate dynamic memory, and only does so by using a user-provided allocator. 

The way this is achieved is by making good use of RESP3's typed responses and Zig's metaprogramming facilities.
The library uses compile-time reflection to specialize down to the parser level, allowing heyredis to decode whenever possible a reply directly into a function frame, **without any intermediate dynamic allocation**. If you want more information about Zig's comptime:
- [Official documentation](https://ziglang.org/documentation/master/#comptime)
- [What is Zig's Comptime?](https://kristoff.it/blog/what-is-zig-comptime) (blog post written by me)

By using `sendAlloc` you can decode replies with arbrirary shape at the cost of occasionally performing dynamic allocations. The interface takes an allocator as input, so the user can setup custom allocation schemes such as [arenas](https://en.wikipedia.org/wiki/Region-based_memory_management).

## Quickstart

```zig
const std = @import("std");
const okredis = @import("./src/okredis.zig");
const SET = okredis.commands.SET;
const OrErr = okredis.OrErr;
const Client = okredis.Client;

pub fn main() !void {
    var client: Client = undefined;
    try client.initIp4("127.0.0.1", 6379);
    defer client.close();

    // Base interface
    try client.send(void, .{ "SET", "key", "42" });
    const reply = try client.send(i64, .{ "GET", "key" });
    if (reply != 42) @panic("out of towels");


    // Command builder interface
    const cmd = SET.init("key", "43", .NoExpire, .IfAlreadyExisting);
    const otherReply = try client.send(OrErr(void), cmd);
    switch (otherReply) {
        .Nil => @panic("command should not have returned nil"),
        .Err => @panic("command should not have returned an error"),
        .Ok => std.debug.warn("success!"),
    }

    // Transactions
    const tr_reply = try client.transaction(OrErr(struct {
        c1: OrErr(FixBuf(10)),
        c2: u64,
        c3: OrErr(void),
    }), .{
        SET.init("banana", 1, .NoExpire, .NoConditions),
        .{ "INCR", "counter" },
        .{ "INCR", "banana" },
    });

    switch (tr_reply) {
        .Err => |e| @panic(e.getCode()),
        .Nil => @panic("got nil"),
        .Ok => |reply| {
            std.debug.warn("{} {} {}\n", .{
                reply.c1.Ok.toSlice(),
                reply.c2,
                reply.c3.Err.getCode(),
            });
        },
    }
}
```

## Available Documentation
1. Using OkRedis Commands
2. 


## TODOS
- Design Zig errors
- Better connection handling (buffering, ...)
- Refine support for async/await
- Pub/Sub
- Refine the Redis traits
