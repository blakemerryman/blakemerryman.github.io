---
layout: post
title:  Youtube View Counter
date:   2014-12-23 16:52:31
description: Explanation behind the Youtube View Counter Breaking
---

Youtube recently ran into issues with it's view counter for the Gangnum Style video. My friend Allen asked me why Google didn't already use the 64-bit version rather than 32-bit version. The answer: bandwidth.

## Some Stats & Facts:

- In [January 2012](http://youtube-global.blogspot.com/2012/01/holy-nyans-60-hours-per-minute-and-4.html), Google had over 4 billion views per day. It's definitely more now, but this is the most up-to-date stat I could find.
- Every time you load a Youtube video, you are downloading that view counter (along with the video and everything else on the page).
- 32-bit integers take up 4 Bytes; 64-bit integers take up 8 Bytes.

## Now For Some Math:

{% highlight %}
oldCounterBandwidth = ( 4 Bytes / view ) * ( 4 billion views / day ) = 16 GB / day *
newCounterBandwidth = ( 8 Bytes / view ) * ( 4 billion views / day ) = 32 GB / day *
{% endhighlight %}

** Note: Final values were converted from Bytes to GigaBytes for clarity.*

By converting to 64-bit integers to store the view counters, they are doubling their bandwidth from 16 GB per day to 32 GB per day.... **just to load the view counter**. Keep in mind that the daily view stat I found is from January 2012.