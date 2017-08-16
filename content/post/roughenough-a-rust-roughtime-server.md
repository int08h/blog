+++
date = "2017-08-14T19:01:26-05:00"
title = "Roughenough: A Rust implementation of Roughtime"
description = "A brief overview of Roughenough, a Roughtime implementation in Rust."

+++

# In Brief

A quick note regarding [Roughenough](https://github.com/int08h/roughenough), a simple [Roughtime](https://roughtime.googlesource.com/roughtime) server written in Rust which I released in July.

There are some comments about Roughenough on the 
[/r/rust subreddit](https://www.reddit.com/r/rust/comments/6lths4/a_roughtime_secure_time_sync_server_written_in/) including the author of the superb [Ring](https://github.com/briansmith/ring) cryptography library weighing-in on Rust Merkle Tree implementation.

For those network programmers interested in how Rust's ownership semantics and first-class byte slices (`&[u8]`) contrast with buffer abstractions in a managed language (`ByteBuf` in Netty), compare Roughenough with its spiritual cousin, [Nearenough](https://github.com/int08h/roughenough). 

Finally if you're new to the Roughtime protocol check out 
this [deep-dive on the Roughtime protocol](https://int08h.com/post/to-catch-a-lying-timeserver/) and
this exploration of [Roughtime on-the-wire messages](https://int08h.com/post/roughtime-message-anatomy/)
to get up to speed.

