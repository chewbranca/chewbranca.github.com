---
layout: post
title:  "On the Viability of Erlang Releases and CouchDB"
date:   2014-05-07 20:17:51
categories: tech
tags: erlang releases couchdb deployment
---


There has been some discussion on what versions of Erlang CouchDB
should support, and what versions of Erlang are detrimental to
use. Sadly there were some pretty substantial problems in the R15 line
and even parts of R16 that are landmines for CouchDB. This post will
describe the current state of things and make some potential
recommendations on approach.

## Scheduler Collapse

It was discovered by Basho that R15* and R16B are susceptible to
scheduler collapse. There's quite a bit of discussion and
information in several threads [1] [2] [3] [4] [5].

So what is scheduler collapse? Erlang schedulers can be put to sleep
when there is not sufficient work to occupy all schedulers, which
saves on CPU and power consumption. When the schedulers that are still
running go through enough reductions to pass the work balancing
threshold, they can trigger a rebalance of work that will wake up
sleeping schedulers. The other mechanism for sharing scheduler load is
work stealing. A scheduler that does not have any work to do can
steal work from other schedulers. However a scheduler that has gone
to sleep cannot steal work, it has to be woken up separately.

Now the real problem of scheduler collapse occurs when you take
sleeping schedulers and long running NIFs and BIFs that do not report
an appropriate amount of reductions. When you have NIFs and BIFs that
don't report an appropriate amount of reductions, you can get into a
situation where a long running function call will only show up as
taking one reduction, and never hit the work balance threshold,
causing that scheduler to be blocked during the operation and no
additional schedulers getting woken up.

I keep mentioning "NIFs and BIFs" because it's important to note that
it is _not_ just user defined NIFs that are problematic, but also a
number of Erlang BIFs that don't properly report reductions.
Particularly relevant to CouchDB are the BIFs `term_to_binary` and
`binary_to_term` which do _not_ behave properly, and each report a
single reduction count, regardless of the size of the value passed to
them. Given that every write CouchDB makes goes through
`term_to_binary`, this is definitely not good.

This problem is systemic to all versions of R15 and R16B. In R16B01,
two changes were made to alleviate the problem. First, in `OTP-11163`
`term_to_binary` now uses an appropriate amount of reductions and will
yield back to the scheduler. The second important change was the
introduction of the `+sfwi` (Scheduler Forced Wakeup Interval) flag
[6] which allows you to specify a time interval for a new watchdog
process to check scheduler run queues and wake up sleeping schedulers
if need be. These two changes help significantly, although from what I
understand, they do not fully eliminate scheduler collapse.

*NOTE*: the `+sfwi` is _not_ enabled by default, you must specify a
greater than zero time interval to enable this. *WE NEED TO ENABLE
THIS SETTING.* We should figure out a way to conditionally add this
to vm.args or some such.

On a side note, Basho runs R15B01 because they backported the `+sfwi`
feature to R15B01 [7] [8]. They recommend running with `+sfwi 500` for
a 500ms interval. It might be worth testing out different values, but
500 seems like a good starting point. For Riak 2.0, they will be
building against R16B03-1 and 17.0 as their set of patches to R16B02
landed in R16B03-1 [9] [10].

## R16B01 and the breaking of monitors

So R16B01 sorted out the scheduler collapse issues, but unfortunately
it also broke monitors, which immediately disqualifies this release as
something we should recommend to users. The issues was fixed in
`OTP-11225` in R16B02.

## R16B02 and R16B03*

I don't know of any catastrophic problems on the order of those
described above in either of these releases. Basho fixed a number of
unrelated bugs in R16B02 [9] [10] that have since landed in R16B03-1,
which indicates we should probably prefer R16B03-1 over R16B02. R16B03
is also disqualified because it broke SSL and `erl_syntax`, resulting
in the patched R16B03-1.

## R14

R14B01, R14B03, and R14B04 are known good stable releases of Erlang,
and in my opinion the only known stable releases > R13 that don't
present issues for CouchDB (I think R16B02/R16B03-1 are too new to
declare stable yet). As for R14B02, there are some bad `ets` issues
with that release.

It's worth pointing out that there are two known bugs in R14B01, as
Robert Newson explains:

```
There are two bugs in R14B01 that we do encounter, however. 1) Another
32/64 bit oops causes the vm to attempt to allocate huge amounts of
ram (terabytes, or more) if it ever tries to allocate more than 2gib
of ram at once. When this happens, the vm dies and is restarted. It’s
annoying, but infrequent. 2) Sometimes when closing a file, the
underlying file descriptor is *not* closed, though the erlang process
exits. This is rare but still quite annoying.
```

## Erlang 17.0

The 17.0 release brings in a number of interesting changes to help the
scheduler collapse situation. `OTP-11648` improves reduction cost and
yielding of `term_to_binary`. It also utilizes `OTP-11388` which
allows for NIFs and BIFs to have more control over when and how they
are garbage collected (we should do some investigation on the
usefulness of this for NIFs like Jiffy).

The 17.0 release also updates `binary_to_term` in `OTP-11535` to
behave properly with reductions and yielding similar to
`term_to_binary`. This marks the 17.0 release as an important one for
CouchDB as now `term_to_binary` and `binary_to_term` both behave
properly.

## Dirty Schedulers

One other interesting item introduced in the 17.0 release is the
concept of dirty schedulers [12] [13]. This is an experimental feature
providing CPU and I/O schedulers specifically for NIFs that are known
to take longer that 1ms to run. In general, we want to make sure the
NIFs we use will yield and report reductions properly, but for
situations where that isn't feasible, we may want to look into using
dirty schedulers down the road when it's a non experimental feature.


## Recommendations for CouchDB

In my opinion we need to take the Erlang release issues more seriously
than we currently do and provide strong recommendations to users on
what versions of Erlang we support. I suggest we loosely take an
approach similar to Debian, and make three recommendations:

  * OldStable: [R14B01, R14B03, R14B04 (NOTE: _not_ R14B02)]
  * Unstable: [R16B03-1 recommended, R16B02 acceptable]
  * Experimental: [17.0]

I'm not suggesting permanently having three Erlang releases
recommended like this, but it currently seems appropriate. I think
long term we should target 17.x as our preferred Erlang release, and
then make a CouchDB 3.0 release that is backwards incompatible with
anything less than 17.0 so that we can switch over to using maps.

The narrowness of the acceptable releases list is going to cause some
problems. Debian Wheezy runs R15B01, which as established above, is
not good to run with unless you have the `+sfwi` patch, and I'm sure
there are many other distros running R15 and R16B or R16B01. I think
it would be useful to users to have a set of packages with a proper
Erlang CouchDB release allowing us to bless specific versions of
Erlang and bundle it together, but I know this idea goes against the
recent change in stance on working with distributions, and I don't
know the ASF stance on this issue well enough to comment on the
legality of it. That said, it does seem like the logical approach
until we get a range of stable releases spread out through the
distros.

## Work to be done

We need to make sure that all NIFs we use that could potentially take
longer than 1ms to run properly yield and report reductions. For
Jiffy, there is already a good start on this work [11]. We'll want to
look into what needs to be done for the rest of the NIFs.


## Wrapping up


There's quite a bit of information here, and plenty more in the
footnotes, so I hope this gives a good overview of the current state
of Erlang releases and helps us to make informed decisions on what
approach to take with Erlang releases.





### Footnotes

[1] [http://comments.gmane.org/gmane.comp.lang.erlang.bugs/3564](http://comments.gmane.org/gmane.comp.lang.erlang.bugs/3564)

[2] [http://erlang.org/pipermail/erlang-questions/2013-April/073490.html](http://erlang.org/pipermail/erlang-questions/2013-April/073490.html)

[3] [http://erlang.org/pipermail/erlang-questions/2012-October/069503.html](http://erlang.org/pipermail/erlang-questions/2012-October/069503.html)

[4] [http://erlang.org/pipermail/erlang-questions/2012-October/069585.html](http://erlang.org/pipermail/erlang-questions/2012-October/069585.html)

[5] [http://permalink.gmane.org/gmane.comp.lang.erlang.bugs/3573](http://permalink.gmane.org/gmane.comp.lang.erlang.bugs/3573)

[6] [http://erlang.org/pipermail/erlang-patches/2013-June/004109.html](http://erlang.org/pipermail/erlang-patches/2013-June/004109.html)

[7] [https://gist.github.com/evanmcc/a599f4c6374338ed672e](https://gist.github.com/evanmcc/a599f4c6374338ed672e)

[8] [http://data.story.lu/2013/06/23/riak-1-3-2-released](http://data.story.lu/2013/06/23/riak-1-3-2-released)

[9] [https://github.com/basho/otp/compare/erlang:maint...OTP_R16B02_basho4](https://github.com/basho/otp/compare/erlang:maint...OTP_R16B02_basho4)

[10] [https://groups.google.com/forum/#!topic/nosql-databases/XpFKVeUBdn0](https://groups.google.com/forum/#!topic/nosql-databases/XpFKVeUBdn0)

[11] [https://github.com/davisp/jiffy/pull/49](https://github.com/davisp/jiffy/pull/49)

[12] [https://github.com/erlang/otp/commit/c1c03ae4ee50e58b7669ea88ec4d29c6b2b67c7b](https://github.com/erlang/otp/commit/c1c03ae4ee50e58b7669ea88ec4d29c6b2b67c7b)

[13] [http://www.erlang.org/doc/man/erl_nif.html#dirty_nifs](http://www.erlang.org/doc/man/erl_nif.html#dirty_nifs)
