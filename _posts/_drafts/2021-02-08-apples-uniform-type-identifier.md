---
layout: post
title: "Apple's Uniform Type Identifier"
date: 2021-02-08
categories: Foundations
---

Recently, I was trying to install a TrueType font on my phone. You see, at the moment I'm obsessed with design, and I wanted to install one of my favourite fonts, [IBM Plex Sans](https://fonts.google.com/specimen/IBM+Plex+Sans?preview.text_type=custom), on my iPhone. Naturally, I couldn't. It's not possible to install custom fonts on an iPhone, at least not without an unreasonable amount of finagling and downloading apps with names like 'FontMaster Pro' whose UIs are written entirely in what looks like Traditional Chinese.

So, naturally, I thought I'd write a quick app to remedy this. I doubt if anyone else is as keen as I am to install random fonts off Google Fonts onto their iPhones, but at least this would mean I'd be able to do it. And I quickly hit a hurdle.

To be able to come up in that special 'Open in...' menu when selecting a file on an iPhone, you need to go to Document Types in Xcode. There you need to enter the file extension (so far so good), and, subsequently, a field called the UTI. UTI in this context does not mean a urinary tract infection, but what it means actually turns out to be even more horrifying. Because what 'UTI' portends here is Apple's Uniform File Identifier.

## Why did Apple introduce the Uniform Type Identifier?

Apple's [scant documentation on the subject](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/understanding_utis/understand_utis_intro/understand_utis_intro.html) explains:

> One of the challenges facing application developers is the proliferation of methods to identify types of data. For example, some text files may be assigned a 'TEXT' file type (as originally designed for Mac OS 9 and earlier), while others may simply have a .txt filename extension. Some may have the .text extension instead. In addition, some file types might be subsets of other types; an application that opens all .txt files should probably also be able to open those with a .html extension. Determining all the possible files an application could read could become impossible. The user experience then suffers, with users not understanding why an application can open one text file but not another.

> To solve this problem, Apple has defined a syntax for special data identifiers called uniform type identifiers. Each UTI provides a unique identifier for a particular file type, data type, directory or bundle type, and so on. In addition, other type identifier namespaces for a particular type can be grouped under one UTI, with utility functions available to translate from one format to another.

In short, what Apple is saying here is that there is not a perfect one-to-one mapping between file extensions and abstract 'file types'. Sometimes two files may share an extension but be different in important ways (think different flavours of Markdown). Other times files may have a different extension but be the same in every other way (for example: just as the Inuit supposedly have a hundred words for 'snow', we have a hundred file extensions for JPEGs).

So far, so reasonable.
