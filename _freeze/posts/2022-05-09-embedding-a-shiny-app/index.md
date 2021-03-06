---
title: How to embed a Shiny app into your blog posts
format: hugo
author: ''
date: '2022-05-09'
slug: []
categories: []
tags: ['shorts', 'shiny']
description: This is a one-minute blog post to share how to incorporate Shiny apps in blog posts.
hideToc: no
enableToc: no
enableTocContent: no
tocFolding: no
tocPosition: inner
tocLevels:
  - h2
  - h3
  - h4
series: ''
image: ''
libraries:
  - mathjax
---



Today's a short blog post.
It's mainly for sharing a cool trick I just learned.

Here's a simple template to incorporate your Shiny app into an HTML file.
For instance, you can incorporate your shiny app into your blog post like I do here.
Simply exchange the src argument by your Shiny app's URL and then you're good to go.
Here, I use the app that I have [shown you a couple of months ago](https://albert-rapp.de/post/2022-01-17-drawing-a-ggplot-interactively/).

    <iframe src="https://rappa.shinyapps.io/interactive-ggplot" data-external="1" width="925px" height="800px">
    </iframe>

From what I could tell, this is same code that `knitr::include_app()` drops.
But including the `iframe` manually let's you adjust the width **and** height of your
frame.
Beware that you will have to choose the dimensions large enough for your Shiny app.

UPDATE: Originally, I had demonstrated the above code chunk here.
But that causes unnecessary traffic on my shinyapps.io account, so I removed the demo.
