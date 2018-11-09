# `Intl.Segmenter`: Unicode segmentation in JavaScript

Stage 3 proposal, champion Daniel Ehrenberg (Igalia)

## Motivation

A code point is not a "letter" or a displayed unit on the screen. That designation goes to the grapheme, which can consist of multiple code points (e.g., including accent marks, conjoining Korean characters). Unicode defines a grapheme segmentation algorithm to find the boundaries between graphemes. This may be useful in implementing advanced editors/input methods, or other forms of text processing.

Unicode also defines an algorithm for finding breaks between words and sentences, which CLDR tailors per locale. These boundaries may be useful, for example, in implementing a text editor which has commands for jumping or highlighting words and sentences. There is an analogous algorithm for opportunities for line breaking.

Grapheme, word and sentence segmentation is defined in [UAX 29](http://unicode.org/reports/tr29/). Line breaking is defined in [UAX 14](http://www.unicode.org/reports/tr14/). Web browsers need an implementation of both kinds of segmentation to function, and shipping it to JavaScript saves memory and network bandwidth as compared to expecting developers to implement it themselves in JavaScript.

Chrome has been shipping its own nonstandard segmentation API called `Intl.v8BreakIterator` for a few years. However, [for a few reasons](https://github.com/tc39/ecma402/issues/60#issuecomment-194041835), this API does not seem suitable for standardization. This explainer outlines a new API which attempts to be more in accordance with modern, post-ES2015 JavaScript API design.

## Example

```js
// Create a segmenter in your locale
let segmenter = new Intl.Segmenter("fr", {granularity: "word"});

// Get an iterator over a string
let iterator = segmenter.segment("Ceci n'est pas une pipe");

// Iterate over it!
for (let {segment, breakType} of iterator) {
  console.log(`segment: ${segment} breakType: ${breakType}`);
  break;
}

// logs the following to the console:
// segment: Ceci breakType: letter
```

## API

[polyfill](https://gist.github.com/inexorabletash/8c4d869a584bcaa18514729332300356) for a historical snapshot of this proposal

### `new Intl.Segmenter(locale, options)`

Interpretation of options:
- `granularity`, which may be `grapheme`, `word`, `sentence` or `line`.
- `strictness`, valid only for `line` granularity, which may be `'strict'`, `'normal'`, or `'loose'`, following <a href="https://drafts.csswg.org/css-text-3/#line-break-property">CSS Text Module Level 3</a>.

### `Intl.Segmenter.prototype.segment(string)`

This method creates a new `%SegmentIterator%` over the input string, which will lazily find breaks, starting at index 0.

### `%SegmentIterator%`

This class iterates over segment boundaries of a particular string.

#### Iteration result data

* `index` is the code point index immediately following the last found boundary.
* `segment` is the substring between the last found boundary and its preceding boundary.
* `breakType` is a value characterizing `segment` and its terminating boundary. Some particularly important values are:
  * For `word` granularity, "word" for letter/number/ideograph segments vs. "none" for spaces/punctuation/etc.
  * For `line` granularity, "hard" for mandatory line breaks such as follow U+000A LINE FEED (`\n`) vs. "soft" for line break opportunities such as follow U+0020 SPACE.

### Methods on %SegmentIterator%:

#### `%SegmentIterator%.prototype.next()`

The `next` method implements the _Iterator_ interface, finding the next boundary and returning an `IteratorResult` object relating to it. The object includes `index`, `segment`, and `breakType` fields corresponding to iteration result data.

#### `%SegmentIterator%.prototype.following(index)`

Move the iterator to the next break position after the given code unit index _index_, or if no index is provided, after its current index. Returns *true* if the end of the string was reached.

#### `%SegmentIterator%.prototype.preceding(index)`

Move the iterator to the previous break position before the given code unit index _index_, or if no index is provided, before its current index. Returns *true* if the beginning of the string was reached.

#### `get %SegmentIterator%.prototype.index`

Return the code unit index of the most recently discovered break position, as an offset from the beginning of the string. Initially the `index` is 0.

#### `get %SegmentIterator%.prototype.breakType`

The `breakType` of the most recently discovered segment. If there is no current segment (e.g., a just-instantiated SegmentIterator, or one which has reached the end), or if the break type is "grapheme", then this will be `undefined`.

## FAQ

Q: Why should we pass a locale and options bag for grapheme breaks? Isn't there just one way to do it?

A: The situation is a little more complicated, e.g., for Indic scripts. Work is ongoing to support grapheme break options for these scripts better; see [this bug](http://unicode.org/cldr/trac/ticket/2142), and in particular [this CLDR wiki page](http://cldr.unicode.org/development/development-process/design-proposals/grapheme-usage). Seems like CLDR/ICU don't support this yet, but it's planned.

Q: Shouldn't we be putting new APIs in built-in modules?

A: If built-in modules had come out before this gets to Stage 3, that sounds like a good option. However, so far the idea in TC39 has been not to block either thing on the other. Built-in modules still have some big questions to resolve, e.g., how/whether polyfills should interact with them.

Q: Why is hyphenation not included?

A: Hyphenation is expected to have a different sort of API shape for various reasons:
- Adding a hyphenation break may change the spelling of the affected text
- There may be hyphenation breaks of different priorities
- Hyphenation plays into line layout and font rendering in a more complex way, and we might want to expose it at that level (e.g., in the Web Platform rather than ECMAScript)
- Hyphenation is just a less well-developed thing in the internationalization world. CLDR and ICU don't support it yet; certain web browsers are only getting support for it now in CSS. It's often not done perfectly. It could use some more time to bake. By contrast, word, grapheme, sentence and line breaks have been in the Unicode specification for a long time; this is a shovel-ready project.

Q: Why is this API stateful?

It would be possible to make a stateless API without a SegmentIterator, where instead, a Segmenter has two methods, with two arguments: a string and an offset, for finding the next break before or after. This method would return an object `{breakType, index}` similar to what `next()` returns in this API. However, there are a few downsides to this approach:
- Performance:
  - Often, JavaScript implementations need to take an extra step to convert an input string into a form that's usable for the external internationalization library. When querying several break positions on a single string, it is nice to reuse the new form of the string; it would be difficult to cache this and invalidate the cache when appropriate.
  - The `{breakType, index}` object may be a difficult allocation to optimize away. Some usages of this library are performance-sensitive and may benefit from a lighter-weight API which avoids the allocation.
- Convenience: Many (most?) usages of this API want to iterate through a string, either forwards or backwards, and get all of the appropriate breaks, possibly interspersed with doing related work. A stateful API may be more terse for this sort of use case--no need to keep track of the previous break position and feed it back in.

It is easy to create a stateless API based on this stateful one, or vice versa, in user JavaScript code.

Q: Why is this an Intl API instead of String methods?

A: All of these break types are actually locale-dependent, and some allow complex options. The result of the `segment` method is a SegmentIterator. For many non-trivial cases like this, analogous APIs are put in ECMA-402's Intl object. This allows for the work that happens on each instantiation to be shared, improving performance. We could make a convenience method on String as a follow-on proposal.
