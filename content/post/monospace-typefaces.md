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
* [Vera Sans Mono](https://www.gnome.org/fonts/) (and derivatives [DejaVu Sans Mono](https://dejavu-fonts.github.io/), [DejaVu Sans Code](https://github.com/SSNikolaevich/DejaVuSansCode), [Hack](http://sourcefoundry.org/hack/))

Several otherwise great typefaces lacked one or more of my requirements and did not make the cut:

| Name | Disambiguated | Book+Bold | Italics |  woff{2} | Licensing |
|---|---|---|---|---|---|
| [Fira Mono](https://mozilla.github.io/Fira/) + derivatives | Yes | Yes | No |  Yes | Yes |
| [M+ Family](https://mplus-fonts.osdn.jp/) | Yes | Yes | No |  No[^2] | Yes |
| [Share Tech Mono](https://fonts.google.com/specimen/Share+Tech+Mono) | Yes | No | No |  Yes | Yes |
| [TheSans Mono](http://www.lucasfonts.com/fonts/thesansmono/) | Yes | Yes | Yes |  Yes | No[^3] |

[^2]: The `woff` version offered on the M+ site does not render all whitespace correctly. Taking the `ttf` versions and converting to `woff` via FontSquirrel ("Fix missing ..." features enabled) produces correct versions of 1M and 1MN but 2M still renders incorrectly. 

[^3]: Webfont licensing is per-style so each of book, italic, bold, and bold italic requires a separate fee. Noteworthy that the fee is one-time, making it a great deal for a high-traffic site, but a bit rich for a personal blog.

# Typefaces of Note

Being a bit of a typophile I knew that making a typeface is a lot of work and this search 
reinforced my appreciation of the craft. Take a look at the thought Monoid's author put into [glyph design](https://medium.com/larsenwork-andreas-larsen/distinguishable-glyphs-in-coding-fonts-d74f5f0969ed#.o8u4qjh0m). Or how about PragmataPro's **>7,000** glyphs?!? Did you know Unicode has this many [arrows](https://github.com/fabrizioschiavi/arrow-finder)?

General observations and commentary on different typefaces you may encounter.

## Triplicate

[Triplicate](http://practicaltypography.com/triplicate.html) by Matthew Butterick. "Unlike the usual monospaced snoozefest"
according to the [specimen](http://typo.la/trts). Features a bevy of typographical features like 
real small caps, true italics, oldsyle figures, etc. Licensing is extremely generous and simple.

Disambiguated programming characters not enabled by default; accessing them requires use of OpenType features (stylistic set 2).
