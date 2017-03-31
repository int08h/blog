+++
date = "2017-03-20"
title = "About int08h"
description = ""
hidefromhome = "true"
+++

## Contact info

* Email `stuart {at} int08h.com`
* Twitter `@int08h`

PGP key at [keybase.io/sstock](https://keybase.io/sstock) if required.


## License and Disclaimer

All content on `int08h` is copyright (c) 2017 int08h LLC and licensed under the Creative Commons 
[Attribution-ShareAlike 4.0 License](https://creativecommons.org/licenses/by-sa/4.0/legalcode). 

Stuart Stock writes on `int08h` in his personal capacity and is solely responsible for its content. 
The opinions expressed and statements made do not reflect the opinions, policies, or positions of any 
employers past or present. 

## WTF is an `int08h`?

It's the old-school [PC BIOS system timer interrupt](http://www.delorie.com/djgpp/doc/rbinter/id/48/0.html). 
`int` is short for "interrupt" and `08h` notates hex. Today one might write `int0x08` instead.
An [Intel 852x](http://wiki.osdev.org/Programmable_Interval_Timer) programmable interval timer generated `int08h` 
~18.2 times per second (every ~55 milliseconds). 

"Well behaved" programs did not mess with `int08h`. Of course, not all programs (and programmers) were 
well behaved, thus those tempting fate by patching `int08h` could be *quite* interesting.

## Okay, why `08` then?

Because the `.com` domain of my favorite PC interrupt, `int13h`, was already taken. Sadface.

## Colophon

Generated using [Hugo](https://gohugo.io/). Theme derived from [Hugo-Zen](https://github.com/rakuishi/hugo-zen).
Body typeface is [Merriweather](https://fonts.google.com/specimen/Merriweather).
Titles and headings use [Concourse](http://concoursefont.com/).
Monospace is [Input Mono Compressed](http://input.fontbureau.com/) in various weights.
Syntax highlighting by [`highlight.js`](https://highlightjs.org/).
