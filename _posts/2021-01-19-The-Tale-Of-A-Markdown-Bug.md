---
toc: false
badges: false
layout: post
description: Headings or subheadings randomly not rendering in markdown? In this post I recount an insidious markdown bug that took me much way too long to track down.
categories: [misc]
keywords: markdown, headings not working, subheadings not working, markdown subheadings bug, non-breaking space, nbsp, md
title: The Tale of the Stochastic Markdown Bug
image: images/md-preview.png
---

I've recently started writing a blog as a way to "slow down" my learning and document things for my future self. Today I was typing out some quick notes for a blog post in markdown on VSCode and was making liberal use of subheadings, as one does when explaining a topic in a structured way. I like to type without distractions so I don't preview things right away. Upon saving and previewing the post some headings simply did not render. I tried reloading multiple times, editing the spacing, ensuring there was 1 space between the hashtags and the words, anything I could think of. Here's a condensed example of what the issue looked like:

The markdown (heading type is irrelevant, and horizontal lines added for demo confinement):

```markdown
---

## Subheading that doesn't work

## Subheading that works

---
```

And the output:

---

## Subheading that doesn't work

## Subheading that works

---

What the heck?!

## Blaming Fastpages

Now for some context: the post was LONG, and full of tables, image placeholders pointing to images I had not yet created, randomly indented lists and missing line breaks after the headings. I immediately thought some weird combo had broken the parser. Furthermore, my blog is currently powered by [fastpages](https://github.com/fastai/fastpages), an opinionated customization layer on top of Jekyll that allows for quick publishing of Jupyter notebooks as blog posts hosted on github.io with a slightly tweaked minima interface (further tweaked by me). Chances are you're still looking at it right now (highly recommended by the way).

Given the recent launch of fastpages, I suspected there was a bug in the markdown rendering pipeline, mainly due to the fact that my post made heavy use of lists and tables, potentially causing some parsing conflicts. So I tried fiddling with the text itself, trying to see if I could get around the parsing bug. I tried adding text before/after the heading, removing earlier parts of the post to see if a strange character in a code block was causing butterfly effects in the parsing several paragraphs down... nothing!

## Establishing a Ground Truth

To check whether fastpages was the culprit, I then searched for a couple online markdown previewers as well as VSCode's own markdown preview tab. Suprise suprise...the markdown text above didn't work there either. Here's an example:

![]({{ site.baseurl }}/images/md-code.png "The markdown input")

![]({{ site.baseurl }}/images/md-preview.png "The previewed output")

So I installed the most popular markdown linter VSCode extension ([Markdown Lint](https://github.com/DavidAnson/markdownlint)) to check whether I had made a mistake somewhere along the way that had propagated into an incorrect parsing of the headings. The linter identified several issues such as extra line breaks where there shouldn't have been, trailing spaces, missing line breaks before a list--everything...but that damn heading. After fixing everything that was flagged, the problem persisted and the heading seemed perfectly A-OK for the linter!

![]({{ site.baseurl }}/images/md-linter.png "Are you sure about that, Markdown Lint?")

**Ok, something's up.**

## Occam's Razor

It's in situations like these, Occam's Razor always comes to mind:

> Occam's razor, or law of parsimony is the problem-solving principle that "entities should not be multiplied without necessity", or more simply, **the simplest explanation is usually the right one**. - _Wikipedia_

There was something I hadn't tried. I had only cut, copied, pasted, adjusted, changed the title, moved the title around...but I had never actually tried to delete it and write it from scratch.

A moment of deep calm surrounds me. I hit the backspace character repeatedly until the whole heading disappears--including the line break for good measure. I press enter again and type a new heading _slowly_. It works. Damn you Occam! But why? Well, Occam again: if there's a problem with the heading...the problem is probably in the heading, not everywhere else. So I start dissecting the heading and evalute possibilities:

- The original line break was a weird unix/windows mix that is causing conflicts _(unlikely, using macOS)_
- The original hashtags were special character equivalents _(...are you drunk?)_
- The space inbetween the hashtags and the heading text was a special character _(only NOW you think of this?)_

I have never heard of "special" hashtag characters, and I had tried to add and remove the linebreaks multiple times, and I ruled both out more or less immediately--in my heart I already knew the answer. Had reading HN daily really not taught me anything? The internet is full of examples of how whitespace can be leveraged to exploit form vulnerabilities, perform XSS injections and in general mess with software in fun and interesting ways. So I copied the space character from a heading that worked, and from the heading that didn't work and fired up the old Javascript console. I took out my scalpel for the job, good ol' `String.prototype.charCodeAt()`, and got to work.

![]({{ site.baseurl }}/images/md-charcodeat.png)

A-ha! Those numbers are decimal charcodes. I'll give you a hint: 32 is the decimal charcode for a regular space. Can you guess what 160 is? If you're a webdev keep it to yourself, don't spoil it for the others! Charcode 160, aka `&nbsp;` in HTML, is called a **non-breaking space**. Essentially identical to a normal space in every single way, including its unique ability to go undetected in Markdown Lint, except for the fact that it's a _completely different character_. It's usually used when you don't want a trailing space or the space between two HTML tags to be automatically ignored. Ironic that it was ignored by both the markdown parser and the markdown linter.

Spaces are quite a rabbit hole. After a little research I found out there are (at least) 19 types of spaces.

```
U+0009 Horizontal tab (HT)
U+0020 Space
U+00A0 Non-break space
U+1680 Ogham space mark
U+180E Mongolian vowel separator
U+2000 En quad
U+2001 Em quad
U+2002 En space
U+2003 Em space
U+2004 Three-per-em space
U+2005 Four-per-em space
U+2006 Six-per-em space
U+2007 Figure space
U+2008 Punctuation space
U+2009 Thin space
U+200A Hair space
U+202F Narrow no-break space
U+205F Medium mathematical space
U+3000 Ideographic space
```

Granted it's unlikely that all 19 will insidiously occupy my markdown documents, I'll never trust an innocent looking space ever again!

## Conclusions

Adventurers, please be wary of the insidiousness of the evil non-breaking space. It will strike when you least expect it. Oh, and I made Markdown Lint aware of the [issue](https://github.com/DavidAnson/markdownlint/issues/367), so hopefully it won't happen to you. Side note: upon trying other online markdown tools, the "whitespace award" goes to [dillinger.io](https://dillinger.io) for correctly marking the non-breaking space as an illegal character.

**There's only one latent question**: how did that non-breaking space get there in the first place? Good question. If I find out, I'll update this post. If you find out, please reach out on Twitter at [muttonia@](https://twitter.com/muttonia)! Current suspects are:

- having copied and pasted the file from a non-markdown format (i.e. an untitled VSCode file, into a new .md file.)
- fastpages accidentally altering the contents in its (first) rendering pass if the post is not properly formatted with a header metadata block at the beginning (which it wasn't)
- me being an idiot

## Addendum: Mystery Solved

Turns out I _was_ an idiot. With an Italian keyboard layout, the `#` hashtag character is created by pressing the `Alt Gr` modifier key followed by the `à` key. Here's the keyboard layout for those not familiar:

![]({{ site.baseurl }}/images/eurkey.png "A typical it-IT keyboard layout, original from Wikimedia")

It _also_ turns out, that `Alt Gr` followed by the `space` key generates a **non-breaking space** (note: if this is common knowledge, I will bow my head in silence and sacrifice my `Caps Lock` key to the typing gods, preventing it's reincarnation as a remapped CMD or CTRL key).

So what happened? Well, when typing with haste, after inserting hashtags for the heading, my thumbs occasionally pressed `space` before the `Alt Gr` key had time to fully unpress, causing a non-breaking space to sneak in the heading. Since the spaces look identical, there was no visual queue on VSCode or Markdown Lint that a problematic character had sneaked in. The stochastic nature of the issue (i.e. depending entirely on my typing speed and finger timing) meant that it also re-occured at random in my early troubleshooting further confusing my bug triaging. I told you those damn spaces are insidious.

So I guess the ultimate Occam's Razor axiom is: if something goes wrong, it's probably your own damn fault. So, whenever you are facing a furiously difficult bug, remember Occam's Razor:

> Occam's razor, or law of parsimony is the problem-solving principle that "entities should not be multiplied without necessity", or more simply, **the simplest explanation is that you messed up**. - _Wikipedia, and me_
