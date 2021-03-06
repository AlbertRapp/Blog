---
title: An Exploratory Introduction to the Plotly Package
author: ''
date: '2021-10-16'
slug: []
categories: []
tags: ["visualization", "exploratory-intro"]
description: "We try to do a few simple things with the plotly package in order to figure out how it works."
hideToc: no
enableToc: yes
enableTocContent: no
tocFolding: no
tocPosition: inner
tocLevels:
  - h2
  - h3
  - h4
series: ~
image: ~
libraries:
  - mathjax
---



Currently, I am quite curious about interactive plots which is why I am reading [Scott Murray's highly recommendable book on D3](https://alignedleft.com/work/d3-book-2e).
For those of you who don't know it,
[D3.js](https://d3js.org/) is a JavaScript library that is great for creating amazing interactive **D**ata-**D**riven-**D**ocuments on the web.

Unfortunately, compared to `ggplot2`, D3's learning curve feels quite steep and I am not yet able to work with it yet.
Fortunately, there are other interactive graphing libraries out there that I can use to get a first feel for creating interactive plots.
For instance, there is another JavaScript library [plotly.js](https://plotly.com/graphing-libraries/), which is built on top of D3 and can be easily used in R through the `plotly` package.

Therefore, I decided to play around with this R package in the hope of figuring out how it works.
Also, I summarized what I played around with in this blog post by writing what I like to call an "exploratory introduction".

As the name implies, this is not really a formal introduction to `plotly` and more of an experience report.
Nevertheless, I suspect that this can be useful for people who have already knowledge about `ggplot2` and want to get started with `plotly` as well.

Further, in case you do not want to read my informal commentary, you can also check out the [video version](https://youtu.be/rzpbQ93pmPM) of this blog post.
Think of it as a summary of this blog post.

As of now, I plan on doing a similar exploratory introduction to [ `r2d3`](https://rstudio.github.io/r2d3/) which is an R interface to D3.
Possibly, this will then be combined with knowledge I gathered from Scott Murray's book.

## One Side Note before We Dive in

According to [Wikipedia](https://en.wikipedia.org/wiki/JavaScript), JavaScript "is one of the core technologies of the World Wide Web".
Surprisingly, JavaScript's ubiquity on the interwebs also messed with this blog's theme template such that font formatting was completely destroyed when I included an interactive `plotly` plot.

To work around this issue, I had to save the plots in an auxillary html-file and include it as a separate frame.
This is why you will find that, here, all plots are saved in a variable (for later export) even if it does not really make sense to save it for plotting purposes.

Finally, be aware that, for some reason, interactive `plotly` plots are quite large such that it might take some time to load them all.

## From ggplot2 to plotly

Let us begin by transforming a ggplot to a plotly plot.
Using a built-in function from the `plotly` package, it is straight-forward to convert a ggplot.


```r
library(tidyverse)
library(plotly)
p <- mpg %>% 
  ggplot(aes(hwy, cty, fill = class)) +
  geom_jitter(shape = 21, size = 2, alpha = 0.5)

plotly_p <- ggplotly(p) 
plotly_p 
```


<iframe src="first_plotly.html" width="100%" height="400px"></iframe>

Marvel at what one can do here:
* Get additional information by hovering your cursor over a point
* Filter classes by clicking on the corresponding legend items
* Click and draw a rectangle with your cursor to zoom into the plot 

But there is one thing that bothers me.
If I move the cursor across the plot window, then the plotly options bar at the top of the window overlaps the "class" legend label.
Can I fix this by moving the legend to the bottom?


```r
p <- p + theme(legend.position = "bottom")
ggplotly(p)
```

<iframe src="trial_legend_bottom.html" width="100%" height="400px"></iframe>

Hmm, it appears that the legend cannot be moved by this.
Do other changes in `theme()` get registered?
Let's check by applying a theme.


```r
p <- p + theme_light()
ggplotly(p) 
```

<iframe src="trial_theme_light.html" width="100%" height="400px"></iframe>

Apparently, at least some changes will be conducted.
Let's see, if I can use `layout()` as shown in the [getting started documentation of plotly](https://plotly.com/ggplot2/getting-started/#cutomizing-the-layout) to move the legend.

The R documentation of the `layout()` function is somewhat minimalistic.
Basically, it refers to the [online plotly reference manual](https://plotly.com/r/reference/#Layout_and_layout_style_objects).
Using the table of content on that website, one can easily find the [layout options](https://plotly.com/r/reference/layout/#layout-showlegend).

I believe the options `xanchor` and `yanchor` are exactly what I need as both of them can be set to options like `left`, `bottom`, etc.
Though, I wonder why the value `auto` was not changed by `ggplotly()` as the initial plot `p` contains `legend.position = "bottom"`.

In any case, I am not a hundred percent sure what kind of syntax `layout()` requires.
An example in the R documentaion would have been nice.
Well, let's try the straight-forward approach then.


```r
p %>% 
  ggplotly() %>% 
  layout(xanchor = "center")
## Warning: 'layout' objects don't have these attributes: 'xanchor'
## Valid attributes include:
## 'font', 'title', 'uniformtext', 'autosize', 'width', 'height', 'margin', 'computed', 'paper_bgcolor', 'plot_bgcolor', 'separators', 'hidesources', 'showlegend', 'colorway', 'datarevision', 'uirevision', 'editrevision', 'selectionrevision', 'template', 'modebar', 'newshape', 'activeshape', 'meta', 'transition', '_deprecated', 'clickmode', 'dragmode', 'hovermode', 'hoverdistance', 'spikedistance', 'hoverlabel', 'selectdirection', 'grid', 'calendar', 'xaxis', 'yaxis', 'ternary', 'scene', 'geo', 'mapbox', 'polar', 'radialaxis', 'angularaxis', 'direction', 'orientation', 'editType', 'legend', 'annotations', 'shapes', 'images', 'updatemenus', 'sliders', 'colorscale', 'coloraxis', 'metasrc', 'barmode', 'bargap', 'mapType'
```

Well, that didn't go as expected, but the warning displays valid attributes and `legend` is among them.
Upon closer inspection, I also realize that, according to the reference manual, `xanchor` and `yanchor` have parent `layout.legend`.
So, I guess, I will have to use this somehow.
Maybe, pass a list to `legend` via `layout()`?


```r
p_layout <- p %>% 
  ggplotly() %>% 
  layout(legend = list(xanchor = "center")) 
p_layout
```

<iframe src="layout_xanchor.html" width="100%" height="400px"></iframe>

Aha!
At least this did not crash again but the resulting plot does not look as I would expect it to.
Let's see what happens when we change `yanchor` as well and then we'll go from there.


```r
p_layout <- p %>% 
  ggplotly() %>% 
  layout(legend = list(
    xanchor = "center",
    yanchor = "bottom"
  )) 
p_layout
```

<iframe src="layout_x_yanchor.html" width="100%" height="400px"></iframe>

Unsurprisingly, this did not help at all.
In retrospect, I don't know how I could think that changing `yanchor` as well would magically cure things.
Again, referring back to the manual (I should really read the descriptions instead of relying on the possible values), it appears that `xanchor` only sets a reference points for the option `x`.
Same thing for `yanchor` and `y`.

So, how about changing `x` and `y` instead?
Possible values for both options range from -2 to 3, so my best guess is that ranges from 0 to 1 refer to the window the points are plotted in.
Consequently, moving the legend to the bottom could be as easy as using a positive `x` value and negative `y` value which are both close to zero.


```r
p_layout <- p %>% 
  ggplotly() %>% 
  layout(legend = list(x = 0.1, y = -0.1))
p_layout
```

<iframe src="layout_xy.html" width="100%" height="400px"></iframe>

Now, this brings us closer to what I had in mind when I set `legend.position = "bottom"` in `theme()`.
Possibly, we can change the orientation of the legend from vertical to horizontal, tweak the `x` and `y` values a bit and then we're there.
Scrolling through the manual (again), reveals that there is an option `orientation` which can be set to `h`.
This sounds promising.


```r
p_layout <- p %>% 
  ggplotly() %>% 
  layout(legend = list(
    x = 0.1, 
    y = -0.2, 
    orientation = "h"
  )) 
p_layout
```

<iframe src="layout_xy_orientation.html" width="100%" height="400px"></iframe>

Nice!
Finally, I am satisfied.
Out of curiosity, let us investigate what had happened, if we had set orientation to `"h"` from the start.


```r
p_layout <- p %>% 
  ggplotly() %>% 
  layout(legend = list(orientation = "h")) 
p_layout
```

<iframe src="layout_orientation.html" width="100%" height="400px"></iframe>

This already looks nice enough, so our manual tweaking was not technically necessary.
But then again, this does not recreate what `legend.position = "bottom"` usually does.
Now that we understand how `layout()` works, we can roam the reference manual and try to tweak the legend box.
Let's try to change a couple of things.
This does [not have to be pretty](https://www.allisonhorst.com/post/do-your-worst/), we only want to see how plotly works.


```r
p_layout <- p %>% 
  ggplotly() %>% 
  layout(
    legend = list(
      orientation = "h",
      borderwidth = 3,
      bgcolor = "grey",
      bordercolor = "red",
      font = list(
        color = "white",
        family = "Gravitas One",
        size = 15
      ),
      title = list(
        text = "Class",
        side = "top",
        font = list(
          color = "white",
          family = "Gravitas One",
          size = 15
        )
      )
    )
  ) 
p_layout
```

<iframe src="layout_more_legend_stuff.html" width="100%" height="400px"></iframe>

Note how we have used lists in lists (in lists) to customize the legend.
Interestingly, our initial blunder of ignoring the "parent" resp. the hierarchy of options earlier helped to understand that as an option's parent's name gets longer, e.g. `layout.legend.title.font`, we will have to use more convoluted lists to change that option. 

## Creating a Plotly Chart Manually

Since we have learned how to tweak a plotly object, we might as well figure out how to create one without having to use `ggplotly()`.
It is not that I want to avoid using ggplot altogether but, in principle, it cannot hurt if we can understand plotly's implementation in the R package `plotly`.

So, from the book [Interactive web-based data visualization with R, plotly, and shiny](https://plotly-r.com/index.html) I gather that the `plotly` R package implements the JavaScript `plotly.js` library via a Grammar of Graphics approach.
Thus, it works similar to `ggplot2` in the sense that we can add layers of graphical objects to create a plot.
In `plotly`'s case, we pass a plotly object from one `add_*` layer to the next (via `%>%`).


```r
plt <- mpg %>% 
  plot_ly() %>% 
  add_markers(x = ~hwy, y = ~cty, color = ~class)
plt
```

<iframe src="manual_plotly.html" width="100%" height="400px"></iframe>

Notice the `~`.
These are used to ensure that the variables are mapped from the data (similar to `aes()` in `ggplot2`).
Alternatively, one could also use `x = mpg$hwy` to create the same plot.

Because we can see a lot of overplotting, let us jitter the points.
Unfortunately, I couldn't find a built-in option for that.
Therefore, let's do the jittering manually.


```r
set.seed(123)
jitter_hwy <- 2
jitter_cty <- 1
jittered_mpg <- mpg %>% 
  mutate(
    hwy = hwy + runif(length(hwy), -jitter_hwy, jitter_hwy),
    cty = cty + runif(length(cty), -jitter_cty, jitter_cty)
  )
plt <- jittered_mpg %>% 
  plot_ly() %>% 
  add_markers(x = ~hwy, y = ~cty, color = ~class)
plt 
```

<iframe src="manual_jittering.html" width="100%" height="400px"></iframe>

Regarding the customization of the points aka markers, we can pass a list of options (taken from the reference manual again) to the `marker` argument in `add_markers()`.


```r
set.seed(123)
plt <- jittered_mpg %>% 
  plot_ly() %>% 
  add_markers(
    x = ~hwy, 
    y = ~cty, 
    color = ~class,
    marker = list(
      size = 8,
      opacity = 0.6,
      line = list(color = "black", width = 2)
    )
  ) 
plt
```

<iframe src="manual_jittering_custom.html" width="100%" height="400px"></iframe>

Alternatively, and what I find surprisingly convenient, we can leave our initial plotly object as it is and pass it to the `style()` function.
This functions works just like the `layout()` function we have seen before.


```r
plt <- jittered_mpg %>% 
  plot_ly() %>% 
  add_markers(
    x = ~hwy, 
    y = ~cty, 
    color = ~class
  ) %>% 
  style(
    marker = list(
      size = 8,
      opacity = 0.6,
      line = list(color = "black", width = 2)
    )
  ) 
plt 
```

Similarly, we could pass this along to `layout()` if we want to customize the legend box again.


```r
plt_layout <- plt %>% 
  layout(
    legend = list(
      orientation = "h",
      borderwidth = 3,
      bgcolor = "grey",
      bordercolor = "red",
      font = list(
        color = "white",
        family = "Gravitas One",
        size = 15
      ),
      title = list(
        text = "Class",
        side = "top",
        font = list(
          color = "white",
          family = "Gravitas One",
          size = 15
        )
      )
    )
  )
plt_layout
```

<iframe src="style_layout_custom.html" width="100%" height="400px"></iframe>

Interesting!
For some obscure reason the exact same legend command behaves differently now.
Honestly, I have no clue what is going on here.
If I had to hazard a guess, I would say that during the conversion of a ggplot object to a plotly object via `ggplotly()` some default values were implemented that cause the change but this is just a hunch.
Possibly, this is connected to `xanchor` or `yanchor`.

In any case, we got a glimpse of how the `plotly` R package works which uses the JavaScript library `plotly.js` to create interactive plots.
Also, we have learned how to convert a ggplot2 object to a plotly object and how we can customize this further.

Interestingly, when converting from ggplot2 to plotly, the pop-up window that appears when you hover over a point is already customized compared to the default from `plot_ly()`.
Did you notice the difference already?

So, in order to end our "exploratory introduction" let us adjust the `hovertemplate` according to the description in the reference manual.
Here, we will use `\n` for line breaks.


```r
plt <- plt %>% 
  style(hovertemplate = "hwy: %{x:.2f}\ncty: %{y:.2f}") %>% 
  layout(legend = list(orientation = "h", y = -0.2)) 
plt
```

<iframe src="custom_tooltip.html" width="100%" height="400px"></iframe>

Notice how the class labels appear outside of the box.
The reference manual refers to this position as "secondary" box.
To get rid of this, we simply add `<extra></extra>` to our hover template.
Unfortunately, it appears as if what is not displayed in the primary box cannot be used as part of the hover template.

Thus, we cannot use `%{color}`.
Instead, we simply map `class` to the `text` attribute of the markers as well.
Then, we can use `%{text}`.


```r
plt <- jittered_mpg %>% 
  plot_ly() %>% 
  add_markers(x = ~hwy, y = ~cty, color = ~class, text = ~class) %>% 
  style(
    marker = list(
      size = 8,
      opacity = 0.6,
      line = list(color = "black", width = 2)
    )
  ) %>% 
  style(hovertemplate = "hwy: %{x:.2f}\ncty: %{y:.2f}\nclass: %{text} <extra></extra>") %>% 
  layout(legend = list(orientation = "h", y = -0.2))
plt
```

<iframe src="custom_tooltip_final.html" width="100%" height="400px"></iframe>

All right! 
That's enough exciting plotting action for today.
Hope you enjoyed this blog post and see you next time.
