---
title: "Get top values by group (animated)"
excerpt: Animated explanation of dplyr::top_n() and code to create the gif.
category: rstats
tags:
  - dplyr
  - gganimate
  - gifs
  - luna
header:
  image: /assets/images/featureAnimate.png
---

Thanks to the 'gganimate' package, expository demos are in season:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Me too! Didn’t see that coming</p>&mdash; Thomas Lin Pedersen (@thomasp85) <a href="https://twitter.com/thomasp85/status/1029586660915326976?ref_src=twsrc%5Etfw">August 15, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

For previous posts, I've made graphical summaries of R functions such as this one below.  

<figure>
    <a href="/assets/images/featureUnbreak.png"><img src="/assets/images/featureUnbreak.png"></a>
        <figcaption></figcaption>
</figure>

Now that I know how to make crude animations of the data-wrangling process, I made this animation of various tidyerse functions piped together. I focused on the _top\_n()_ function in 'dplyr' because I've written about it in the past and the posts still seem to get pretty regular traffic (links below).

![gif demo]({{ site.baseurl }}/assets/images/topndemo.gif)
 
top_n() posts:  
[English version](https://luisdva.github.io/rstats/Top-and-bottom-values-for-groups/)  
[Spanish version](https://luisdva.github.io/Valores-m%C3%A1ximos-y-m%C3%ADnimos-para-datos-agrupados/) 


Say we have a dataset with three dogs, three cats, and three birds. We have names for each one, plus their 'cuteness' rating as a continuous value. The code to create the animation is in the gist at the end of this post (full of hacks but reproducible).

I pasted the code as the plot title, but we can interpret each step like so:

- Mutate to modify character strings, using multiple conditional statements.
- Filter to discard birds.
- Group by the type of animal.
- Get the top value for each group.

These are Luna and Lili, respectively. 

<figure>
    <a href="/assets/images/cutest.jpg"><img src="/assets/images/cutest.jpg"></a>
        <figcaption>cutest</figcaption>
</figure>

peace!  

the code:
{% gist luisDVA/91c99f0df5995e85f725ff134b792c18 %}
