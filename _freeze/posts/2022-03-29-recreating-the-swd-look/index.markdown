---
title: 'Recreating the Storytelling with Data look with ggplot'
author: ''
date: '2022-03-29'
slug: []
categories: []
tags: []
description: "We try to imitate the Storytelling with Data look with ggplot"
hideToc: no
enableToc: yes
enableTocContent: no
tocFolding: no
tocPosition: inner
tocLevels:
  - h2
  - h3
  - h4
series: ggplot2-tips
image: ~
libraries:
  - mathjax
---



So, I found a [great video](https://www.youtube.com/watch?v=st7_vPjq0SU) from Storytelling with Data (SWD).
In this video, a data storyteller demonstrates how a dataviz that does not demonstrate a clear story can be improved.
Let's take a look at the dataviz but, first, here's the data.


```r
library(tidyverse)
dat <- tibble(
  id = 1:19,
  fulfilled = c(803, 865, 795, 683, 566, 586, 510, 436, 418, 364, 379, 372, 374, 278, 286, 327, 225, 222, 200),
  accuracy = c(86, 80, 84, 82, 86, 80, 80, 93, 88, 87, 85, 85, 83, 94, 86, 78, 89, 88, 91),
  error = c(10, 14, 10, 14, 10, 16, 15, 6, 11, 7, 12, 13, 8, 4, 12, 12, 7, 10, 7),
  null = 100 - accuracy - error
) %>% 
  mutate(across(accuracy:null, ~. / 100))
dat
## # A tibble: 19 x 5
##       id fulfilled accuracy error  null
##    <int>     <dbl>    <dbl> <dbl> <dbl>
##  1     1       803     0.86  0.1   0.04
##  2     2       865     0.8   0.14  0.06
##  3     3       795     0.84  0.1   0.06
##  4     4       683     0.82  0.14  0.04
##  5     5       566     0.86  0.1   0.04
##  6     6       586     0.8   0.16  0.04
##  7     7       510     0.8   0.15  0.05
##  8     8       436     0.93  0.06  0.01
##  9     9       418     0.88  0.11  0.01
## 10    10       364     0.87  0.07  0.06
## 11    11       379     0.85  0.12  0.03
## 12    12       372     0.85  0.13  0.02
## 13    13       374     0.83  0.08  0.09
## 14    14       278     0.94  0.04  0.02
## 15    15       286     0.86  0.12  0.02
## 16    16       327     0.78  0.12  0.1 
## 17    17       225     0.89  0.07  0.04
## 18    18       222     0.88  0.1   0.02
## 19    19       200     0.91  0.07  0.02
```

This data set contains a lot of accuracy and error rates from different (anonymous) warehouses.
Additionally, there are "null rates". 
These are likely related to data quality issues.
Furthermore, this data set is apparently taken from a client the data storytellers helped.
In any case, here is a `ggplot2` recreation of the client's initial plot.
Note that the plot does not match exactly but it's close enough to get the gist.


```r
theme_set(theme_minimal())
dat_long <- dat %>% 
  pivot_longer(
    cols = accuracy:null,
    names_to = 'type',
    values_to = 'percent'
  )

dat_long %>% 
  ggplot(aes(id, percent, fill = factor(type, levels = c('null', 'accuracy', 'error')))) +
  geom_col() +
  labs(
    title = 'Warehouse Accuracy Rates',
    x = 'Warehouse ID',
    y = '% of total orders',
    fill = element_blank()
  ) +
  scale_y_continuous(labels = ~scales::percent(., accuracy = 1), breaks = seq(0, 1, 0.1))
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-3-1.png" width="672" />

As it is right know, the plot shows data.
But what is the message of this dataviz?
To make the message more explicit, the plot is transformed during the course of the video.
Take a look at what story the exact same data can tell.

<img src="final.png" width="944" />

From reading the SWD book, I know that some of the techniques that were used in this picture can be used in many settings.
Therefore, I decided to document the steps I took to recreate the dataviz with ggplot.

I tried to make this documentation as accessible as possible.
Consequently, if you are already quite familiar with how to customize a ggplot's details, then some of the explanations or references may be superfluous. 
Feel free to skip them.
That being said, let's transform the plot.

## Flip the axes for long names

Although it is not really an issue here, warehouses or other places might be more identifiable by a (long) name rather than an ID.
To make sure that these names are legible, show them on the y-axes.
When I first learned ggplot, there was the layer `coord_flip()` to do that job for us.
Nowadays, though, you can often avoid `coord_flip()` because a lot of geoms already understand what you mean, when you map categorical data to the y-aesthetic.
But make sure that ggplot will know that you mean categorical data (especially if the labels are numerical like here).


```r
categorial_dat <- dat_long %>% 
  mutate(
    id = as.character(id),
  )

categorial_dat %>% 
  ggplot(aes(x = percent, y = id)) +
  geom_col(
    aes(group = factor(type, levels = c('error', 'null', 'accuracy'))),
    col = 'white', # set color to distinguish bars better
  )
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-5-1.png" width="672" />

Notice that I used the `group`- instead of `fill`-aesthetic because I only need grouping. 
Also, it is always a good idea to avoid [excessive use of colors](https://albert-rapp.de/post/2022-02-19-ggplot2-color-tips-from-datawrapper/).
This will allow us to emphasize parts of our story with colors later on.

## Add reference points

Another good idea it to put your data into perspective.
To do so, include a reference point.
This can be a summary statistic like the average error rate.
For more great demonstration of reference points you can also check out [the evolution of a ggplot by C??dric Scherer](https://www.cedricscherer.com/2019/05/17/the-evolution-of-a-ggplot-ep.-1/).


```r
averages <- dat_long %>% 
  group_by(type) %>% 
  summarise(percent = mean(percent)) %>% 
  mutate(id = 'ALL') 

dat_with_summary <- categorial_dat %>% 
  bind_rows(averages)

dat_with_summary %>% 
  ggplot(aes(x = percent, y = id)) +
  geom_col(
    aes(group = factor(type, levels = c('error', 'null', 'accuracy'))),
    col = 'white',
  )
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-6-1.png" width="672" />


## Order your data

To allow your reader to gain a quick overview, put your data into some form of sensible ordering.
This eases the burden of having to make sense of what the visual shows.
Also, notice that we already did part of that.
See, with the order of the levels in the `group` aesthetic, we influenced the ordering of the stacked bars.
Here, we made sure that important quantities start at the left resp. right edges.

Why is that helpful, you ask?
Well, the bars that start on the left all start at the same reference point.
Therefore comparisons are quite easy for these bars.
The same holds true for the right edge.
Consequently, it is best that we reserve these vip seats for the important data.
Check out what happens if I were to put the accuracy in the middle.


```r
dat_with_summary %>% 
  ggplot(aes(x = percent, y = id)) +
  geom_col(
    aes(group = factor(type, levels = c('error', 'accuracy', 'null'))),
    col = 'white',
  )
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-7-1.png" width="672" />

Now, we can't really make out which warehouses have a higher accuracy.
Given that the accuracy is likely something we care about, this is bad.
But we can change the order even more.
For instance, we can also order the bars by error rate.
Here, `fct_reorder()` is our friend.


```r
ordered_dat <- dat_with_summary %>% 
  mutate(
    type = factor(type, levels = c('error', 'null', 'accuracy')),
    id = fct_reorder(id, percent, .desc = T)
  ) 

ordered_dat %>% 
  ggplot(aes(x = percent, y = id)) +
  geom_col(
    aes(group = type),
    col = 'white',
  )
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-8-1.png" width="672" />

## Highlight your story points

Next, it's time to highlight your story points.
This can be done with the `gghighlight` as I have demonstrated [in another blog post](https://albert-rapp.de/post/2022-02-19-ggplot2-color-tips-from-datawrapper/#emphasize-just-one-or-a-few-categories).
Alternatively, we can set the colors manually.
The latter approach gave me the best results in this case, so we'll go with that.
But I am still a big fan of `gghighlight`, so don't discard its power just yet.


```r
# Set colors as variable for easy change later
unhighlighted_color <- 'grey80'
highlighted_color <- '#E69F00'
avg_error <- 'black'
avg_rest <- 'grey40'

# Compute new column with colors of each bar
colored_dat <- ordered_dat %>% 
  mutate(
    custom_colors = case_when(
      id == 'ALL' & type == 'error' ~ avg_error,
      id == 'ALL' ~ avg_rest,
      type == 'error' & percent > 0.1 ~ highlighted_color,
      T ~  unhighlighted_color
    )
  )

p <- colored_dat %>% 
  ggplot(aes(x = percent, y = id)) +
  geom_col(
    aes(group = type),
    col = 'white',
    fill = colored_dat$custom_colors # Set colors manually
  )
p
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-9-1.png" width="672" />

Notice how your eyes are immediately drawn to the intended region.
That's the power of colors!
Also, note that setting the colors manually like this worked because `fill` in `geom_col()` is vectorized.
This is not always the case.
In these instances, you may find that [functional programming solves your problem](https://albert-rapp.de/post/2022-03-25-functional-programming-when-geoms-are-not-vectorized/).

## Remove axes expansion and allow drawing outside of grid

Did you notice that there is still some clutter in the plot?
Removing clutter from a plot is a central element of the SWD look.
Personally, I like this approach. 
So, let's get down to the essentials and remove what does not need to be there.
In this case, there are still (faint) horizontal lines behind each bar.
Furthermore, this causes the warehouse IDs to be slightly removed from the bars.
We change that through formatting the coordinate system with `coord_cartesian()`.


```r
p <- p +
  coord_cartesian(
    xlim = c(0, 1), 
    ylim = c(0.5, 20.5), 
    expand = F, # removes white spaces at edge of plot
    clip = 'off' # allows drawing outside of panel
  )
p
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-10-1.png" width="672" />

Here, we turned off the expansion to avoid wasting white space.
Now, the IDs are at their designated place and we do not see lines from their names to the bars anymore.
If you want even more power on the space expansion you can leave `expand = T` and modify the expansion for each axis with `scale_*_continuous()` and the `expansion()` function.
Check out [Christian Burkhart's neat cheatsheet](https://twitter.com/ChBurkhart/status/1492087527511052290/photo/1) that teaches you everything you need to understand expansions.

On an unrelated note, you may wonder why I set `clip = 'off'`.
This little secret will be revealed soon. 
For now, just know that it allows you to draw geoms outside the regular panel.

## Move and format axes

You may have noticed that the x-axis in the finished plot is at the top of the panel rather than at the bottom.
While that is unusual, it helps the reader to get straight to the point as the data is in view earlier.
This assumes that the eyes of a typical dataviz reader will first look at the top left corner and then zigzag downwards.

In `ggplot2`, moving the axes and setting the break points happens in a scale layer.
It is here where we use the `scales::percent()` function to transform the axes labels.
Additionally, changing labels happens in `labs()` and the remaining axes and text changes happen in `theme()`.



```r
unhighlighed_col_darker <- 'grey60'
p <- p +
  scale_x_continuous(
    breaks = seq(0, 1, 0.2),
    labels = scales::percent,
    position = 'top'
  ) +
  labs(
    title = 'Accuracy rates for highest volume warehouses',
    y = 'WAREHOUSE ID',
    x = '% OF TOTAL ORDERS FULFILLED',
  ) +
  theme(
    axis.line.x = element_line(colour = unhighlighed_col_darker),
    axis.ticks.x = element_line(colour = unhighlighed_col_darker),
    axis.text = element_text(colour = unhighlighed_col_darker),
    text = element_text(colour = unhighlighed_col_darker),
    plot.title = element_text(colour = 'black')
  )
p
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-11-1.png" width="672" />

Notice that we have customized the theme elements via `element_*()` functions.
Basically, each geom type like "line", "rect", "text", etc. has their own `element_*()` function.
The `theme()` function expects attributes to be changed using these.
If you are unfamiliar with this concept, maybe the corresponding part in my [YARDS lecture notes](https://yards.albert-rapp.de/statquant#themes) will help you.

## Align labels

Aligning plot elements, e.g. labels, to form clean lines is another major aspect of the SWD look.
Before I read about it, I did not even notice it but once you see it you cannot go back.
Basically, plots feel "more harmonious" if there are clear (not necessarily drawn) lines like with the left and right edge of the stacked bars.
But this concept does not stop with the bars and can be used for the labels too.
Let's demonstrate that by moving the labels with more of `theme()`.


```r
p <- p +
  theme(
    axis.title.x = element_text(hjust = 0),
    axis.title.y = element_text(hjust = 1),
    plot.title.position = 'plot'
    # aligns the title to the whole plot and not the (inner) panel
  )
p
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-12-1.png" width="672" />

Once again, the design enforces that important information like what's on an axis is in the top left corner.
This was done by changing `hjust`. 
In this case `hjust = 0` corresponds to left-justified whereas `hjust = 1` corresponds to right-justified.
Of course, `vjust` works similarly.
For more details w.r.t. `hjust` and `vjust`, check out [this stackoverflow answer](https://stackoverflow.com/questions/7263849/what-do-hjust-and-vjust-do-when-making-a-plot-using-ggplot) that gives you everything that you need in one visual.
For your convenience, here is a slightly changed form of that visual.

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-13-1.png" width="672" />


But once you start aligning the axes titles, you notice that the 0% and 100% labels fall outside the grid.
We could try to set `hjust` of `axis.text.x` in `theme()` but sadly this is not vectorized.
Subsequently, all `hjust` values must be the same.
That's not bueno.
Therefore, I drew the axes labels manually with `annotate()` but make sure that you remove the current labels in `scale_x_continuous()`.
Also, now you know why we had to set `clip = 'off'` earlier.
The axes labels are outside of the regular panel.


```r
p <- p +
  # Overwriting previous scale will generate a warning but that's ok
  scale_x_continuous(
    breaks = seq(0, 1, 0.2), # We still want the axes ticks
    labels = rep('', 6), # Empty strings as labels
    position = 'top'
  ) +
  annotate(
    'text',
    x = seq(0, 1, 0.2),
    y = 20.75,
    label = scales::percent(seq(0, 1, 0.2), accuracy = 1),
    size = 3,
    hjust = c(0, rep(0.5, 4), 1),
    # individual hjust here
    vjust = 0, 
    col = unhighlighed_col_darker
  ) +
  theme(
    axis.title.x = element_text(hjust = 0, vjust = 0)
    # change vjust to avoid overplotting
  )
## Scale for 'x' is already present. Adding another scale for 'x', which will
## replace the existing scale.
p
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-14-1.png" width="672" />

## Add text labels

The same trick can be used to add the category description (accuracy, null, error) to the right top corner and label the highlighted bars.
For the latter part, we simply extract the corresponding rows from our data and use that in conjunction with `geom_text()`.


```r
text_labels <- colored_dat %>% 
  filter(type == 'error', percent > 0.1) %>% 
  mutate(percent = scales::percent(percent, accuracy = 1))

p <- p +
  geom_text(
    data = text_labels, 
    aes(x = 1, label = percent), 
    hjust = 1.1,
    col = 'white',
    size = 4
  )
p
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-15-1.png" width="672" />

Notice that I used a `hjust` value greater than 1 here to add some white space on the right side of the labels.
Otherwise, the percent sign will be too close to the bar's edge.

Next, we add the category descriptions.
This is a bit more tricky, though, because we want to highlight a word too,
So, we will add a `richtext` as described [in my previous blog post](https://albert-rapp.de/post/2022-02-19-ggplot2-color-tips-from-datawrapper/#label-directly).


```r
library(ggtext)
p <- p +
  annotate(
    'richtext',
    x = 1,
    y = 21.25,
    label = "ACCURATE | NULL | <span style = 'color:#E69F00'>ERROR</span>",
    hjust = 1,
    vjust = 0, 
    col = unhighlighed_col_darker, 
    size = 4,
    label.colour = NA,
    fill = NA
  )
p
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-16-1.png" width="672" />



## Add story text

Now that the bar plot is finished we can work on the story text.
For that, we create another plot that contains only the text.
Later on, we will combine both of our plots with the `patchwork` package.
There are no really knew techniques here, so let's get straight to the code.


```r
# Save text data in a tibble
tib_summary_text <- tibble(
  x = 0, 
  y = c(1.65, 0.5), 
  label = c("<span style = 'color:grey60'>OVERALL:</span> **The error rate is 10% across all<br>66 warehouses**. <span style = 'color:grey60'>The good news is that<br>the accuracy rate is 85% so we\'re hitting<br>the mark in nearly all our centers due to<br>the quality initiatives implemented last year.</span>",
  "<span style = 'color:#E69F00'>OPPORTUNITY TO IMPROVE:</span> <span style = 'color:grey60'>10 centers<br>have higher than average error rates of<br>10%-16%.</span> <span style = 'color:#E69F00'>We recommend investigating<br>specific details and **scheduling meetings<br>with operations managers to<br>determine what's driving this.**</span>"
  )
)

# Create text plot with geom_richtext() and theme_void()
text_plot <- tib_summary_text %>% 
  ggplot() +
  geom_richtext(
    aes(x, y, label = label),
    size = 3,
    hjust = 0,
    vjust = 0,
    label.colour = NA
  ) +
  coord_cartesian(xlim = c(0, 1), ylim = c(0, 2), clip = 'off') +
  # clip = 'off' is important for putting it together later.
  theme_void()
text_plot
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-17-1.png" width="672" />


## Add main message as new title and subtitle

As I said before, we will put the two plots together with `patchwork`.
If you have never dealt with `patchwork`, feel free to check out my short [intro to patchwork](https://albert-rapp.de/post/2021-10-28-extend-plot-variety/).
Putting the plots together gives us another opportunity: 
We can now set additional titles and subtitles of the **whole** plot.
Use these to add the main message of your plot.

But make sure that there is enough white space around them by setting the title margins in `theme()`.
Otherwise, your plot will feel "too full".
Adding spacing is achieved through a `margin()` function in `element_text()`.
Though, in this case we use `element_markdown()` which works exactly the same but enables [Markdown syntax](https://www.markdownguide.org/basic-syntax/) like using asterisks for bold texts.


```r
# Save texts as variables for better code legibility
# Here I used Markdown syntax
# To enable its rendering, use element_markdown() in theme
title_text <- "**Action needed:** 10 warehouses have <span style = 'color:#E69F00'>high error rates</span>"
subtitle_text <- "<span style = 'color:#E69F00'>DISCUSS:</span> what are <span style = 'color:#E69F00'>**next steps to improve errors**</span> at highest volume warehouses?<br><span style = 'font-size:10pt;color:grey60'>The subset of centers shown (19 out of 66) have the highest volume of orders fulfilled</span>"
caption_text <- "SOURCE: ProTip Dashboard as of Q4/2021. See file xxx for additional context on remaining 47 warehouses<br><span style = 'font-size:6pt;color:grey60'>Original: Storytelling with Data - improve this graph! exercise | {ggplot2} remake by Albert Rapp (@rappa753)."

# Compose plot
library(patchwork)
p +
  text_plot +
  # Make text plot narrower
  plot_layout(widths = c(0.6, 0.4)) +
  # Add main message via title and subtitle
  plot_annotation(
    title = title_text,
    subtitle = subtitle_text,
    caption = caption_text,
    theme = theme(
      plot.title = element_markdown(
        margin = margin(b = 0.4, unit = 'cm'),
        # 0.4cm margin at bottom of title
        size = 16
      ),
      plot.subtitle = element_markdown(
        margin = margin(b = 0.4, unit = 'cm'),
        # 0.4cm margin at bottom of title
        size = 11.5
      ),
      plot.caption.position = 'plot',
      plot.caption = element_markdown(
        hjust = 0, 
        size = 7, 
        colour = unhighlighed_col_darker, 
        lineheight = 1.25
      ),
      plot.background = element_rect(fill = 'white', colour = NA)
      # This is only a trick to make sure that background really is white
      # Otherwise, some browsers or photo apps will apply a dark mode
    )
  ) 
```

<img src="final.png" width="944" />


## Get the sizes right

In the last plot, I cheated.
I gave you the correct code I used to generate the picture.
But I did not execute it.
Instead, I only displayed the code and then showed you the (imported) picture from the start of this blog post.
Why did I do this?
Because getting the sizes right sucks!

If you have dealt with ggplot enough, then you will know that text sizes are often set in absolute rather than in relative terms.
Therefore, if you make the bar plot smaller in width (like we did), then the bars may be appropriately scaled to the new width but, more often than not, the texts are not.
In this case, this led to way too large fonts as beautifully demonstrated in [Christophe Nicault's helpful blog post](https://www.christophenicault.com/post/understand_size_dimension_ggplot2/).

So, how do you avoid this?
First off, choose size and [fonts](https://albert-rapp.de/post/2022-03-04-fonts-and-icons/) last (choose the font first, though). 
This will save you a lot of repetitive work when you change the alignment in your plot.
But this tip will only get you so far, because you have to fix some sizes in between to get a feeling for the visualization you are trying to create.

Therefore, try to get you canvas into an appropriate size first.
I try to do this by using the `camcorder` package at the start of my visualization process.
This will ensure that my plots are saved as a png-file with predetermined dimensions and the resulting file is displayed in the Viewer pane in RStudio (as opposed to the Plots pane).

For example, at the start of working on this visualization I have called

```r
camcorder::gg_record(
  dir = 'img', dpi = 300, width = 16, height = 9, units = 'cm'
)
```

This made getting the sizes right for my final output somewhat easier because the canvas size remains the same throughout the process.
Though be sure to call `gg_record()` **after** `library(ggtext)` or make sure that you call `gg_record()` again if you add `ggtext` only later.
Otherwise, your plots will revert back to being displayed in the Plots pane (with relative sizing).
Finally, if you want to use `camcorder` in conjunction with `showtext`, then be sure that `showtext` will know what dpi value you chose when calling `gg_record()`.

```r
showtext::showtext_opts(dpi = 300)
```

Alright, that concludes this somewhat long blog post. I hope that you enjoyed it and learned something valuable.
If you did, feel free to leave a comment.
Also, you can stay in touch with my work by subscribing to my [RSS feed](https://albert-rapp.de/index.xml) or following me on [Twitter](https://twitter.com/rappa753).
