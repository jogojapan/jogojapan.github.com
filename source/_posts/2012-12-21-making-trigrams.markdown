---
layout: post
title: "Making trigrams"
date: 2012-12-21 19:57
comments: true
published: false
categories: 
---

Today I want to quickly introduce the first building block of the
suffix array library that is part of Sufex: Trigram generation.

One of the most important modern algorithms for suffix array
generation is the **DC algorithm**, also known as **Skew algorithm**
([DOI
10.1007/3-540-45061-0_73](http://link.springer.com/chapter/10.1007%2F3-540-45061-0_73?LI=true)),
discovered by Juha Kärkkäinen and Peter Sanders in 2003.

I won't describe the algorithm here, however, its first step is to
generate trigrams from the input test and sorting them
lexicographically. The algorithm does not generate _all_ trigrams, but
only those located at positions not divisible by three, i.e. positions
`p` such that `p % 3` is `1` or `2`. The module of Sufex that performs
this step is
[`sux/trigram.hpp`](https://github.com/jogojapan/sufex/blob/master/src/sux/trigram.hpp).


# Trigram implementation

In later steps of the algorithm, Sufex needs to access the trigrams at
the place where there are found in the input text. Therefore, the
trigram implementation must include either the relative position of
the trigram within the text, or a pointer to the character location.

During the sorting step, lots of trigrams must be compared to each
other, which requires access only to the three characters that belong
to each trigram. For better cache-efficiency, it is therefore useful
to actually store these three characters inside the trigram
representation, along with the position information or
pointer. However, this isn't _necessary_.

Sufex provides three different implementations of trigrams, and the
sorting algorithms are designed to work with any of them:

{% codeblock Trigram implementations lang:cpp %}
namespace sux {
  /* Three implementations of trigrams: */
  enum class TGImpl { tuple, arraytuple, pointer };

  /* Data type for the trigram implementation: */
  template <TGImpl tgimpl, typename Char, typename Pos>
  struct TrigramImpl;
}
{% endcodeblock lang:cpp %}

`Char` and `Pos` represent data types for individual characters and
position/offset information, respectively. Hence, when using `char`
for characters and `long` for positions, a single trigram is of one of
the three types below:

{% codeblock Trigram implementations lang:cpp %}
using namespace sux;

/* Using a single pointer to represent the trigram: */
TrigramImpl<TGImpl::pointer,char,long>    trigram1;

/* Using std::tuple<long,char,char,char> to encode position
   and contents of the trigram: */
TrigramImpl<TGImpl::tuple,char,long>      trigram2;

/* Using std::tuple<long,std::array<char,3>> to encode position
   and contents of the trigram: */
TrigramImpl<TGImpl::arraytuple,char,long> trigram3;
{% endcodeblock lang:cpp %}

The first one, `TGImpl::pointer`, consists of a single pointer `char
*` only and is therefore small in memory but relatively slow during
sorting.

The second one, `TGImpl::tuple`, uses a `std::tuple` to store both the
relative position of the trigram and its contents. This is the fastest
implementation during sorting.

The third one, `TGImpl::arraytuple`, uses a combination of
`std::tuple` and `std::array` to store the relative position and the
contents of the trigram. It is a little slower than `TGImpl::tuple`,
but can more easily be extended to store n-grams with higher n, which
may become important in alternative implementations of suffix array
construction.

# Generating trigrams

The DC algorithm involves creating an array of trigrams at positions p
such that `p % 3 = 1,2`. For this, the `TrigramMaker` can be used:

