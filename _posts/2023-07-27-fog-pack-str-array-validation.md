---
layout: post
title: fog-pack: better string & array validation
---

There's nothing quite like trying to use your crate to help you understand what 
it lacks. Using [fog-pack][fog-pack], I've been writing little data structures 
and schemas for various things, and my latest attempt has been to make one 
covering [MathML Core][MathML]. I ultimately hit three problems:

1. Keeping the schema in sync with a non-trivial set of data structures is 
	tricky and unreliable.
2. MathML has global "data-*" attributes with an annoying set of name 
	restrictions.
3. Encoding the IDs in a way that enforces uniqueness isn't actually possible.

Keeping the schema in sync is a problem I'm going to tackle later by making a 
new crate, fog-schemars, styled heavily after how the [schemars][schemars] crate 
works. As for the others, I realized I had need of some new validator 
capabilities in fog-pack.

[MathML]: https://www.w3.org/Math/
[schemars]: https://graham.cool/schemars/
[fog-pack]: https://crates.io/crates/fog-pack

String validation - Banning certain prefixes, suffixes, and characters
-----

So, I'm probably not going to put this in the final MathML-like schema, but it 
was good to see what kinds of naming convention rules are out there. The 
["data-*" attributes][attr] on MathML elements can have any name to replace the
`*` given the following restrictions:

- Follow the [production rules of XML names](https://www.w3.org/TR/REC-xml/#NT-Name)
- The name must not start with xml (case-insensitive).
- The name must not contain any colon characters (:).
- The name must not contain any capital letters.

The production rules are a bit verbose, but pretty easily captured in a regex - 
`NameStartChar` as the first character followed by one or more `Namechar`. 
Except I also have to capture that it doesn't start with: xml, XML, xmL, xMl, 
xML, Xml, XmL, or XMl. Easy, you say, just use negative lookahead! Except *I 
can't*, because the regex crate used by fog-pack doesn't support lookaround, and 
for good reasons (not supporting it guarantees linear time complexity). So how 
do we work around that? Well... if it really is just banning certain prefixes, 
that's easy enough to check for, though it means needing a new validation 
capability for String types. I've encountered this pattern a number of other 
times now, in the form of reserved prefixes or suffixes, so making both of those 
capabilities seems good. And while we're banning prefixes and suffixes, might as 
well add a "prohibited character" option too, which should let us use regexes a 
bit more infrequently. No need to use a regex just to say "this string can't 
contain the NUL character". So, let's add those as:

- `ban_prefix`: an array of strings the checked string can't begin with
- `ban_suffix`: an array of strings the checked string can't end with
- `ban_char`: a string of characters that must not appear in the checked string

And we'll also add `ban` as a query permisson flag to let queries use these as 
well. Cool. Finally, we'll make sure the Unicode Normalization setting carries 
through to the prefixes and suffixes as well, mirroring what we already did to 
the regex checks. We won't normalize the `ban_char` since that's really a list 
of characters, not a true string.

Cool, now we could support this type of naming convention! If we wanted to.

At the same time as doing this, I realized it'd be good to have for the Key 
Validation of Maps, but the KeyValidator structure was getting to look more and 
more like just another StrValidator. So I made it exactly that! Much easier to 
make a string validator do validation of key strings than whatever nonsense I 
was doing before.

[attr]: https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/data-*

Array validation - Forcing equal-length arrays
-----

MathML (and XHTML, and probably other things) has this notion of a an optional 
unique ID for an element in the tree. How can we enforce that inside a schema? 
The short answer is we can't, at least if we embed it directly inside the
element. But we *can* make a Map where the unique ID is the key and the value is 
a sequence of numbers that points to the exact item in the tree! If we do that, 
the IDs are checked for uniqueness by virtue of the Map requiring all keys be 
unique.

But wait, that only guarantees uniqueness in one direction - multiple unique IDs 
can point to the same element. And there's no existing validation rule we can 
add to change that. So we need a new one...

One option is to add a rule that requires all Map values be unique. That would 
work, but I think I can solve a broader range of issues at the same time if I 
use a different rule. See, when I made fog-pack require Map keys always be 
strings, I made working with it a bit easier, but I shut out BTreeMaps and 
HashMaps that have non-string keys. And there wasn't a way to enforce that 
1-to-1 Key-value mapping while also enforcing uniqueness of keys. Unless...

What if we added a rule to Maps that requires certain keys all have arrays of 
equal length for their values? For example, one that would pass:

```json
{
	"key": [0, 2, 4, 8],
	"val": [1, 2, 3, 4]
}
```

But would ban:

```json
{
	"key": [0, 2],
	"val": [1, 2, 3, 4]
}
```

Let's add that as a rule to Map validators: make `same_len`, which contains an 
array of key names. To pass, all of those keys must either be absent in the 
checked Map, or they must all be present and have arrays of the same length. 
We'll also add `same_len_ok` as a query permission flag so queries can use this 
(though I doubt many ever will - it's more for completeness' sake).

This lets us create two arrays, require they be the same length, and require 
that both of them be completely unique, ensuring a perfect one-to-one mapping 
between IDs and nodes in the tree. We can now also encode HashMaps and BTreeMaps 
as two separate arrays of equal length, letting us encode arbitrary key types. 
And we can enforce even more restrictions on them, like ensuring this 1-to-1 
correspondence. Plus, structs that contain many arrays whose lengths are all the 
same can also be enforced at the schema level - A
[Structure of Arrays instead of an Array of Structures][aos_soa].

Fantastic, and all it took was one more validation rule.

[aos_soa]: https://en.wikipedia.org/wiki/AoS_and_SoA

Looking Forward
-------

Every time I add or change validation rules, I worry about how much more I'll 
have to change in the future. Will this ever stabilize? Where should I draw the 
line on validation complexity? Well, the good thing is that so far, both of 
these rules were ones I'd been thinking of for at least a year, so at least the 
rate of additions has been quite slow. And I'm still mostly above and beyond the 
capabilities of JSON Schemas, and that's a standard that's been subject to way 
more scrutiny than little old fog-pack.

That said, it does occur to me that if I do ever add more rule options, existing 
schemas won't need to change to accomodate them, and it'll be limited to 
whatever new schemas exist after the rules are added. Given that schemas are 
generally only used by programs that already know about them, I probably don't 
have to worry about it overmuch.
