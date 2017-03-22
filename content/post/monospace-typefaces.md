+++
title = "An Unreasonably Detailed Look at Monospace Typefaces"
date = "2017-03-21T11:34:37-05:00"
hidefromhome = "true"

+++

I needed to choose a typeface to display code on this site. That task morphed into this post.

# Pick Your Poison

Every programmer at some point chooses a typeface for coding. Perhaps you
are satisfied with your editor's defaults (I envy you). Maybe you are 
a typophile or perfectionist. Good luck if so: coding typefaces proliferate like 
programming languages. And no programming-related domain is complete without endless 
arguments over implementation details: monospace vs. proportional; ligatures vs. not; bitmapped vs. vector.

My desiderata in choosing a typeface to display code on this site:

* Monospaced
* Narrower width than typical
* Disambiguates similar glyphs such as `il1` and `oO0`
* Regular (book) and bold weights with an italic for each
* Flawless `woff` and `woff2` versions
* Reasonable (to me) licensing if commercial

Width is an especially important criteria. Narrower typefaces fit more code 
on screen which is especially beneficial on tablets and phones.

# Candidates

Filtering on the above criteria, a number of commercial and free typefaces fit the bill (alphabetical order):

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
| [Fira Mono](https://mozilla.github.io/Fira/) + derivatives | Yes | Yes | **No** |  Yes | Yes |
| [M+ Family](https://mplus-fonts.osdn.jp/) | Yes | Yes | **No** |  **No**[^1] | Yes |
| [Share Tech Mono](https://fonts.google.com/specimen/Share+Tech+Mono) | Yes | **No** | **No** |  Yes | Yes |
| [TheSans Mono](http://www.lucasfonts.com/fonts/thesansmono/) | Yes | Yes | Yes |  Yes | **No**[^2] |

For you Apple fans: **Monaco** lacks both italic and bold while **Menlo** is already represented since it's a **Vera Sans Mono** derivative.

**Triplicate**, a font "[u]nlike the usual monospaced snoozefest"[^3], didn't make the cut for 
my own abitrary aesthetic reasons. Sorry Matthew.

[^1]: The `woff` version offered on the M+ site does not render all whitespace correctly. Taking the `ttf` versions and converting to `woff` via [FontSquirrel](https://www.fontsquirrel.com/tools/webfont-generator) (various "Expert" option permutations attempted) produces correct versions of 1M and 1MN but 2M still renders incorrectly. 

[^2]: LucasFonts [webfont](http://www.lucasfonts.com/webfonts/) licensing is per-style: book, italic, bold, and bold italic each require a separate fee. The fee is both tiered and one-time, making it a great deal for a high-traffic site, but is a bit rich for a personal blog.

[^3]: From the [specimen](http://typo.la/trts)

# How They Look

Below are the candidate typefaces at 18 points (except Monoid as noted below) rendered[^4] on Windows and MacOS.

[^4]: Why images and not direct comparison via webfont? For some of the commercial fonts I purchased only a desktop license of the book/regular weight. Webfont licenses are separate and vary in their terms and costs.

{{< figure src="/images/by_width_windows.png" link="/images/by_width_windows.png" caption="Typefaces by width on Windows 7; 18pt unless otherwise noted (click to enlarge)" >}}

{{< figure src="/images/by_width_mac.png" link="/images/by_width_mac.png" caption="Typefaces by width on MacOS; 18pt unless otherwise noted (click to enlarge)" >}}

Observations:

1. **Monoid** is big, *really big*. Its 18 point form so dwarfs the others I suspect an inadvertant error or typo in its definition.
2. A cluster of *Courer-sized* cousins: **Decima Mono Pro**, **DejaVu Sans Mono** (and other **Vera** derivatives), **Fira Mono**, **Hack**, **Input Mono Narrow**, and **Source Code Pro** have identical metrics. They are drop-in replacements (size-wise) for *Courier* and *Courier New*.
3. **Consolas** is much more compact than I realized. Windows users that have unconciously habituated to its compact form may find the above *Courier-sized* faces too wide.
4. **PragmataPro** and **Iosevka** take the cake for most compact monospace typeface.
5. For the observant: **Decima Mono Pro** does have a slashed zero. It's available as an OpenType alternate and I forgot to enable it for the screen-shots.

Given I want a face that's more compact the Courier-sized gang, my candiates are narrowed (ha!) to Input Mono Condensed and everything below it.

# Closing Thoughs

*Holy cow it's a lot of work to make a good typeface*.

Being a bit of a typophile I knew that making a typeface was 
a heady effort. But my goodness! This search for a code font has completely reset my appreciation of the craft! Take a 
look at the thought Monoid's author put 
into [glyph design](https://medium.com/larsenwork-andreas-larsen/distinguishable-glyphs-in-coding-fonts-d74f5f0969ed#.o8u4qjh0m).
Or how about PragmataPro's **>7,000** glyphs?!? 
Did you know Unicode has this many [arrows](https://github.com/fabrizioschiavi/arrow-finder)?

## Triplicate


Disambiguated programming characters not enabled by default; accessing them requires use of OpenType features (stylistic set 2).
