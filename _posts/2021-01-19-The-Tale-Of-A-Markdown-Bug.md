---
toc: false
badges: false
layout: post
description: Headings or subheadings randomly not working in markdown? There's a reason, and here's how to fix it. In this post I recount an evil markdown bug that took way too much time to fix.
categories: [misc]
keywords: markdown, headings not working, subheadings not working, markdown subheadings bug, non-breaking space, nbsp, md
title: The Tale of a Markdown Bug
image: images/md-preview.png
---

Ok this happened -- I was writing an blog post and was making liberal use of subheadings, as one does when explaining a topic in a structured way. Upon saving and previewing the post some headings simply did not render. I tried reloading multiple times, editing the spacing, ensuring there was 1 space between the hashtags and the words, anything I could think of. Here's what the issue looked like:

Here's the markdown:

```markdown
---

## Subheading that doesn't work

## Subheading that works

---
```

And here is the output:

---

## Subheading that doesn't work

## Subheading that works

---

What the heck?!

## Blaming Fastpages

Now for some context: my blog is currently powered by [fastpages](github.com/fastai/fastpages),  an opinionated customization layer on top of Jekyll that allows for quick publishing of Jupyter notebooks and markdown files hosted on github.io with a slightly tweaked minima interface. Chances are you're still looking at it right now.

Given the recent launch of fastpages, I suspected there was a bug in the markdown rendering pipeline, mainly due to the fact that my post made heavy use of lists and tables, potentially causing some parsing conflicts. So I tried fiddling with the text itself, trying to see if I could get around the parsing bug. I tried adding text before/after the heading, removing earlier parts of the post to see if a strange character in a code block was causing butterfly effects in the parsing several paragraphs down... nothing!

## Establishing a Ground Truth

To check whether fastpages was the culprit, I then searched for a couple online markdown previewers as well as VSCodes own markdown previewer tab. Suprise suprise...the markdown text above didn't work there either. Here's an example:

![]({{ site.baseurl }}/images/md-code.png "The markdown input")

![]({{ site.baseurl }}/images/md-preview.png "The previewed output")

So I downloaded a VSCode extension called Markdown Lint to check whether I had made a noobish mistake. The linter identified several issues such as extra line breaks where there shouldn't have been, trailing spaces, missing line breaks before a list, etc. But that heading seems perfectly A-OK for the linter!

![]({{ site.baseurl }}/images/md-linter.png "Are you sure about that Markdown Lint?")

**Ok, something's up.**

## Occam's Razor

It's in situations like these, Occam's Razor always comes to mind:

> Occam's razor, or law of parsimony is the problem-solving principle that "entities should not be multiplied without necessity", or more simply, **the simplest explanation is usually the right one**. - _Wikipedia_

There was something I hadn't tried. I had only cut, copied, pasted, adjusted, changed the title, moved the title around...but I had never actually tried to delete it and write it from scratch. A moment of deep calm surrounds me and I hit the backspace character until the whole heading disappears--and I delete the line break for good measure. I press enter again and type a new heading. It works. Damn you Occam! But why? Well, Occam again: if there's a problem with the heading...the problem is probably with the heading, not everywhere else. So I start dissecting the heading and evalute possibilities:

- The line break is a weird unix/windows mix that is causing conflicts
- The hashtags are special character versions
- The space inbetween the hashtags and the heading text is a special character

I had never heard of special hashtag characters, and the linebreaks I had tried to add and remove multiple times, so I ruled them out. This leaves us with our space character. Now the internet is full of demonstrations of how to leverage form vulnerabilities, perform XSS injections and in general mess with software with weird spaces. So I copied the spaces from a heading that worked, and the heading that didn't work and fired up the old Javascript console. Like a surgeon, I took out my scalpel of choice, that I lovingly called `String.prototype.charCodeAt()`, and got to work.

![]({{ site.baseurl }}/images/md-charcodeat.png)

A-ha! Those numbers are decimal charcodes. I'll give you a hint: 32 is the decimal charcode for a regular space. Can you guess what 160 is? If you're a webdev keep it to yourself, don't spoil it for the others! Charcode 160, aka `&nbsp;` in HTML, is called a **non-breaking space**. Essentially identical to a normal space in every single way, including it's unique ability to go undetected in Markdown Lint, except it's a _completely different character_. It's usually used when you don't want a trailing space or the space between two HTML tags to be automatically ignored. Ironic that it was ignored by both the markdown parser and the markdown linter.

## Conclusions

Adventurers, please be wary of the insidiousness of the evil non-breaking space. It will strike when you least expect it. Oh, and I made Markdown Lint aware of the [issue](https://github.com/DavidAnson/markdownlint/issues/367), so hopefully it won't happen to you.