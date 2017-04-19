+++
date = "2017-04-13T14:06:58-05:00"
title = "Scalable, Lie-Detecting Timeserving with Roughtime"
hidefromhome = "true"

+++

[Roughtime](https://roughtime.googlesource.com/roughtime/) is a new(-ish) protocol 
designed to provide internet-scale secure time synchronization. Designed by Googlers, it
aims to address shortcomings of how NTP is typically used in the wild.

# Why This Is Important

Many aspects of day-to-day computing assume an accurate local clock. Not just your appointment
reminders and iMessage timestamps: assumptions of correct local time can be security-critical such as in 
certificate expiry and Kerberos tickets. 

The dominant internet time-sync protocol, Network Time Protocol (NTP), is showing its age:

1. It is unauthenticated[^1] and most clients blindly trust any reply (`pool.ntp.org` anyone?). 
2. Should a client find that an NTP server is responding with bad values there's no way to prove to others that a particular server is a bad actor[^2].
3. NTP has some (mis)features that can be abused to create DDoS traffic.

[^1]: Yes, there are [authentication mechanisms](https://tools.ietf.org/html/rfc5906) in NTPv4 as well as in-development NTP/PTP authentication protocols like [NTS](https://tools.ietf.org/html/draft-ietf-ntp-network-time-security-14). None of these have seen widespread deployment on the public internet, in mobile devices, or as out-of-the-box OS defaults. Furthermore these solutions require that clients unconditionally trust one or move servers. 

[^2]: Sure you could capture packets or somesuch but that doesn't actually prove anything...packets are easy spoofed after all.

# Roughtime Goals

Roughtime attempts to address these shortcomings. Paraphrasing its design goals:

1. **Seconds accuracy** -- The protocol aims to get local clocks within a few seconds of the "true" time. The Roughtime creators point out that [25% of Chrome's certificate errors](https://roughtime.googlesource.com/roughtime#Roughstime-2) are caused by *incorrect local clocks*. A local clock within a few seconds of GPS or some other reference time is a sufficient remedy. The designers make clear that NTP and PTP are preferred if more precision is required.
2. **Secure** -- Roughtime servers sign every reply and the signature is verifiable by any client. Clients can build a chain of replies to establish proof of server misbehavior. This proof is also verifiable by any client. No central trust is required.
3. **Internet scalable** -- The protocol is stateless and built upon efficient and batchable operations. The network layer is constructed to prevent DDoS amplification. 
4. **Healthy ecosystem** -- Roughtime includes mechanisms for ensuring "freshness" of clients and correctness of their implementations. The intent is to surface bugs quickly and mitigate abusive client practices[^3].

[^3]: See [NTP Server Misuse and Abuse](https://en.wikipedia.org/wiki/NTP_server_misuse_and_abuse)

Bellow I'll explore some of the interesting aspects of Roughtime that may not be immediately obvious. 

# Scalable Public Key Signatures

Roughtime signs responses using the [Ed25519](https://ed25519.cr.yp.to/) public-key signature system. 
Ed25519 signatures are quite fast: according to [eBATS](https://bench.cr.yp.to/results-sign.html) 
an Intel Skylake CPU takes about ~49,000 cycles to compute an Ed25519 signature. Assuming a 3.0 GHz 
clock speed that's ~61,000 signatures per second on a single core. Signatures are also compact: 64 bytes each.

To scale the signing workload, Roughtime uses a [Merkle Tree](https://en.wikipedia.org/wiki/Merkle_tree) 
to sign a **batch** of client requests with a single signature operation (the root of the tree is signed).
With a batch size of 64 a single 3.0 GHz Skylake core can sign **3.9 million** requests per second. 

What about the processing and overhead needed to create responses? 
The bulk of the response processing can be reused for all replies in a single batch[^4]. Ergo "sign once, reply many times".

[^4]: Details: only the `PATH` and `INDX` tags vary client-to-client. The `SIG`, `SREP`, and `CERT` tags will be identical in all responses. 

# Anti-Amplification and Request/Response Size Asymmetry 

All Roughtime requests are required to be at least 1024 bytes long and any request shorter is simply dropped. Why? To ensure that a Roughtime server cannot be used as 
a [DDoS amplifier](https://www.incapsula.com/ddos/attack-glossary/ntp-amplification.html). 

| Message    | Size (bytes) |
| ----- | -----:|
| Roughtime **Request**  | 1024 |
| Roughtime **Response**, single request | 360 |
| Roughtime **Response**, batch of 64 requests | 744 |

The 1 KiB request size ensures an attacker has a negative return on any traffic sent to a Roughtime server.
It also naturally rate-limits requests: at 1 KiB per request a 10 Gbps link can deliver a maximum of 1.2 million
requests per second. As shown above, that rate is easily handled by a single Skylake core.

# Signature Size

You probably noticed that the response for 64 requests is suspiciously small: 744 bytes. This is not 
a typo, recall that Roughtime uses a Merkle Tree to batch requests. When replying to clients, Roughtime
sends only the tree nodes each client needs to verify its request is included in the tree[^5]. Roughtime does 
not send the whole tree.

[^5]: This elegant idea shows up in [many](https://www.certificate-transparency.org/log-proofs-work) [other](https://blog.ethereum.org/2015/11/15/merkling-in-ethereum/) [places](https://petertodd.org/2016/opentimestamps-announcement).

An example might help. We'll work with this Merkle Tree that batches 7 requests, `A` though `G`:

```text
              _____ h(ABCDEFG) ______            
             /                       \
         h(ABCD)                   h(EFG)
       /         \                /      \
   h(AB)         h(CD)         h(EF)      \
   /   \         /   \         /   \       \
 h(A)  h(B)    h(C)  h(D)    h(E)  h(F)   h(G)

h(...) = SHA512
```

Imagine constructing a response to client C:

1. To compute `h(CD)` the client needs `D` (client C remembers what it sent).
2. To compute `h(ABCD)` the client already has `h(CD)` from step #1, so it needs `h(AB)`.
3. For `h(ABCDEFG)` the client has `h(ABCD)` from step #2 and now needs `h(EFG)`.

So the path in the reply to client C is `h(D)`, `h(AB)`, and `h(EFG)`.

Being a complete tree, the number of path elements required by each client is bounded at `ceil(log2(batch size))`. 
In the case of a 64 request batch `log2(64) == 6` and `6 * 64 bytes per sha512 == 384 bytes`. This is how
we arrive at 744 bytes for a 64-element batch response: 360 bytes response + 384 bytes tree path data =
744 bytes total.


# Blah Blah

If batches of 64 requests are allowed then a Skylake chip can sign 4.3 million requests per core-second.

At that rate, the CPU time for public-key signatures becomes insignificant compared to the work needed to 
handle that number of packets. Since we require that requests be padded to 1KB to avoid becoming a DDoS 
amplifier, a 10Gbps network link could only deliver 1.2 million requests per second anyway.

Responses smaller than requests. log2(tree size) means even a batch of 64 signatures the tree
path sent to client is log2(depth 64) == depth 6 and 6 * 64 bytes == 384 bytes of path data.

Response size ex-path data : 360 bytes
   64 depth tree path data : 384 bytes
            Response Total : 744 bytes


# Amortizng Signature Costs
Sending request to roughtime.sandbox.google.com/173.194.202.158:2002
Read message of 488 bytes from roughtime.sandbox.google.com/173.194.202.158:2002:
RtMessage|5|{
  SIG(64) = 91c2018ee3bf3f6fae2f749dda25e54e993add44f37a18c98772532a7da7d0d8169d6f48e321cb847e3b8bab4744bd3a3d6590ca4eb702889b72d09ab9bd6e03
  PATH(128) = f387b87e30d84b85bf28988ca1573b153f2e7066c23a38b728490c9d509796328290b8ff2048ba32f7336ff097159700553e9dd7917e1e75d85527ed6aa1872febaf26afd2bf2c158a507e863a2e081bc43dceb31e8d1e0374bb4afd6689ece4e0a2aad2fe468f9e59b55e7e4075c33c2a44ea23f2e43ff928563482745cd941
  SREP(100) = RtMessage|3|{
    RADI(4) = 40420f00
    MIDP(8) = 35c4bd72114d0500
    ROOT(64) = 862d184f6992e07712fc3b72062e47861fe660378677627354888c0f3c5eb521503e748990e96f9bff9cab14bb3bfae611b612a4a66d50215bbf172c5536cbaf
  }
  CERT(152) = RtMessage|2|{
    SIG(64) = 615533dc03a483eebe78db8c1161d5fa31bdb424c78c681bf64e4b9e679ca924f2c6211c435f2500d62934869c75acdca4b4ff3bd359f32d3eb419cf8cf0f809
    DELE(72) = RtMessage|3|{
      PUBK(32) = 7cc511f1d9b2c7004297c5fdc7df947b73b1459ee2689bac5df1e70ff44bb3a1
      MINT(8) = 00a068400f4d0500
      MAXT(8) = 00809dd5734d0500
    }
  }
  INDX(4) = 00000000
}

midpoint    : 2017-04-13T19:36:58.375Z (radius 1 sec)
local clock : 2017-04-13T19:37:04.325Z

Read message of 488 bytes from roughtime.sandbox.google.com/173.194.202.158:2002:
RtMessage|5|{
  SIG(64) = 91c2018ee3bf3f6fae2f749dda25e54e993add44f37a18c98772532a7da7d0d8169d6f48e321cb847e3b8bab4744bd3a3d6590ca4eb702889b72d09ab9bd6e03
  PATH(128) = 732ae00a62cc3de2d9656fdb9fcbcd8c9439b269b2074332c9a8d1c6f1da36d85f7361328aaa503ef76054a8fbc7e9a168ed4719567cbd0c25f5f7b028b49428ebaf26afd2bf2c158a507e863a2e081bc43dceb31e8d1e0374bb4afd6689ece4e0a2aad2fe468f9e59b55e7e4075c33c2a44ea23f2e43ff928563482745cd941
  SREP(100) = RtMessage|3|{
    RADI(4) = 40420f00
    MIDP(8) = 35c4bd72114d0500
    ROOT(64) = 862d184f6992e07712fc3b72062e47861fe660378677627354888c0f3c5eb521503e748990e96f9bff9cab14bb3bfae611b612a4a66d50215bbf172c5536cbaf
  }
  CERT(152) = RtMessage|2|{
    SIG(64) = 615533dc03a483eebe78db8c1161d5fa31bdb424c78c681bf64e4b9e679ca924f2c6211c435f2500d62934869c75acdca4b4ff3bd359f32d3eb419cf8cf0f809
    DELE(72) = RtMessage|3|{
      PUBK(32) = 7cc511f1d9b2c7004297c5fdc7df947b73b1459ee2689bac5df1e70ff44bb3a1
      MINT(8) = 00a068400f4d0500
      MAXT(8) = 00809dd5734d0500
    }
  }
  INDX(4) = 01000000
}

Read message of 488 bytes from roughtime.sandbox.google.com/173.194.202.158:2002:
RtMessage|5|{
  SIG(64) = 91c2018ee3bf3f6fae2f749dda25e54e993add44f37a18c98772532a7da7d0d8169d6f48e321cb847e3b8bab4744bd3a3d6590ca4eb702889b72d09ab9bd6e03
  PATH(128) = 00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000ec2e49c19cd840e953c9351d5b191943d4ae6b51f95d7713f1108f178747ecdcd31fd5433670008805bfac53885e5325ee35e5df4d7ae58d5f871e3b1fb42f2f
  SREP(100) = RtMessage|3|{
    RADI(4) = 40420f00
    MIDP(8) = 35c4bd72114d0500
    ROOT(64) = 862d184f6992e07712fc3b72062e47861fe660378677627354888c0f3c5eb521503e748990e96f9bff9cab14bb3bfae611b612a4a66d50215bbf172c5536cbaf
  }
  CERT(152) = RtMessage|2|{
    SIG(64) = 615533dc03a483eebe78db8c1161d5fa31bdb424c78c681bf64e4b9e679ca924f2c6211c435f2500d62934869c75acdca4b4ff3bd359f32d3eb419cf8cf0f809
    DELE(72) = RtMessage|3|{
      PUBK(32) = 7cc511f1d9b2c7004297c5fdc7df947b73b1459ee2689bac5df1e70ff44bb3a1
      MINT(8) = 00a068400f4d0500
      MAXT(8) = 00809dd5734d0500
    }
  }
  INDX(4) = 02000000
}

Client 0 : 732ae00a62cc3de2d9656fdb9fcbcd8c9439b269b2074332c9a8d1c6f1da36d85f7361328aaa503ef76054a8fbc7e9a168ed4719567cbd0c25f5f7b028b49428
Client 2 : f387b87e30d84b85bf28988ca1573b153f2e7066c23a38b728490c9d509796328290b8ff2048ba32f7336ff097159700553e9dd7917e1e75d85527ed6aa1872f
Client 3 : 6a3cde162db863302629011f6aba5ed01a5d4f38537d3708db2b5f6ba13ac3a0353a4cfde06a0d0f7949e29db50016bf8b55428676bdd2d7ae56a4609f8e3407

xxx Client 1 : 2eb2bc5d856a7e0608e40b0e5562cb1a47ae3b77f2a5999703c9f2197b9ee1c6fabaf7726b022b1c1266913cec321fadbf58c49ce67118e2fcf6c6f8ae28d513

RtMessage|5|{
  SIG(64) = 936e14812e0297a9fb388b411428ee3485ac8d61ba815e01703d146918420f8c2b31f3ef73f930f03e655511b228d92c9499ac9f9f058da4d99807a097f57b03
  PATH(0) = 
  SREP(100) = RtMessage|3|{
    RADI(4) = 40420f00
    MIDP(8) = f3c3bd72114d0500
    ROOT(64) = 2eb2bc5d856a7e0608e40b0e5562cb1a47ae3b77f2a5999703c9f2197b9ee1c6fabaf7726b022b1c1266913cec321fadbf58c49ce67118e2fcf6c6f8ae28d513
  }
  CERT(152) = RtMessage|2|{
    SIG(64) = 615533dc03a483eebe78db8c1161d5fa31bdb424c78c681bf64e4b9e679ca924f2c6211c435f2500d62934869c75acdca4b4ff3bd359f32d3eb419cf8cf0f809
    DELE(72) = RtMessage|3|{
      PUBK(32) = 7cc511f1d9b2c7004297c5fdc7df947b73b1459ee2689bac5df1e70ff44bb3a1
      MINT(8) = 00a068400f4d0500
      MAXT(8) = 00809dd5734d0500
    }
  }
  INDX(4) = 00000000
}

# Client Request Chains as Evidence

## Why Not Server Chains Too?

# Additional Info

Hackernews discussions [one](https://news.ycombinator.com/item?id=12540941) and 
[two](https://news.ycombinator.com/item?id=12599705). 


