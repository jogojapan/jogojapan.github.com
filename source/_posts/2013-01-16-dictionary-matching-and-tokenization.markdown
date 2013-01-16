---
layout: post
title: "Dictionary Matching and Tokenization"
date: 2013-01-16 17:42
comments: true
published: false
categories: 
---

Consider this:

U.K.

Assume we want to apply a country name dictionary.

We do expect the entry "UK" to match, as this would imply that token
boundaries can generally be ignored.

So we put "U.K." in the dictionary. This means we can't remove or
normalise all token boundaries of the input text before performing the
dictionary lookup.

Now consider two dictionary entries:

1) University of Tokyo
2) University of Tokyo Atacama Observatory

and an input text:

University of Tokyo.

When using longest-match, (2) will win due to the added token
boundary, which we cannot ignore because of the U.K. problem.
