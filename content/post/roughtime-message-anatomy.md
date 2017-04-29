+++
title = "Roughtime Message Anatomy"
date = "2017-04-27T16:38:00-05:00"
description = "On-the-wire client and server Roughtime messages."

+++

This post A) digs into the details of
[Roughtime](https://roughtime.googlesource.com/roughtime) protocol messages, 
and B) demonstrate use of the [Nearenough](https://github.com/int08h/nearenough) client library[^1].
It's intended as a reference more than anything.

Familiarity with Roughtime is assumed. If needed, consult 
[this prior post]({{< ref "post/to-catch-a-lying-timeserver.md" >}}) for an intoduction
to the protocol and how it works. 

[^1]: Using commit `523ff4a`

# Nearenough Client API

`RoughtimeClient` is the high-level Nearenough API for interacting with a Roughtime server. It provides
six methods:

* `createRequest()` - Constructs a RoughTime client request
* `processResponse(RtMessage)` - Validates the server's response
* `isResponseValid()` - Indicates if the response passed validation
* `midpoint()` and `radius()` - Returns the server's provided time value (midpoint) and uncertainty (radius) if the response was valid
* `invalidResponseCause()` - If the response is invalid, provides the reason why validation failed

The constructor requires one argument: the remote Roughtime server's long-term key. We'll be interacting 
with Google's server (`roughtime.sandbox.google.com`) using the long-term public key found in the project's 
[list](https://roughtime.googlesource.com/roughtime/+/master/roughtime-servers.json) of servers.

```java
  // Long-term public key of Google's Roughtime server
  byte[] GOOGLE_SERVER_PUBKEY = hexToBytes(
      "7ad3da688c5c04c635a14786a70bcf30224cc25455371bf9d4a2bfb64b682534"
  );
  
  // Create a new RoughtimeClient instance using Nearenough client API
  RoughtimeClient client = new RoughtimeClient(GOOGLE_SERVER_PUBKEY);
```

The single-argument constructor uses 
[`SecureRandom`](https://docs.oracle.com/javase/8/docs/api/java/security/SecureRandom.html) 
to generate 64 random bytes for the nonce and should be sufficient for the majority of users.
For expert use, constructor overloads exist if controlling the random 
number source or explicitly providing the nonce value are required.

# Client Requests

All Roughtime operations are initiated by a client request. Requests must 
be >=1024 bytes long and all have two tags: 

1. `NONC` with the 64-byte client nonce.
2. `PAD` to pad the request up to the 1024 byte minimum size.

## Creating a Request

Creating a request is a one-liner:

```java
  RtMessage request = client.createRequest();
```

## Pretty-Printing

`RtMessage` overrides `toString` for convenient pretty-printing, so `System.out.println(request)` outputs:

```text
RtMessage|2|{
  NONC(64) = aaacc1a6de530026f2500721b078967107734e173755f3dc6019218bffb1ce8bcfb1a87144386f45af0f1c5ce41bca4ebfeb727d27fe7a7d6baa9b08a3b50f68
  PAD(944) = 000...000
}
```

How to read the output:

* `RtMessage|2|` indicates the message has two tags.
* `NONC(64)` tag with a 64 byte long nonce value (`aaacc1a6...`)
* `PAD(944)` tag with 944 bytes of zeros

## Encoding for Transmission

Nearenough uses [Netty](https://netty.io/) to implement the network
layer. Netty's "buffer" abstraction is the [ByteBuf](https://netty.io/4.1/api/io/netty/buffer/ByteBuf.html).
Static methods on `RtWire` encode an `RtMessage` into a `ByteBuf` for network transmission. 

```java
  ByteBuf buf = RtWire.toWire(request);
  // ready to send 'buf', e.g. ctx.writeAndFlush(buf)

  // Or, if you're in the NIO world and need a Java ByteBuffer
  ByteBuffer nioBuf = buf.nioBuffer();
```

## On-The-Wire Bytes

To view the request's bytes as they'll be sent on-the-wire, use Netty's `ByteBufUtil.prettyHexDump()`:

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

Walking through the bytes (keep in mind that integers are in little-endian byte order):

| Byte Range | Value | Description |
|---|---|---|
| `0x00` - `0x03` | `02 00 00 00` | Number of tags in the message; 2 in decimal |
| `0x04` - `0x07` | `40 00 00 00` | Offset to start of values; 64 in decimal |
| `0x08` - `0x0b` | `4e 4f 4e 43` | `NONC` tag marker |
| `0x0c` - `0x0f` | `50 41 44 ff` | `PAD` tag marker |
| `0x10` - `0x4f` | `aa ac c1 a6...` | Client nonce value (64 bytes long) |
| `0x50` - `0x3ff` | `00 00 00 00...` | Padding zero bytes (944 bytes long) |

# Server Response

A Roughtime server response is a message with five tags. The values of some tags are
themselves Roughtime messages.

## Response Structure 

```text
RtMessage|5|{
  SIG(64) = fd06a4fb305f4df36e4e6f19941d0e4108d79d2879261ba03acbf48ae3d9fd60525dbfd21534c99f45145fa614afbbdad026437a6f6f6670452dee6766dd8003
  PATH(0) = 
  SREP(100) = RtMessage|3|{
    RADI(4) = 40420f00
    MIDP(8) = e36344212d4e0500
    ROOT(64) = 0e321361f19c96319484f7b7a5915f5f312702e4dd962cc6183361bfd32b4c096f75e8a254aecb612eb9b9c6aefdb1ed609884af19d6dff18cc091aaf2b79d86
  }
  CERT(152) = RtMessage|2|{
    SIG(64) = 29d589e9aaee25e00a2cdf019dcf848a99280fdf03310e00decb36c02535f8d66f79f3c12f69ccd93cf9978dc4c23f2c06b7ebc674c153c4452a42386dc4290f
    DELE(72) = RtMessage|3|{
      PUBK(32) = b411a29d262537cf175c55af4ad2f01155cc9e7bf37ac6502739124acb6bcf25
      MINT(8) = 00e02fe2284e0500
      MAXT(8) = 00c064778d4e0500
    }
  }
  INDX(4) = 00000000
}
```

Working through the response:

* `SIG` a 64 byte Ed25519 signature of the `SREP` message using the `PUBK` key in the `CERT` message. 
* `PATH` empty here, but potentially contains the Merkle Tree path needed to verify that the client nonce was included in the response.
* `SREP` ("signed response") a nested message containing:
  * `ROOT` root of the Merkle Tree 
  * `MIDP` ("midpoint") remote server's time
  * `RADI` ("radius") width of the valid time span
* `CERT` a nested message:
  * `SIG` a 64 byte Ed25519 signature of the `DELE` message using the server's long-term public key. 
  * `DELE` a nested message:
      * `PUBK` ("public key") a time-limited 32 byte Ed25519 public key
      * `MINT` ("minimum time") start of the time period when `PUBK` is valid
      * `MAXT` ("maximum time") end of the time period when `PUBK` is valid
* `INDX` index locating the nonce in the Merkle Tree `PATH`; zero here as `PATH` is empty.
 
## Response Validation

The `processResponse(RtMessage)` method will validate the server's response. It checks that:

0. The message is well-formed and obeys the Roughtime protocol spec.
1. The signature of the `DELE` message, using the server's long-term key, is valid.
2. The top-level signature on `SREP`, using the `PUBK` key in the `DELE` message, is valid.
3. The request's nonce is included in the response's Merkle tree.
4. The midpoint `MIDP` lies within the delegation time bounds (in between `MINT` and `MAXT`).

Use `isResponseValid()` to check the validation result:

```java
  if (client.isResponseValid()) {
    // Validation passed, the response is good
  } else {
    // Validation failed. Print out the reason why.
    System.out.println("Response INVALID: " + client.invalidResponseCause().getMessage());
  }
```

## The Midpoint and Radius

In a valid response, the *midpoint* is the Roughtime server's reported UTC timestamp (in microseconds). 
The *radius* is a span of uncertainty around that midpoint. A Roughtime server asserts that
its "true time" lies within the span.

```java
  Instant midpoint = Instant.ofEpochMilli(client.midpoint() / 1_000L);
  int radiusSec = client.radius() / 1_000_000;
  System.out.println("midpoint: " + midpoint + " (radius " + radiusSec + " sec)");

  // midpoint: 2017-04-27T22:03:42.178Z (radius 1 sec)
```
