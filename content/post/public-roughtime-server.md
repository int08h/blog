+++
date = "2018-02-17T19:01:26-05:00"
title = "roughtime.int08h.com"
description = "About the roughenough-based Roughtime server at roughenough.int08h.com:2002"

+++

# The World Needs More Roughtime

It does! So I bring to you a public Roughtime server:

```text
roughtime.int08h.com port 2002
```

The server runs [roughenough](https://github.com/int08h/roughenough) (no surprise) on a Google Compute Engine
instance in us-central1. 

Time is sourced from Google's [public NTP servers](https://developers.google.com/time/) 
which conveniently implement the [24-hour leap second smear](https://developers.google.com/time/smear) required to 
provide "Roughtime UTC" as specificed in the 
[protocol definition](https://roughtime.googlesource.com/roughtime/+/HEAD/PROTOCOL.md#the-signed-response-and-roughtime-utc).

# Public Key 

```
016e6e0284d24c37c6e4d7d8d5b4e1d3c1949ceaa545bf875616c9dce0c9bec1
```

That's the server's long-term public key which is also stuffed into a DNS `TXT` record should 
you need it, along with a superfluous `SRV` record just for fun:

```bash
$ dig -t txt roughtime.int08h.com
...
;; ANSWER SECTION:
roughtime.int08h.com.  1799  IN  TXT "016e6e0284d24c37c6e4d7d8d5b4e1d3c1949ceaa545bf875616c9dce0c9bec1"

$ dig -t srv _roughtime._udp.int08h.com
...
;; ANSWER SECTION:
_roughtime._udp.int08h.com.  1799 IN SRV 100 1 2002 roughtime.int08h.com.
```




