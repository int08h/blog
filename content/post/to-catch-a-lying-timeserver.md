+++
date = "2017-04-13T14:06:58-05:00"
title = "Scalable, Lie-Detecting Timeserving with Roughtime"
description = "A look at Roughtime, an internet-scale secure time synchronization protocol that addresses legacy shortcomings."

+++

\[*Addendum*: related [Hacker News](https://news.ycombinator.com/item?id=14164662) discussion\]

[Roughtime](https://roughtime.googlesource.com/roughtime/) is a protocol designed to provide internet-scale secure time synchronization and address shortcomings of older protocols like NTP.

# Why This Matters

Many aspects of day-to-day computing assume an accurate local clock. This assumption extends to security-critical operations like 
certificate expiry, OSCP stapling, and Kerberos tickets. An attacker that subverts time sync could violate the security of these operations.

The dominant internet time-sync protocol, Network Time Protocol (NTP), is showing its age. Amongst other things:

1. It is unauthenticated[^1] and most clients blindly trust the reply of any server (`pool.ntp.org` anyone?). 
2. Clients that find an NTP server responding with bad values have no way to prove to others that a particular server is a bad actor[^2].
3. There are protocol (mis)features that can be abused to create DDoS traffic[^3].

[^1]: Yes, there are [authentication mechanisms](https://tools.ietf.org/html/rfc5906) in NTPv4 as well as in-development NTP/PTP authentication protocols like [NTS](https://tools.ietf.org/html/draft-ietf-ntp-network-time-security-14). None of these have seen widespread deployment on the public internet, in mobile devices, or as out-of-the-box OS defaults. Furthermore these solutions require that clients unconditionally trust one or more servers and/or require computationally expensive handshakes.  

[^2]: "Proof" in this case being a cryptographically secure record that can be audited by an untrusted third party. Think spiritual cousin of [certificate transparency](https://www.certificate-transparency.org/). 

[^3]: Granted correctly configured NTP servers will not have these issues...and all internet accessible devices are always correctly configured, right?

# Roughtime Goals

Roughtime attempts to improve on the status quo. Paraphrasing its design goals:

1. **Accuracy in seconds** -- The protocol aims to get local clocks within a few seconds of the "true" time. The Roughtime creators state that [25% of Chrome's certificate errors](https://roughtime.googlesource.com/roughtime#Roughstime-2) are caused by [incorrect local clocks](https://groups.google.com/a/chromium.org/forum/#!msg/security-dev/oj2xXq3CF0E/f7BtsfkVhe8J) (off by *days* or more). A local clock within a few seconds of reference time is good enough.
2. **Secure** -- Roughtime servers sign every reply and the signature is verifiable by any client. Clients can build a chain of replies to establish proof of server misbehavior. This proof is also verifiable by any client. No central trust is required.
3. **Internet scalable** -- The protocol is stateless and built upon efficient and batchable operations. The network layer is constructed to prevent DDoS amplification. 
4. **Healthy ecosystem** -- Roughtime includes mechanisms for ensuring "freshness" of clients and correctness of their implementations. The intent is to surface bugs quickly and mitigate abusive client practices[^4].

[^4]: See [NTP Server Misuse and Abuse](https://en.wikipedia.org/wiki/NTP_server_misuse_and_abuse)

Bellow I'll explore some of the interesting aspects of Roughtime that may not be immediately obvious. 

> # Aside: Roughtime In a Nutshell
>
> For those unfamiliar, below is a **very** abbreviated description of how Roughtime works. The [specification](https://roughtime.googlesource.com/roughtime/+/HEAD/PROTOCOL.md) and [project page](https://roughtime.googlesource.com/roughtime/) details the actual protocol:
> 
> 1. A brand new client generates a 64 byte random [nonce](https://en.wikipedia.org/wiki/Cryptographic_nonce) and sends it to a Roughtime server.
> 2. The Roughtime server replies with the current time, the client's nonce, and a signature of both.
> 3. Subsequent client requests generate their nonce by combining the reply in #2 with a random value (a blind[^5]).
> 
> Clients know that server responses are A) *fresh* because they include their nonce, and B) *authentic* because they are signed.
> 
> By deriving *future* requests from *prior* responses, the client establishes a chain of [happens-before](https://en.wikipedia.org/wiki/Happened-before) ordering between requests and replies. 
> 
> Sending chained requests to *multiple* servers extends the total order across all participating servers (e.g. send `A` to server1 and receive `A'`; send `B = sha512(A')` to server2 and receive `B'`; send `C = sha512(B')` to server3 and receive `C'`, and so on). 

[^5]: The randomness makes subsequent request values unpredictable.

# Scalable Request Processing

A Roughtime server signs its responses using the [Ed25519](https://ed25519.cr.yp.to/) public-key signature system. 
Ed25519 signatures are quite fast: according to [eBATS](https://bench.cr.yp.to/results-sign.html) 
an Intel Skylake CPU takes about ~49,000 cycles to compute an Ed25519 signature. Assuming a 3.0 GHz 
clock speed that's ~61,000 signatures per second on a single core. Signatures are also compact: 64 bytes each.

To scale the signing workload, Roughtime uses a [Merkle Tree](https://en.wikipedia.org/wiki/Merkle_tree) 
to sign a **batch** of client requests with a single signature operation. The **root** of the tree is signed and included in all responses.
With a batch size[^6] of 64 a single 3.0 GHz Skylake core can sign **3.9 million** requests per second. 

[^6]: Batching sizes are not part of the specification. 64 is used as it makes a convenient example and corresponds to a reasonable UDP buffer size (64 KiB).

Another design feature helps limit per-client processing: the non-client-specific parts of a response are **identical** for all replies in a single batch[^7]. Servers can calculate these values once and re-use them in each reply. 

[^7]: Specifically only the `PATH` and `INDX` tags vary response-to-response. The `SIG`, `SREP`, and `CERT` tags will be identical in all responses. 

# Anti-Amplification Request/Response Size Asymmetry 

All Roughtime requests are required to be at least 1024 bytes long and any request shorter is simply dropped. Why? To ensure that a Roughtime server cannot be used as 
a [DDoS amplifier](https://www.incapsula.com/ddos/attack-glossary/ntp-amplification.html). 

| Roughtime Message    | Size (bytes) |
| ----- | -----:|
| Client Request  | 1024 |
| Server Response to a single request | 360 |
| Server Response when batching 64 requests | 744 |

The 1 KiB request size requirement ensures an attacker has no size gain on traffic sent to a Roughtime server; replies are always smaller than requests, even under load (when batching responses).

This also naturally rate-limits requests: at 1 KiB per request a 10 Gbps link can deliver a maximum of 1.2 million
requests per second. As shown above, that rate is easily handled by a single Skylake core.

# Keeping Response Sizes Compact

You probably noticed the response size when batching 64 requests is suspiciously small: 744 bytes. This is not 
a typo. 

Recall that Roughtime uses a Merkle Tree of client requests to construct its batched response. A Roughtime server does not send the whole tree when replying to clients. It sends the signed root plus *only* those nodes each client needs to verify that its request was included in the tree[^8]. Being a [complete tree](http://web.cecs.pdx.edu/~sheard/course/Cs163/Doc/FullvsComplete.html), the *maximum* number of path elements required by any client is `ceil(log2(batch size))`. 

[^8]: This elegant idea shows up in [many](https://www.certificate-transparency.org/log-proofs-work) [other](https://blog.ethereum.org/2015/11/15/merkling-in-ethereum/) [places](https://petertodd.org/2016/opentimestamps-announcement).

An example might help. Consider this simplified Merkle Tree constructed from seven requests, `A` though `G`:

```text
             _____ h(ABCD|EFG) _____            
            /                       \
       h(AB|CD)                    h(EF|G)
      /       \                    /     \
   h(A|B)     h(C|D)            h(E|F)   h(G)
   /   \       /   \            /   \
 h(A)  h(B)  h(C)  h(D)       h(E)  h(F)

h(...) = SHA512
```

Imagine constructing a response to client C that needs to verify its request `C` is in the tree:

1. To compute `h(C|D)` the client needs `h(D)` (client C remembers the `h(C)` it sent).
2. To compute `h(AB|CD)` the client already has `h(C|D)` from step #1, so it needs `h(A|B)`.
3. For the root, `h(ABCD|EFG)`, the client has `h(AB|CD)` from step #2 and needs `h(EF|G)`.

Thus client C is sent `h(D)`, `h(A|B)`, and `h(EF|G)`.

Hopefully it's clearer how the 744 byte response size for a server batching 64 requests is obtained: `ceil(log2(64)) == 6` and `6 * 64 bytes per sha512 == 384 bytes`. Minimum Roughtime response size of 360 bytes + 384 bytes of tree path data = 744 bytes total.

# Catching Lying Servers

Now to examine how request/response chaining allows clients to detect and tattle on misbehaving servers. The diagram below illustrates a sequence of chained requests and responses between one client and two servers:

{{< figure src="/images/roughtime_chaining.png" link="/images/roughtime_chaining.png" caption="Figure 1 - Roughtime request/response chaining (click to enlarge)" class="figshrink" >}}

Proof of server misbehavior emerges from two interlocking properties of the protocol: 

1. **Chaining establishes total ordering between all request/responses.** Composing requests using prior responses establishes a single sequence of events that can be independently verified. 
2. **All responses are cryptographically signed.** Signatures establish authenticity of responses, identity of senders, and prevent forgery. 

A client sending nonce `B` which was derived from `A` can be certain that response `B'` (and the time associated with it) came *after* response `A'` (and its associated time). If the client sees a time in response `B'` that is before the time in `A'` then something is obviously wrong.

By querying several servers (assuming no collusion) a client can identify not only ordering issues but also time quality problems. One server reporting a time way off the others will be obvious and the collection of cross-signed request/responses will supply proof.

A client can store a log of request and response chains. There are no secret values in Roughtime requests or responses so this log can be provided to an audit service or other third-party. This [protobuf definition](https://roughtime.googlesource.com/roughtime/+/master/config.proto#40) from the Roughtime project is intended as the standard JSON format for such chain files.

# Closing Thoughts

Roughtime is elegant and simple but several of its clever features only become apparent after digging in. Hopefully this post helps spread the word. I didn't touch on other neat ideas in Roughtime like deliberate response [fault injection](https://roughtime.googlesource.com/roughtime/+/HEAD/ECOSYSTEM.md#Maintaining-a-healthy-software-ecosystem) or use of [key delegation](https://roughtime.googlesource.com/roughtime/+/HEAD/PROTOCOL.md#Certificates) to limit exposure of long-term keys.

If you're interested in learning more:

* The [project page](https://roughtime.googlesource.com/roughtime/) should be your first stop. The reference C++ and Go implementations live there.
* I wrote [Nearenough](https://github.com/int08h/nearenough), a Java implementation of Roughtime. It can pretty-print Roughtime messages if you need to do that sort of thing.
* Two good Hacker News discussions around the time of the initial project announcement: [one](https://news.ycombinator.com/item?id=12540941) and 
[two](https://news.ycombinator.com/item?id=12599705). 


