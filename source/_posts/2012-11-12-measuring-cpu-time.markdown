---
layout: post
title: "Measuring CPU time, Boost and C++11"
date: 2012-11-25 19:51
comments: true
categories: 
published: true
toc: true
---

The first set of algorithms I am going to discuss will be related to
the elementary steps of suffix array construction, in particular
**trigram sorting** and **lexicographic renaming**, and I will start a
series of blog posts about this soon.

However, to be able to compare various implementations of those
algorithms, it's important to use a clearly defined and consistent
approach to **measuring time**. In addition to using Linux tools like
[time](http://linux.die.net/man/1/time) and
[ps](http://linux.die.net/man/1/ps), I would like to be able to build
precise time-measurements into the C++ code, so as to be able to
measure precisely what amount of time is spent in which part of the
program.

##Two types of time

There are two types of time I need to be able to measure:

 * __Real time__ (aka [wall clock time](http://en.wikipedia.org/wiki/Wall_clock_time)):
   The actual time that passed between a predefined starting point and
   an end point, measured in seconds.

 * __CPU time__ (for example described
   [here](http://en.wikipedia.org/wiki/CPU_time)): The number of ticks
   (of the CPU clock) that passed _while the CPU was busy performing
   the process being measured_. This does not include time spent
   during non-CPU related activities, such as IO, and it does not
   include time the CPU spent performing other processes. While CPU
   time is measured in clock ticks, it is usually converted to seconds
   by dividing it by the _number of clock ticks per second_, which on
   Linux you can retrieve using a system call:
   `::sysconf(_SC_CLK_TCK)`.

There is a third type of time that will be important:
__Thread-specific CPU time__, but I won't talk about this in the
current post.

##Library support
###C++11

C++11 offers a convenient method to measure real time, **but not CPU
time**, through the [`<chrono>` library](http://en.cppreference.com/w/cpp/chrono#chrono_library).
Specifically, there are three functions that return a representation
of the _current point in time_:

    std::chrono::system_clock::now()  // System clock, may be modified by the OS while
                                      // the process is running, e.g. when switching
                                      // to daylight-saving time

    std::chrono::steady_clock::now() // Steady clock, guaranteed to run steadily from the
                                     // moment the process is started to its end

    std::chrono::high_resolution_clock::now() // Steady clock with high resolution

The existence of all three is required by the Standard, so they can be
used regardless of the platform the code is written for. The
`std::time_point` object returned by these functions can easily be
used to compute the time difference between two time points (using
`operator-`), and that difference can be readily converted to any unit
of choice (using `std::chrono::duration_cast`):

{% codeblock lang:cpp %}
auto start = std::chrono::system_clock::now();
/* .. perform work .. */
auto end = std::chrono::system_clock::now();

int elapsed_seconds = std::chrono::duration_cast<std::chrono::seconds>
                         (end - start).count();

std::cout << "Elapsed time: " << elapsed_seconds << " seconds" << std::endl;
{% endcodeblock lang:cpp %}

The benefits of this are **portability** and **convenience**, but
unfortunately this can only be used to measure real time, not CPU
time.

###Boost

The Boost library provides a similarly designed framework with the
same clocks for real time (in fact, of course, `std::chrono` was
designed using `boost:chrono` as a model). In addition, there are
three more clocks:

    boost::chrono::process_user_cpu_clock::now()   // User CPU time since the process
                                                   // started (including user CPU time
                                                   // of child processes)

    boost::chrono::process_system_cpu_clock::now() // System CPU time since the process
                                                   // started (including user CPU time
                                                   // of child processes)

    boost::chrono::process_real_cpu_clock::now()   // Wall-clock time, measured using the
                                                   // CPU clock of the process


Boost also provides a **combined clock**,
`boost::chrono::process_cpu_clock`, which gives access to time points
for all three types of CPU clock. This is also **portable** and
**convenient**, but I had a **number of problems** with this:

### Problems with Boost

* `boost::chrono::time_point` objects returned by the Boost clocks are
**not compatible with `std::time_point`**, which I find makes the code
*look confusing when you use both the time points and duration object
*from `boost::chrono` and `std::chrono` in parallel.

* The Boost CPU clocks measure time in terms of CPU ticks and
  immediately convert this to nanoseconds. The problem with this on
  typical Linux installations is that a clock tick is a relatively
  long unit of time; on my machine it is 10 milliseconds in
  length. This means that even when you measure a relatively short
  process, e.g. 2 seconds CPU time, the internal representation of
  this used by the Boost library will be 2 billion nanoseconds. The
  data type used to store the value is `std::clock_t`, which on 32-bit
  Linux is typically a signed 32-bit integer, which means 2 billion is
  very close to overflow. As a result, you get **overflows and negative
  run times even for processes of just a few seconds (even sub-second
  if several threads are involved).**

* There doesn't seem to be any **pretty-print** function, nor
**unit-converters** for the combined user/system/real clock Boost
provides. I am usually interested in measuring all three of them, but
`boost::chrono::process_cpu_clock::now()` returns time point objects
that are triples, and none of the convenient duration-casts and no
pretty-printing seems to be available for them.

## New implementation: `rlxutil::combined_clock`

What I wanted is a clock that returns time points which include user,
system and real time, where real time is measured using the
high-resolution clock from `std`, and user and system time are
measured using a platform-specific implementation, but using
`std::chrono::time_point` objects (and `std::chrono::duration` objects
when the difference between two time points is calculated).

I ended up implementing this myself. It now exists in the form of the
[util/proctime.hpp](https://github.com/jogojapan/sufex/blob/master/src/util/proctime.hpp)
header within the sufex repository. The main class is
`rlxutil::combined_clock<Precision>`. Note the template parameter
`Precision`: You can use any of the `std::ratio` types defined in
`<ratio>`, e.g. `std::micro` or `std::nano` as argument. This will
determine the internal representation of the time points: As
nanoseconds when `std::nano` is used, as microseconds when
`std::micro` is used, and so on.

### Usage
Here is an example of how it's used:

{% codeblock combined_clock usage lang:cpp https://github.com/jogojapan/sufex/blob/master/src/util/app/combined_clock.cpp %}
#include "util/proctime.hpp"

#include <ratio>
#include <chrono>
#include <thread>
#include <utility>
#include <iostream>

int main()
{
  using std::chrono::duration_cast;
  using millisec   = std::chrono::milliseconds;
  using clock_type = rlxutil::combined_clock<std::micro>;

  auto tp1 = clock_type::now();

  /* Perform some random calculations. */
  unsigned long step1 = 1;
  unsigned long step2 = 1;
  for (int i = 0 ; i < 50000000 ; ++i) {
    unsigned long step3 = step1 + step2;
    std::swap(step1,step2);
    std::swap(step2,step3);
  }

  /* Sleep for a while (this adds to real time, but not CPU time). */
  std::this_thread::sleep_for(millisec(1000));

  auto tp2 = clock_type::now();

  std::cout << "Elapsed time: "
            << duration_cast<millisec>(tp2 - tp1)
            << std::endl;

  return 0;
}
{% endcodeblock %}

Output generated by this program:

    Elapsed time: [user 40, system 0, real 1070 millisec]


### Implementation

The static `now()` function generates a time point by calling
`::times` to retrieve user and system time, as well as calling
`std::chrono::high_resolution_clock::now()` to retrieve real time. The
user and system times are then converted from ticks to the unit
determined by `Precision`. This requires a tick factor, which depends
on the number of ticks per second `::sysconf(SC_CLK_TCK)` and
`Precision`:

{% codeblock rlxutil::tickfactor<Precision> lang:cpp https://github.com/jogojapan/sufex/blob/master/src/util/proctime.hpp %}
/* Code related to log messages removed. */
template <typename Precision>
static std::intmax_t tickfactor() {
  static std::intmax_t result = 0;
  if (result == 0)
  {
    result = ::sysconf(_SC_CLK_TCK);
    if (result <= 0)
      result = -1;
    else if (result > Precision::den)
      result = -1;
    else
      result = Precision::den / ::sysconf(_SC_CLK_TCK);
  }
  return result;
}
{% endcodeblock %}

The key differences to the Boost implementation are:

* We use `std::intmax_t` instead of `clock_t`, which on typical
  systems (including 32-bit systems) provides us with 64 bits to store
  the internal representation of the time point;

* We use the unit suggested by `Precision` rather than defaulting to
  nanoseconds.

Both help avoid overflows and spurious negative duration
results. Specifically, the `Precision` parameter can be used by the
caller to obtain a representation that suits the purpose: For
measuring very short-lived processes, `std::micro` may be best, while
longer processes can be measured using `std::milli`.

The clock data type itself has been implemented following the
requirements by the C++ Standard.

{% codeblock rlxutil::combined_clock<Precision> interface lang:cpp https://github.com/jogojapan/sufex/blob/master/src/util/proctime.hpp %}
template <typename Precision>
class combined_clock
{
  static_assert(std::chrono::__is_ratio<Precision>::value,
      "Precision must be a std::ratio");
public:
  typedef rep_combined_clock<Precision>                     rep;
  typedef Precision                                         period;
  typedef std::chrono::duration<rep,period>                 duration;
  typedef std::chrono::time_point<combined_clock,duration>  time_point;

  static constexpr bool is_steady = true;

  static time_point now() noexcept;
};
{% endcodeblock %}

The internal representation of a time point, `combined_clock::rep` is
a tuple nested into a struct:

{% codeblock rlxutil::rep_combined_clock<Precision> definition lang:cpp https://github.com/jogojapan/sufex/blob/master/src/util/proctime.hpp %}
template <typename Precision>
struct rep_combined_clock
{
  typedef Precision                             precision;
  typedef std::tuple<
      std::intmax_t,
      std::intmax_t,
      std::chrono::high_resolution_clock::rep>  rep;

  rep _d;
};
{% endcodeblock %}

The struct wrapper was necessary to distinguish it clearly from tuple
types that originate somewhere else: The `operator-` and
`duration_cast` function used to calculate elapsed time only accept
the struct defined above, not general tuples.

The functions below were implemented inside namespace `std` to enable
pretty-printing, elapsed-time calculation and duration-casts:

{% codeblock Convenience functions lang:cpp https://github.com/jogojapan/sufex/blob/master/src/util/proctime.hpp %}
template <typename Precision, typename Period>
std::ostream &operator<<(
    std::ostream &strm,
    const chrono::duration<rlxutil::rep_combined_clock<Precision>,Period> &dur);

template <typename Pr, typename Per>
constexpr duration<rlxutil::rep_combined_clock<Pr>,Per>
operator-(
    const time_point<rlxutil::combined_clock<Pr>,
                     duration<rlxutil::rep_combined_clock<Pr>,Per>> &lhs,
    const time_point<rlxutil::combined_clock<Pr>,
                     duration<rlxutil::rep_combined_clock<Pr>,Per>> &rhs);

template <typename ToDur, typename Pr, typename Period>
constexpr duration<rlxutil::rep_combined_clock<Pr>,typename ToDur::period>
duration_cast(const duration<rlxutil::rep_combined_clock<Pr>,Period> &dur);
{% endcodeblock %}
