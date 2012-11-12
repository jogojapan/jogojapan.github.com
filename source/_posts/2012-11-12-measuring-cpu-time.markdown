---
layout: post
title: "Measuring CPU time, Boost and C++11"
date: 2012-11-12 14:20
comments: true
categories: 
published: false
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

##Two types of time##

There are two types of time I need to be able to measure:

 * __Real time__ (aka [wall clock time](http://en.wikipedia.org/wiki/Wall_clock_time)):
   The actual time that passed between a predefined starting point and
   an end point, measured in seconds.

 * __CPU time__ (for example described [here](http://en.wikipedia.org/wiki/CPU_time): The number
   of ticks (of the CPU clock) that passed _while the CPU was busy
   performing the process being measured_. This does not include time
   spent during non-CPU related activities, such as IO. While CPU time
   is measured in clock ticks, it is usually converted to seconds by
   dividing it by the _assumed number of clock ticks per second_
   `CLOCKS_PER_SEC`. On Linux this is a OS-defined constant unrelated
   to the _real_ number of ticks per second, hence the resulting
   number cannot be used to compare CPU time to real time. It can only
   be used to compare the CPU time of one process to that of another
   one running on the same system.

There is a third type of time that will be important:
__Thread-specific CPU time__, but I'll talk about this in a later
post.

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
unfortunately this can only be used to measure real time.

###Boost



Here is the definition of `time_point`:

{% codeblock std::chrono::time_point lang:cpp https://github.com/jogojapan/sufex/blob/master/src/util/proctime.hpp %}
template <typename Duration>
class time_point<rlxutil::cpu_clock,Duration>
{
public:
  typedef rlxutil::cpu_clock         clock;
  typedef Duration                   duration;
  typedef typename Duration::rep     rep;
  typedef typename Duration::period  period;

  constexpr time_point() : _d(duration::zero())
  { }

  constexpr explicit time_point(const duration& _dur)
    : _d(_dur)
  { }

  // conversions
  template<typename Dur2>
  constexpr time_point(const time_point<clock, Dur2>& _t)
    : _d(_t.time_since_epoch())
  { }

  constexpr duration
  time_since_epoch() const
  { return _d; }

  time_point&
  operator+=(const duration& _dur)
  {
    _d += _dur;
    return *this;
  }

  time_point&
  operator-=(const duration& _dur)
  {
    _d -= _dur;
    return *this;
  }

  static constexpr time_point
  min()
  { return time_point(duration::min()); }

  static constexpr time_point
  max()
  { return time_point(duration::max()); }

private:
  duration _d;

};
{% endcodeblock %}
