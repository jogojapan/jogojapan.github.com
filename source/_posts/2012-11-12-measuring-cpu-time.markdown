---
layout: post
title: "Measuring CPU time, Boost and C++11"
date: 2012-11-12 14:20
comments: true
categories: 
published: false
---
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
