+++
title = "Monospace Typefaces: the odds are good, but the goods are odd"
date = "2017-03-21T11:34:37-05:00"
hidefromhome = "true"

+++

Every programmer will at some point need to choose a typeface for coding. If you have
perfectionist tendencies, good luck. Coding typefaces proliferate like programming
languages. And no programming-related domain is complete without endless arguments over the details:
monospace vs. proportional; ligatures vs not; bitmapped vs vector.

In choosing a typeface to display code on this site, my minimum requirements were:

* Monospaced
* Supports *Basic Latin* plus *General Punctuation* Unicode at minimum
* Disambiguated similar glyphs like `il17` and `oO0`
* Regular (book) and bold weights with an italic for each
* TrueType hints for on-screen display
* Flawless `woff` and `woff2` versions

Extensive unicode support and/or ligatures are not only unecessary, but undesireable as they
can bloat download size. 

# Candidates

A number of commercial and free typefaces fit the bill (alphabetical order):

* [Decima Mono](https://www.myfonts.com/fonts/tipografiaramis/decima-mono/)
* [Iosevka](https://be5invis.github.io/Iosevka/)
* [Monoid](http://larsenwork.com/monoid/)
* [PragmataPro](https://www.fsd.it/shop/fonts/pragmatapro/)
* [Roboto Mono](https://fonts.google.com/specimen/Roboto+Mono)
* [Source Code Pro](http://adobe-fonts.github.io/source-code-pro/) (and derivative [Hasklig](https://github.com/i-tu/Hasklig))
* [Triplicate](http://practicaltypography.com/triplicate.html) 
* [Vera Sans Mono](https://www.gnome.org/fonts/) (and derivatives [DejaVu Sans Mono](https://dejavu-fonts.github.io/), [DejaVu Sans Code](https://github.com/SSNikolaevich/DejaVuSansCode), [Hack](http://sourcefoundry.org/hack/))

Being a bit of a typophile I knew that making a typeface is a lot of work and this search 
reinforced my appreciation of the craft. Take a look at the thought Monoid's author put into [glyph design](https://medium.com/larsenwork-andreas-larsen/distinguishable-glyphs-in-coding-fonts-d74f5f0969ed#.o8u4qjh0m). Or how about PragmataPro's **>7,000** glyphs?!? Did you know Unicode has this many [arrows](https://github.com/fabrizioschiavi/arrow-finder)?


I encountered several great typefaces that lacked one or more of the requirements and 
therefore did not make the cut:

| Name | Disambiguated | Book+Bold | Italics | Hinted | woff{2} | 
|---|---|---|---|---|---|---|
| Fira Mono + derivatives | Yes | Yes | No | Yes | Yes |
| M+ Family | Yes | Yes | No | Yes | No[^2] |
| Share Tech Mono | Yes | No | No | Yes | Yes | 
| TheSans Mono | No[^3] | Yes | Yes | Yes | Yes |

[^2]: The `woff` version offered on the M+ site does not render all whitespace correctly. Taking the `ttf` versions and converting to `woff` via FontSquirrel produces correct versions of 1M and 1MN but 2M still renders incorrectly. 

[^3]: Zero is not slashed nor dotted. It may be accessible via OpenType alternate form, but I didn't see it.

# Typefaces of Note

General observations and commentary on different typefaces you may encounter.

## M+ Family

M+ by [Coji Morishita](https://twitter.com/coz). *Very* condensed, looks particularly good on high-resolution "retina" displys
but struggles a bit on lower-DPI monitors. 

Multitude of styles drawing inspiration from different 
Japanese Kana script styles. The 'M' styles (1M, 2M, 1MN) are the relevant monospaced
forms. I found 1MN particularly intruiging and used it for the `int08h` logo in the header. 
[Others](http://www.macwright.org/2014/07/09/mplus.html) have also noted M+'s qualities.
 
## Triplicate

[Triplicate](http://practicaltypography.com/triplicate.html) by Matthew Butterick. "Unlike the usual monospaced snoozefest"
according to the [specimen](http://typo.la/trts). Features a bevy of typographical features like 
real small caps, true italics, oldsyle figures, etc. Licensing is extremely generous and simple.

Disambiguated programming characters not enabled by default; accessing them requires use of OpenType features (stylistic set 2).
