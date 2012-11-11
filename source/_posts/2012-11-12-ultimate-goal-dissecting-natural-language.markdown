---
layout: post
title: "Ultimate Goal: Better Natural Language Processing"
date: 2012-11-12 00:02
comments: true
categories: 
---

Natural language processing has become big. Almost every university
employs at least a few researchers specializing in it, almost every
major internet company hires NLP specialists to help analyse the
natural language part of "big data", and hundreds of workshops and
conferences are held every year.

Its methods have changed, too: Several decades ago, ideas based on
mathematical logic dominated the *semantic analysis*, and ideas based
on phrase structure grammars dominated the *syntactic
analysis*. However, for the past ten years or so, statistical machine
learning and large-scale corpus studies have dominated the field.

This blog is dedicated to its own variety of natural language
processing. Its principles are different from those of today's
mainstream NLP, and they are different from most (though probably not
all) of the NLP and computational linguistics of the past. The key
elements are:

 * __Lexeme-specific rules:__ The syntax of a language is described
   using rules, but there are far more rules than in traditional
   phrase structure grammars, and there are many rules that apply to a
   single lexeme only.

 * __Structural analogy:__ Sentence structures that cannot be analysed
   using existing rules are analysed in the same way as sentences that
   are structurally _similar_ or _analogous_. This notion of
   structural analogy will be defined in terms of predetermined
   transformation rules as well as approximate (fuzzy) matching
   against corpora of example sentences.

 * __Interactive refinement:__ The formal description of the grammar
   (lexeme-specific rules, transformation rules and other elements) is
   created iteratively by testing intermediate stages against large
   corpora. To this end, a specialized corpus query system (CQS) is
   used.

A lot of this is still very unclear. I started my activities by
creating a CQS, and doing this involved a significant amount of
research into the architecture of search engines, and the data
structures and algorithms they use.

This blog is going to describe my approach to NLP as it evolves, and
it will do so in great detail. I'll start with the CQS, which I
decided to re-implement from scratch. I'll blog about my choice of
algorithms and implementation strategies, as well as specific &ndash;
and sometimes very minor &ndash; programming techniques as I rebuild
the CQS. This alone will take many months, perhaps more than a year.

I have also started to work on the definition of the lexeme-specific
rule framework and prototypical designs for a parser. Posts related to
this will start to appear in the blog soon, perhaps in a few weeks or
months from now. Over time, the focus will gradually shift from string
algorithms and implementation strategies towards grammar description
and linguistics, and the blog title, _Algorithms on Strings_, will be
increasingly

My programming language is C++ (C++11 and a lot of Boost), and the
operating system is Linux. The hardware I use includes both
Intel-based PCs and large Xeon-based servers with tens of CPU cores
and 1 TB of RAM.
