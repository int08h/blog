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
reminders and email timestamps: assumptions of correct local time can be security-critical such as in 
certificate expiry and Kerberos tickets. 

The current dominant time-sync protocol, NTP (and SNTP), is 
unauthenticated (`pool.ntp.org` anyone?) and typical clients blindly trust any reply[^1]. 
Additionally there's no way to prove that a particular NTP server is a bad actor. Sure you could
capture packets or somesuch but that doesn't actually prove anything...packets are easy spoofed after all.

[^1]: Yes, there are [authentication mechanisms](https://tools.ietf.org/html/rfc5906) in NTPv4 as well as in-development NTP/PTP authentication protocols like [NTS](https://tools.ietf.org/html/draft-ietf-ntp-network-time-security-14). None of these have seen widespread deployment on the public internet, in mobile devices, or as out-of-the-box OS defaults.

# Roughtime Goals

Roughtime attempts to address these shortcomings. Briefly, its design goals are:

1. **Seconds accuracy** -- The protocol aims to get local clocks within a few seconds of the "true" time. The Roughtime creators point out that [25% of Chrome's certificate errors](https://roughtime.googlesource.com/roughtime#Roughstime-2) are caused by *incorrect local clocks*. A local clock within a few seconds of the server time is a sufficient remedy. 
2. **Secure** -- Roughtime servers sign every reply and that signature can be verified by any client. Clients can build a chain of replies to establish proof of server misbehavior. This proof is also verifiable by any client.
3. **Internet scalable** -- The protocol is stateless and built upon efficient and batchable operations. The network layer is constructed to prevent DDoS amplification. 
4. **Healthy ecosystem** -- Roughtime includes mechanisms for ensuring "freshness" of clients and correctness of their implementations. The intent is to surface bugs quickly and mitigate address hard-coding and other abusive client practices[^2].

[^2]: See https://en.wikipedia.org/wiki/NTP_server_misuse_and_abuse

Let's explore some of these points in detail.

# Scalable Public Key Signatures

Roughtime signs responses using the [Ed25519](https://ed25519.cr.yp.to/) public-key signature system. 
Ed25519 signatures are quite fast, [eBATS](https://bench.cr.yp.to/results-sign.html) results show that 
an Intel Skylake chip completes an Ed25519 signature in about ~49,000 cycles. Assuming a 3.0 GHz 
clock speed that's ~61,225 signatures per-second on a single core. 

Roughtime can sign batches of requests at once

# Anti-Amplification and Request/Response Rate Asymmetry 

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


