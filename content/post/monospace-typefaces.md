+++
title = "An Unreasonably Detailed Look at Monospace Typefaces"
date = "2017-03-21T11:34:37-05:00"
hidefromhome = "true"

+++

I wanted to display code on this site. It turned into this post.

Every programmer at some point chooses a typeface for coding. Perhaps you
are satisfied with your editor's defaults (I envy you). Maybe you are 
a typophile or perfectionist--good luck if so. Coding typefaces proliferate like 
programming languages. And no programming-related domain is complete without endless 
arguments over the details: monospace vs. proportional; ligatures vs not; bitmapped vs vector.

My desiderata in choosing a typeface to display code on this site:

* Monospaced
* Narrower width 
* Disambiguates similar glyphs such as `il1` and `oO0`
* Regular (book) and bold weights with an italic for each
* Flawless `woff` and `woff2` versions
* Reasonable licensing (if commercial)

Width is an especially important criteria. Narrower typefaces fit more code 
on screen which is especially beneficial on tablets and phones.

# Candidates

A number of commercial and free typefaces fit the bill (alphabetical order):

* [Anonymous Pro](http://www.marksimonson.com/fonts/view/anonymous-pro)
* [Decima Mono](https://www.myfonts.com/fonts/tipografiaramis/decima-mono/)
* [Envy Code R](https://damieng.com/blog/2008/05/26/envy-code-r-preview-7-coding-font-released)
* [Input](http://input.fontbureau.com/)
* [Iosevka](https://be5invis.github.io/Iosevka/)
* [Monoid](http://larsenwork.com/monoid/)
* [PragmataPro](https://www.fsd.it/shop/fonts/pragmatapro/)
* [Roboto Mono](https://fonts.google.com/specimen/Roboto+Mono)
* [Source Code Pro](http://adobe-fonts.github.io/source-code-pro/) (and derivative [Hasklig](https://github.com/i-tu/Hasklig))
* [Triplicate](http://practicaltypography.com/triplicate.html) 
* [Vera Sans Mono](https://www.gnome.org/fonts/) (and derivatives [DejaVu Sans Mono](https://dejavu-fonts.github.io/), [DejaVu Sans Code](https://github.com/SSNikolaevich/DejaVuSansCode), [Hack](http://sourcefoundry.org/hack/), [Menlo](https://en.wikipedia.org/wiki/Menlo_(typeface)))

Several otherwise great typefaces lacked one or more of my requirements and did not make the cut:

| Name | Disambiguated | Book+Bold | Italics |  woff{2} | Licensing |
|---|---|---|---|---|---|
| [Fira Mono](https://mozilla.github.io/Fira/) + derivatives | Yes | Yes | No |  Yes | Yes |
| [M+ Family](https://mplus-fonts.osdn.jp/) | Yes | Yes | No |  No[^2] | Yes |
| [Share Tech Mono](https://fonts.google.com/specimen/Share+Tech+Mono) | Yes | No | No |  Yes | Yes |
| [TheSans Mono](http://www.lucasfonts.com/fonts/thesansmono/) | Yes | Yes | Yes |  Yes | No[^3] |

For you Apple fans: **Monaco** lacks both italic and bold while **Menlo** is already represented as it's a **Vera Sans Mono** derivative.

[^2]: The `woff` version offered on the M+ site does not render all whitespace correctly. Taking the `ttf` versions and converting to `woff` via [FontSquirrel](https://www.fontsquirrel.com/tools/webfont-generator) (various "Expert" option permutations attempted) produces correct versions of 1M and 1MN but 2M still renders incorrectly. 

[^3]: LucasFonts [webfont](http://www.lucasfonts.com/webfonts/) licensing is per-style. Book, italic, bold, and bold italic each require a separate fee. The fee is both tiered and one-time, making it a great deal for a high-traffic site, but is a bit rich for a personal blog.

# How They Look

Below are the candidate typefaces at 18 points (except Monoid as noted below) rendered on Windows and MacOS[^4].

[^4]: Why images and not direct comparison via webfont? For some of the commercial fonts I purchased only a desktop license of the book/regular weight. Webfont licenses are separate. 

{{< figure src="/images/by_width_windows.png" link="/images/by_width_windows.png" caption="Typefaces by width on Windows 7; 18pt unless otherwise noted (click to enlarge)" >}}

{{< figure src="/images/by_width_mac.png" link="/images/by_width_mac.png" caption="Typefaces by width on MacOS; 18pt unless otherwise noted (click to enlarge)" >}}

Observations:

1. **Monoid** is big, *really big*. Its 18 point form so dwarfs the others I suspect an inadvertant error or typo in its definition.
2. From a metrics standpoint **Decima Mono Pro**, **DejaVu Sans Mono** (and other **Vera** derivatives), **Fira Mono**, **Hack**, **Input Mono Narrow**, and **Source Code Pro** are identical. They can serve as drop-in replacements (size-wise) for *Courier* and *Courier New*.
3. **Consolas** is much more compact than I realized. Those Windows users that have unconciously habituated to its compact form might find the above *Courier-esque* families too big.
4. **PragmataPro** and **Iosevka** take the cake for most compact monospace typeface.

# Typefaces of Note

Being a bit of a typophile I knew that making a typeface is a lot of work and this search 
reinforced my appreciation of the craft. Take a look at the thought Monoid's author put into [glyph design](https://medium.com/larsenwork-andreas-larsen/distinguishable-glyphs-in-coding-fonts-d74f5f0969ed#.o8u4qjh0m). Or how about PragmataPro's **>7,000** glyphs?!? Did you know Unicode has this many [arrows](https://github.com/fabrizioschiavi/arrow-finder)?

General observations and commentary on different typefaces you may encounter.

## Triplicate

[Triplicate](http://practicaltypography.com/triplicate.html) by Matthew Butterick. "Unlike the usual monospaced snoozefest"
according to the [specimen](http://typo.la/trts). Features a bevy of typographical features like 
real small caps, true italics, oldsyle figures, etc. Licensing is extremely generous and simple.

Disambiguated programming characters not enabled by default; accessing them requires use of OpenType features (stylistic set 2).
