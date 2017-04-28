+++
title = "Roughtime Message Anatomy"
date = "2017-04-27T16:38:00-05:00"
description = "On-the-wire client and server Roughtime messages."

+++

This is a technical deep-dive examining the on-the-wire messages of the [Roughtime](https://roughtime.googlesource.com/roughtime) 
protocol. Familiarity with Roughtime is assumed. If needed this 
[prior post]({{< ref "post/to-catch-a-lying-timeserver.md" >}}) has an overview of the 
protocol and how it works. 

Code below uses my Roughtime client implementation, [Nearenough](https://github.com/int08h/nearenough),
to create and display the Roughtime messages[^1].

[^1]: Using commit `523ff4a`

# Creating a Nearenough `RoughtimeClient` Instance

`RoughtimeClient` is the high-level Nearenough API for interacting with a Roughtime server. 
The constructor requires one argument: the server's long-term key. We'll be interacting 
with Google's server (`roughtime.sandbox.google.com`) and its long-term public key is found in the project's 
[list](https://roughtime.googlesource.com/roughtime/+/master/roughtime-servers.json) of servers.

```java
  // Long-term public key of Google's Roughtime server
  byte[] GOOGLE_SERVER_PUBKEY = hexToBytes(
      "7ad3da688c5c04c635a14786a70bcf30224cc25455371bf9d4a2bfb64b682534"
  );
  
  // Create a new RoughtimeClient instance using Nearenough client API
  RoughtimeClient client = new RoughtimeClient(GOOGLE_SERVER_PUBKEY);
```

# Client Request 

All Roughtime operations are initiated by a client request which
must be >=1024 bytes long. The request has two tags: 

1. `NONC` with the 64-byte client nonce.
2. `PAD` to pad the request up to the 1024 byte minimum size.

## Creating A Request

Creating a request is a one-liner:

```java
  RtMessage request = client.createRequest();
```

## Pretty-Printing

`RtMessage` overrides `toString` to pretty-print messages so `System.out.println(request)` outputs:

```text
RtMessage|2|{
  NONC(64) = aaacc1a6de530026f2500721b078967107734e173755f3dc6019218bffb1ce8bcfb1a87144386f45af0f1c5ce41bca4ebfeb727d27fe7a7d6baa9b08a3b50f68
  PAD(944) = 000...000
}
```

How to read the output:

* `RtMessage|2|` indicates the message has two tags.
* `NONC(64)` tag with a 64 byte long nonce value (`aaacc1a6d...`)
* `PAD(944)` tag with 944 bytes of zeros

## Encoding for Transmission

The current version of Nearenough uses the excellent [Netty](https://netty.io/) to implement the network
layer. Netty's "buffer" abstraction is the [ByteBuf](https://netty.io/4.1/api/io/netty/buffer/ByteBuf.html).
Static methods on `RtWire` encode an `RtMessage` into a `ByteBuf` for network transmission. 

```java
  ByteBuf buf = RtWire.toWire(request);
  // ready to send 'buf', e.g. ctx.writeAndFlush(buf)

  // Or, if you're in the NIO world and need a Java ByteBuffer
  ByteBuffer nioBuf = buf.nioBuffer();
```

## Dissecting the On-The-Wire Message

The on-the-wire encoded message can be hex-dumped via Netty's 
`ByteBufUtil.prettyHexDump()`:

```text
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 02 00 00 00 40 00 00 00 4e 4f 4e 43 50 41 44 ff |....@...NONCPAD.|
|00000010| aa ac c1 a6 de 53 00 26 f2 50 07 21 b0 78 96 71 |.....S.&.P.!.x.q|
|00000020| 07 73 4e 17 37 55 f3 dc 60 19 21 8b ff b1 ce 8b |.sN.7U..`.!.....|
|00000030| cf b1 a8 71 44 38 6f 45 af 0f 1c 5c e4 1b ca 4e |...qD8oE...\...N|
|00000040| bf eb 72 7d 27 fe 7a 7d 6b aa 9b 08 a3 b5 0f 68 |..r}'.z}k......h|
|00000050| 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|
|00000060| 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|
...
|000003f0| 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 |................|
+--------+-------------------------------------------------+----------------+
```

