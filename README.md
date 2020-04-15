# Proposal For Local Font Access And Origin-Based Caching

**Existential Status:** personal draft, unapproved by anyone else so far

## Background ##

Related topics:

* <https://github.com/w3cping/font-anti-fingerprinting>
* <https://github.com/w3cping/font-fp-protection-browser-hinting>

The proposals linked above both propose
that browsers generally turn off webpages' access to local fonts,
and then turn access back on for smaller subsets
that protect user's privacy.
(That is,
which don't let hostile trackers
turn a user's set of installed local fonts
into a unique identifier.)

The two proposals so far agree on exposing fonts installed by the user's OS by default,
and having *some* sort of method for crawling for popular fonts to aggressively and proactively cache.

While these methods work well at addressing *most* cases for popular/needed fonts,
allowing pages to avoid loading webfonts in many cases,
they don't allow for sites that need rare or specific fonts for correct display.
For example:

* design/publishing sites that want to allow users to use their own font collection
* sites showcasing ancient/dead languages
	(such as Egyptology sites showing heiroglyphics)
	that aren't well-supported in system fonts
	and won't ever be popular enough to be "above the line" in the crawl data
* minority languages wanting to use a new font
	that's not yet popular enough to show up in the crawl data.


## Proposal ##

This proposal addresses the need explained above
by giving sites the ability to request individual local fonts from users,
exposing them as a new subclass of `FontFace`
and usable anywhere a `FontFace` object is,
and providing an easy persistence mechanism for these new objects
so that the entire origin can have access to them
on subsequent page loads
without requiring additional JS.

### `<local-font-picker>` ###

(Name subject to bikeshedding, of course.)

`<local-font-picker>` is a new HTML element
that allows a user to choose a local font from their device
and expose it for the web page to use.

The UI is expected to be `<button>`-like,
showing the element's contents in a button UI.
When clicked, the browser must show a font-chooser UI,
allowing the user to select a local font from their system.

The element defines a few attributes:

* `font-hint`: a string that is parsed as a CSS `font` property value.
	The font-chooser UI should present local fonts that would match this value
	as "preferred" or "default" in some way.

	(This allows sites which know the specific fonts they probably need
	to present that up-front,
	rather than requiring the user to scroll thru all their fonts.)
* `unicode-range`: a string that is parsed as a CSS `unicode-range` descriptor value.
	The font-chooser UI should filter fonts
	that do not have defined values for the listed codepoints
	and present them as a less-preferred choice,
	possibly omitting them altogether.
* `locale`: a string parsed as a locale (FIXME: specify what this means).
	The font-chooser UI should present fonts which are appropriate for that locale
	(FIXME: defined how? mapping to codepoint ranges?)
	as "preferred" or "default" in some way.

	(This one might be too handwavey, I'm not sure yet.
	IIRC the CSSWG has a similar problem
	of wanting to map language tags to codepoints ranges
	for easier specification in `unicode-range`;
	maybe we can just lean on that instead.)

In the DOM interface,
in addition to reflecting the above attributes with the obvious names,
a `.value` attribute
either contains `null`
(if the user hasn't yet selected a font)
or a `LocalFontFace` object representing the chosen font
(see below for details on that interface).

Like `<input type=file>`,
script should be able to call `.click()` on the element
from a user-initiated gesture,
even if the element isn't connected,
to trigger the font-chooser UI directly.

Like form controls,
"change" and "input" events are fired at this element
when the user makes a font selection.

### `LocalFontFace` Objects ###

`LocalFontFace` is a subclass of the existing `FontFace` interface,
representing an opaque reference to a local font.

It cannot be constructed directly;
calling the constructor throws.

Its various attributes are initialized from the font data,
reflecting its name,
weight,
etc.
(FIXME: Fill in details here.)
They can be changed as desired in script,
per the standard rules for `FontFace` objects.

Like `FontFace`, the underlying data is not exposed in any way.

(Note: <https://github.com/WICG/local-font-access> is a proposal
to expose the underlying binary data of a local font file,
gated behind higher permissions and sanitized/normalized first.
It should be compatible with this proposal,
possibly with some minor modifications.)

As a subclass of `FontFace`,
`LocalFontFace` can be added to `document.fonts`, etc.

`LocalFontFace` is structured-cloneable,
so it can be stored in IndexedDB,
sent to workers or service workers via `postMessage()`,
etc.
(Underneath the covers they're just a path to a local font file,
so this is cheap;
they're not huge blobs.)
(Issue: can we restrict transferring them to same-origin only?
Or equivalent security boundary?)

### document.localFonts ###

While `LocalFontFace` objects *can* be stored in IndexedDB,
fetched on page load,
and added to `document.fonts`,
that's a pretty clumsy interface
if you just want to ask a user for fonts the site can use.

It also means you might get a "flash of unfonted text",
for any text rendered before the script runs.
This is pretty unfortunate!

To avoid this and make the persistence use-case easier,
a new `window.localFontStorage` API is provided.
(Or should this be on `navigator` or `document`?
Haven't done research for similar cases,
but `localStorage` is on `window`.)

This property exposes a `LocalFontStore` interface,
which is an **async set-like**
that accepts only `LocalFontFace` objects.
(Async set-likes don't exist yet in WebIDL,
but it's just a set-like that returns promises for everything.)

`LocalFontFace` objects stored in `localFontStorage`
are considered available for font matching
(just like the fonts stored in `document.fonts`)
for *all same-origin documents*
(same exposure rules as `localStorage`).
`localFontStorage` object are also automatically persisted across page loads,
using the same logic for persistence, expiration, etc
as `localStorage`.

FIXME: Define the ordering of `localFontStorage` fonts
relative to all other fonts.

(Note that the operations are async,
so other pages will only see updates at a microtask checkpoint.)

The objects held in `localFontStorage` represent the same local font
(with the same `FontFace` attributes, etc),
but are not the same object;
pulling a `LocalFontFace` object out of the store
and mutating it
will *not* mutate the face exposed to the page
(or other pages);
you must put the mutated face back in the set to make the changes visible.
(I think this also implies that the operations that return a `LocalFontFace` would have to be `[NewObject]` to be reasonable?
This is awkward,
but getting fonts out of the set should be a rare operation.)

(Item equality in `localFontStorage` would be value-based, not reference based;
`LocalFontFace` objects referring to the same local font file,
and with identical attributes,
will be considered the same thing.
Repeatedly adding the same `LocalFontFace` to the set,
or even multiple different `LocalFontFace`s with the same data,
will reflect as only one item in the set.)

FIXME: Define a predictable ordering for the objects in `LocalFontStorage`.
Since they can come from outside the page,
can't rely on insertion order;
probably need to define a lexicographic ordering over the attributes or something.

## Additional Issues ##

### What about sites that won't get updated? ###

Tho this API was designed
to make adding exposing a local font to a site
as easy as possible
(just a few lines of very simple script),
it still requires the site to be written to do so.

If an old site is no longer maintained,
it won't have this capability;
if it expected users to install fonts locally
for it to use,
it won't get them.

This proposal **does not preclude**
user agents providing additional UI in their Settings pages or similar
allowing users to manually specify local fonts
that should be exposed to a given origin.
(They should be origin-restricted;
web-wide exposure violates privacy.)

If a user agent does so,
it must expose those user-provided local fonts
in the `localFontStorage` for the origin.
