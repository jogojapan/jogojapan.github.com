---
layout: post
title: "Making trigrams"
date: 2012-12-21 19:57
comments: true
published: true
toc: true
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

Sufex provides **three different implementations** of trigrams, and
the sorting algorithms are designed to work with any of them. The
three types are specified by the values of an enumeration called
`sux::TGImpl`, while the actual data type for trigrams is defined as a
struct called `sux::TrigramImpl`:

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
for characters and `long` for positions, a single trigram can be
stored in one of three data structures shown below:

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
such that `p % 3 = 1,2`. For this, the `sux::TrigramMaker` can be
used:

{% codeblock Trigram generation lang:cpp %}
  using namespace std;
  using namespace sux;
  string input { "abcabeabxd" };

  /* Generating trigrams. */
  auto trigrams =
      TrigramMaker<TGImpl::tuple,string::value_type,size_t>::make_23trigrams(begin(input),end(input));

  /* Printing them. */
  for (const auto &trigram : trigrams)
    cout << triget1(trigram) << triget2(trigram) << triget3(trigram) << '\n';
  cout.flush();
{% endcodeblock lang:cpp %}

Evidently, `TrigramMaker` takes three template arguments:

 * The trigram implementation (`TGImpl::tuple`, `TGImpl::arraytuple`
   or `TGImpl::pointer`);
 * The character type (e.g. `char`)
 * The position type (e.g. `unsigned long`)

It's `make_23trigrams` function must be applied to a range of
characters specified by two iterators. The value returned is a
`std::vector` of trigrams.

There is a **convenience function** `sux::string_to_23trigrams()` that
can be used if the input is stored in a `std::basic_string`:

{% codeblock Trigram generation lang:cpp %}
  using namespace std;
  using namespace sux;
  string input { "abcabeabxd" };

  /* Generating trigrams. */
  auto trigrams = string_to_23trigrams<TGImpl::arraytuple>(input);

  /* Printing them. */
  for (const auto &trigram : trigrams)
    cout << triget1(trigram) << triget2(trigram) << triget3(trigram) << '\n';
  cout.flush();
{% endcodeblock lang:cpp %}

It takes only one template argument, the trigram implementation. It
defaults to `TGImpl::tuple`.


# Sorting trigrams

Trigrams are sorted in three passes of radix sort. During the first
pass, trigrams are put into buckets according to the third character
of each trigrams. During the second pass, they are stably sorted into
buckets defined by the second character, and during the third pass
they are stably sorted into buckets defined by the first
character. Each pass can be parallelized by running several radix-sort
threads in parallel, each on a different chunk of the trigrams.

The implementation is encapsulated in the `sux::TrigramSorter`
template, which can be applied to any implementation of trigrams,
i.e. regardless of whether `TGImpl::tuple`, `TGImple::arraytuple` or
`TGImpl::pointer` is used, and regardless of the data types used for
characters and positions.

Again, there is a **convenience function** `sux::sort_23trigrams()`,
which takes two arguments:

 * The trigram container (i.e. what was returned by
   `make_23trigrams()`)
 * The desired number of threads (defaults to 1)

Hence, the easiest way to create the 2,3-trigrams from a `std::string`
and sort them using parallel 2 threads is this:

{% codeblock Trigram generation lang:cpp %}
  using namespace std;
  using namespace sux;

  /* Input. */
  string input { "abcabeabxd" };

  /* 2,3-Trigrams. */
  auto trigrams = string_to_23trigrams(input);

  /* Trigram sorting using 2 parallel threads. */
  sort_23trigrams(trigrams,2);

  /* Printing the results. */
  for (const auto &trigram : trigrams)
    cout << triget1(trigram) << triget2(trigram) << triget3(trigram) << '\n';
  cout.flush();
{% endcodeblock lang:cpp %}


# Running times for different trigram implementations

I've compared running times of trigram sorting for the three
implementations of trigrams, and for two different sorting algorithms:
The radix sort described above, and `std::stable_sort`.

The comparison involved running each version once on a dual-core Intel
machine (2 T9300 cores, L1-cache 32KB per core, L2 cache 6144 KB for
both cores, 4 GB RAM) and measuring user time, system time and real
time in milliseconds. The results are reported in [this Google spreadsheet](https://docs.google.com/spreadsheet/pub?key=0AqDFvV2_pwzSdHBKSWRrdWdUUzFiSnhENnFJcFljWUE&output=html).

{% img http://jogojapan.github.com/images/trigram-running-time.png 650 550 4-threads radix sort vs. 1-thread std::stable_sort %}

**Main observations**

1. `TGImpl::tuple` and `TGImpl::arraytuple` are equally fast, but
`TGImpl::pointer` is considerably slower. This is no surprise, as
using pointer requires derefencing it in each comparison step.
1. In **one-thread mode**, `std::stable_sort` is faster than my radix
sort implementation. I believe this is mainly caused by the fact that
radix sort needs to **determine the alphabet** and count the number of
occurrences of every character, in *each* radix pass. This is
acceptable, because during the recursive steps of suffix array
construction (not yet implemented), an integer alphabet is used, which
does not require this step.
1. In **multithreading mode**, radix sort is **much faster in terms of
real time** (yellow bars) than the built-in `std::stable_sort` (which
runs on one thread only).
