+++
date = "2017-03-01"
title = "About int08h"
description = ""
hidefromhome = "true"
+++

## Contact info

* Email `stuart {at} int08h.com`
* Twitter `@int08h`

PGP key can be obtained from [keybase.io/sstock](https://keybase.io/sstock) if required.


## License and Disclaimer

All content on `int08h` is Copyright (c) 2017 int08h LLC and licensed under the Creative Commons 
[Attribution-ShareAlike 4.0 License](https://creativecommons.org/licenses/by-sa/4.0/legalcode). 


Stuart Stock writes on `int08h` in his personal capacity and is solely responsible for its content. 
The opinions expressed and statements made do not reflect the opinions, policies, or positions of any 
employers past or present. 

## WTF is an `int08h`?

Nerd history. It's the old-school [PC BIOS system timer interrupt](http://www.delorie.com/djgpp/doc/rbinter/id/48/0.html). 
An [Intel 8524](http://wiki.osdev.org/Programmable_Interval_Timer) programmable interval timer generated it ~18.2 times 
per second (every 55 milliseconds). The `int08h` handler was responsible for calling `int1Ch` which was the preferred 
periodic interrupt for "well behaved" programs. 

Of course, not all programs were well behaved. Those that re-vectored `int08h` (or re-vectored any of the lower interrupts) 
were typically quite *interesting*.


## Okay, why `int08h` then?

Because the `.com` domain of my favorite PC interrupt, `int13h`, was already taken. Sadface.

## Colophon

Generated using [Hugo](https://gohugo.io/). Theme derived from [Hugo-Zen](https://github.com/rakuishi/hugo-zen).
Body typeface is [Merriweather](https://fonts.google.com/specimen/Merriweather) and headings are 
[Merriweather Sans](https://fonts.google.com/specimen/Merriweather+Sans) with slightly tightened tracking. Monospace
typeface is your system default with syntax highlighting by [`highlight.js`](https://highlightjs.org/).
