---
format: hugo
title: "How to collect dataviz from Twitter into your note-taking system"
author: ''
date: '2022-04-14'
slug: []
categories: []
tags: ['API', 'automatization']
description: "The goal is to send a tweet's URL to a dummy mail account and let R extract it, call the Twitter API and then put everything as a Markdown file into your note-taking system."
hideToc: no
enableToc: yes
enableTocContent: no
tocFolding: no
tocPosition: inner
tocLevels:
  - h2
  - h3
  - h4
series: ''
libraries:
  - mathjax
---



## Intro

It is mid-April and the
[\#30daychartchallenge](https://twitter.com/30DayChartChall) is well on
its way. One glace at the hashtag's Twitter feed suffices to realize
that there are great contributions. That's a perfect opportunity to
collect data viz examples for future inspirations.

Ideally, I can scroll through Twitter and with a few clicks incorporate
these contributions straight into my [Obsidian](https://obsidian.md/) or
any other Markdown-based note-taking system. Unfortunately, `rtweet`'s
snapshot function does not seem to work anymore. So, let's build
something on our own that gets the job done. The full script can be
found on [GitHub
gist](https://gist.github.com/AlbertRapp/37a2e0993acea9b4e36400037b797391).
Here's what we will need:

-   Twitter app bearer token (to access Twitter's API) - I'll show you
    how to get that
-   Elevated API access (just a few clicks once you have a bearer token)
-   Dummy mail account to send tweets to

## Overview

Before we begin, let me summarize what kind of note-taking process I
have in mind:

1.  Stroll through Twitter and see great data viz on twitter.

2.  Send tweet link and a few comments via mail to a dummy mail account

3.  A scheduled process accesses the dummy mail account and scans for
    new mails from authorized senders.

4.  If there is a new mail, R extracts tweet URL and uses Twitter's API
    to download the tweet's pictures and texts.

5.  A template Markdown file is used to create a new note that contains
    the images and texts.

6.  Markdown file is copied to your note-taking system within your file
    system.

7.  Ideally, your Markdown template contains tags like \#dataviz and
    \#twitter so that your new note can be easily searched for.

8.  Next time you look for inspiration, stroll through your collections
    or search for comments.

## Preparations

Ok, we know what we want to accomplish. Time to get the prelims done.
First, we will need a Twitter developer account. Then, we have to mask
sensitive information in our code. If you already have a twitter app
resp. a bearer token and know the `keyring` package, feel free to skip
this section.

### Get Twitter developer account

Let's create a developer account for Twitter. Unfortunately, there is no
way to get such an account without providing Twitter with your phone
number. Sadly, if this burden on your privacy is a problem for you, then
you cannot proceed. Otherwise, create an account at
[developer.twitter.com](https://developer.twitter.com/en).

In your developer portal, create a project. Within this project create
an app. Along the way, you will get a bunch of keys, secrets, IDs and
tokens. You will see them only once, so you will have to save them
somewhere. I suggest saving them into a password manager like
[bitwarden](https://bitwarden.com/).

When you create your app or shortly after, you will need to set the
authentication settings. I use `OAuth 2.0`. This requires

-   type of app: `Automated bot or app`
-   Callback URI / Redirect URI: `http://127.0.0.1:1410` (DISCLAIMER:
    This is magic to me but the `rtweet` docs - or possibly some other
    doc (not entirely sure anymore)- taught me to set up an app that
    way)
-   Website URL: Your Twitter link (in my case
    `https://twitter.com/rappa753`)

Next, you will likely need to upgrade your project to 'elevated' status.
This can be done for free on your project's dashboard. From what I
recall, you will have to fill out a form and tell Twitter what you want
to do with your app. Just be honest and chances are that your request
will immediately be granted. Just be yourself! What could possibly go
wrong? Go get the ~~girl~~ elevated status (ahhh, what a perfect
opportunity for a [Taylor
song](https://www.youtube.com/watch?v=6FQ11gCO64o)).

<figure>
<img src="project-elevated.PNG" width="669"
alt="Click on detailed features to apply for higher access" />
<figcaption aria-hidden="true">Click on detailed features to apply for
higher access</figcaption>
</figure>

### How to embed your bearer token and other sensitive material in your code

Use the `keyring` package to first save secrets via `key_set` and then
extract them in your session via `key_get()`. This way, you won't share
your sensitive information by accident when you share your code (like I
do). In this post, I do this for my bearer token, my dummy mail, my
dummy mail's password and for the allowed senders (that will be the mail
where the tweets come from).

``` r
bearer_token <- keyring::key_get('twitter-bearer-token')
user_mail <- keyring::key_get('dataviz-mail')
password_mail <- keyring::key_get('dataviz-mail-password')
allowed_senders <- keyring::key_get('allowed_senders')
```

The `allowed_senders` limitation is a precaution so that we do not
accidentally download some malicious spam mail from God knows who onto
our computer. I am no security expert but this feels like a prudent
thing to do. If one of you fellow readers knows more about this security
business, feel kindly invited to reach out to me with better security
strategies.

## What to do once we have a URL

Let's assume for the sake of this section that we already extracted a
tweet URL from a mail. Here's the URL that we will use. In fact, it's
[Christian Gebhard](https://twitter.com/c_gebhard)'s tweet that inspired
me to start this project. From the URL we can extract the tweet's ID
(the bunch of numbers after `/status/`). Also, we will need the URL of
Twitter's API.

``` r
library(stringr) # for regex matching
library(dplyr) # for binding rows and pipe
tweet_url <- 'https://twitter.com/c_gebhard/status/1510533315262042112'
tweet_id <- tweet_url %>% str_match("status/([0-9]+)") %>% .[, 2]
API_url <- 'https://api.twitter.com/2/tweets'
```

### Use GET() to access Twitter API

Next, we use the `GET()` function from the `httr` package to interact
with Twitter's API.

``` r
library(httr) # for API communication

auth <- paste("Bearer", bearer_token) # API needs format "Bearer <my_token>"

# Make request to API
request <- GET(
  API_url, 
  add_headers(Authorization = auth), 
  query = list(
    ids = tweet_id, 
    tweet.fields = 'created_at', # time stamp
    expansions = 'attachments.media_keys,author_id', 
    # necessary expansion fields for img_url
    media.fields = 'url' # img_url
  )
) 
request
## Response [https://api.twitter.com/2/tweets?ids=1510533315262042112&tweet.fields=created_at&expansions=attachments.media_keys%2Cauthor_id&media.fields=url]
##   Date: 2022-04-14 07:32
##   Status: 200
##   Content-Type: application/json; charset=utf-8
##   Size: 690 B
```

So, how do we know how to use the `GET()` function? Well, I am no expert
on APIs but let me try to explain how I came up with the arguments I
used here.

Remember those toys you would play with as a toddler where you try to
get a square through a square-shaped hole, a triangle through a
triangle-shaped hole and so on? You don't? Well, neither do I. Who
remembers that stuff from very early childhood?

But I hear that starting a sentence with "Remember those..." is good for
building a rapport with your audience. So, great! Now that we feel all
cozy and connected, I can tell you how I managed to get the API request
to work.

And the truth is actually not that far from the toddler ["intelligence
test"](https://www.youtube.com/watch?v=NNl7GQFTULU). First, I took a
look at a [help
page](https://developer.twitter.com/en/docs/twitter-api/data-dictionary/object-model/media)
from Twitter's developer page. Then, I hammered at the `GET()` function
until its output contained a URL that looks similar to the example I
found. Here's the example code I was aiming at.

    curl --request GET 'https://api.twitter.com/2/tweets?ids=1263145271946551300&
    expansions=attachments.media_keys&
    media.fields=duration_ms,height,media_key,preview_image_url,public_metrics,type,url,width,alt_text' 
    --header 'Authorization: Bearer $BEARER_TOKEN'

This is not really R code but it looks like usually you have to feed a
GET request with a really long URL. In fact, it looks like the URL needs
to contain everything you want to extract from the API. Specifically,
the structure of said URL looks like

-   the API's base URL (in this case https://api.twitter.com/2/tweets)
-   a question mark `?`
-   pairs of `keywords` (like `ids`) and a specific value,
    e.g.??`ids=1263145271946551300`, that are connected via `&`

Therefore, it is only a matter of figuring out how to make the output of
`GET()` deliver this result. Hints on that came from `GET()` examples in
the docs.

``` r
GET("http://google.com/", path = "search", query = list(q = "ham"))
## Response [http://www.google.com/search?q=ham]
##   Date: 2022-04-14 07:32
##   Status: 200
##   Content-Type: text/html; charset=ISO-8859-1
##   Size: 98.2 kB
## <!doctype html><html lang="de"><head><meta charset="UTF-8"><meta content="/im...
## document.documentElement.addEventListener("submit",function(b){var a;if(a=b.t...
## var a=window.performance;window.start=Date.now();a:{var b=window;if(a){var c=...
## var f=this||self;var g,h=null!=(g=f.mei)?g:1,m,n=null!=(m=f.sdo)?m:!0,p=0,q,r...
## e);var l=a.fileName;l&&(b+="&script="+c(l),e&&l===window.location.href&&(e=do...
## var c=[],e=0;window.ping=function(b){-1==b.indexOf("&zx")&&(b+="&zx="+Date.no...
## var k=this||self,l=function(a){var b=typeof a;return"object"==b&&null!=a||"fu...
## b}).join(" "))};function w(){var a=k.navigator;return a&&(a=a.userAgent)?a:""...
## !1}e||(d=null)}}else"mouseover"==b?d=a.fromElement:"mouseout"==b&&(d=a.toElem...
## window.screen&&window.screen.width<=c&&window.screen.height<=c&&document.getE...
## ...
GET("http://httpbin.org/get", add_headers(a = 1, b = 2))
## Response [http://httpbin.org/get]
##   Date: 2022-04-14 07:32
##   Status: 200
##   Content-Type: application/json
##   Size: 397 B
## {
##   "args": {}, 
##   "headers": {
##     "A": "1", 
##     "Accept": "application/json, text/xml, application/xml, */*", 
##     "Accept-Encoding": "deflate, gzip", 
##     "B": "2", 
##     "Host": "httpbin.org", 
##     "User-Agent": "libcurl/7.64.1 r-curl/4.3.2 httr/1.4.2", 
##     "X-Amzn-Trace-Id": "Root=1-6257ce20-7902b3b55883ec5d3c0a1389"
## ...
```

So, the first example shows how an argument `query` can be filled with a
list that creates the URL we need. The second examples shows us that
there is something called `add_headers()`. Do I know exactly what that
is? I mean, from a technical perspective of what is going on behind the
scenes? Definitely not. But Twitter's example request had something
called header. Therefore, `add_headers()` is probably something that
does what the Twitter API expects.

And where do the key-value pairs come from? I found these strolling
through Twitter's [data
dictionary](https://developer.twitter.com/en/docs/twitter-api/data-dictionary/introduction).
Thus, a `GET()` request was born and I could feel like a true
[Hackerman](https://knowyourmeme.com/memes/hackerman).

``` r
auth <- paste("Bearer", bearer_token) # API needs format "Bearer <my_token>"

# Make request to API and parse to list
request <- GET(
  API_url, 
  add_headers(Authorization = auth), 
  query = list(
    ids = tweet_id, 
    tweet.fields = 'created_at', # time stamp
    expansions = 'attachments.media_keys,author_id', 
    # necessary expansion fields for img_url
    media.fields = 'url' # img_url
  )
) 
```

Alright, we successfully requested data. Now, it becomes time to parse
it to something useful. The `content()` function will to that.

``` r
parsed_request <- request %>% content('parsed')
parsed_request
## $data
## $data[[1]]
## $data[[1]]$author_id
## [1] "1070306701"
## 
## $data[[1]]$created_at
## [1] "2022-04-03T08:23:01.000Z"
## 
## $data[[1]]$text
## [1] "#30DayChartChallenge #Day3 - Topic: historical\n\nBack to the Shakespeare data! Hamlet is is longest play, the comedies tend to be shorter.\n\nTool: #rstats\nData: kaggle users LiamLarsen, aodhan\nColor-Scale: MetBrewer\nFonts: Niconne, Noto Sans (+Mono)\nCode: https://t.co/iXAbniQDCb https://t.co/JCNrYH9uP4"
## 
## $data[[1]]$attachments
## $data[[1]]$attachments$media_keys
## $data[[1]]$attachments$media_keys[[1]]
## [1] "3_1510533145334104067"
## 
## 
## 
## $data[[1]]$id
## [1] "1510533315262042112"
## 
## 
## 
## $includes
## $includes$media
## $includes$media[[1]]
## $includes$media[[1]]$media_key
## [1] "3_1510533145334104067"
## 
## $includes$media[[1]]$type
## [1] "photo"
## 
## $includes$media[[1]]$url
## [1] "https://pbs.twimg.com/media/FPZ95H0XwAMHA8q.jpg"
## 
## 
## 
## $includes$users
## $includes$users[[1]]
## $includes$users[[1]]$id
## [1] "1070306701"
## 
## $includes$users[[1]]$name
## [1] "Christian Gebhard"
## 
## $includes$users[[1]]$username
## [1] "c_gebhard"
```

### Extract tweet data from what the API gives us and download images

We have seen that `parsed_request` is basically a large list that
contains everything we requested from the API. Unfortunately, it is a
highly nested list, so we have to do some work to extract the parts we
actually want. `pluck()` from the `purrr` package is our best friend on
this one. Here's all the information we extract from the
`parsed_request`.

``` r
library(purrr) # for pluck and map functions
# Extract necessary information from list-like structure
tweet_text <- parsed_request %>% 
  pluck("data", 1, 'text') 
tweet_text
## [1] "#30DayChartChallenge #Day3 - Topic: historical\n\nBack to the Shakespeare data! Hamlet is is longest play, the comedies tend to be shorter.\n\nTool: #rstats\nData: kaggle users LiamLarsen, aodhan\nColor-Scale: MetBrewer\nFonts: Niconne, Noto Sans (+Mono)\nCode: https://t.co/iXAbniQDCb https://t.co/JCNrYH9uP4"

tweet_user <-  parsed_request %>% 
  pluck("includes", 'users', 1, 'username')
tweet_user
## [1] "c_gebhard"

# We will use the tweet date and time as part of unique file names
# Replace white spaces and colons (:) for proper file names
tweet_date <- parsed_request %>% 
  pluck("data", 1, 'created_at') %>% 
  lubridate::as_datetime() %>% 
  str_replace(' ', '_') %>% 
  str_replace_all(':', '')
tweet_date
## [1] "2022-04-03_082301"

img_urls <- parsed_request %>% 
  pluck("includes", 'media') %>% 
  bind_rows() %>% # bind_rows for multiple pictures, i.e. multiple URLS
  filter(type == 'photo') %>% 
  pull(url)
img_urls
## [1] "https://pbs.twimg.com/media/FPZ95H0XwAMHA8q.jpg"
```

Next, download all the images via the `img_urls` and `download.file()`.
We will use `walk2()` to download all files (in case there are multiple
images/URLs) and save the files into PNGs that are named using the
unique `tweet_date` IDs. Remember to set `mode = 'wb'` in
`download.file()`. I am not really sure why but without it you will save
poor quality images.

``` r
# Download image - set mode otherwise download is blurry
img_names <- paste('tweet', tweet_user, tweet_date, seq_along(img_urls), sep = "_")
walk2(img_urls, img_names, ~download.file(.x, paste0(.y, '.png'), mode = 'wb'))
```

So let's do a quick recap of what we have done so far. We

-   Assembled an API request
-   Parsed the return of the request
-   Cherrypicked the information that we want from the resulting list
-   Used the image URLs to download and save the files to our working
    directory.

Let's cherish this mile stone with a dedicated function.

``` r
request_twitter_data <- function(tweet_url, bearer_token) {
  # Extract tweet id by regex
  tweet_id <- tweet_url %>% str_match("status/([0-9]+)") %>% .[, 2]
  auth <- paste("Bearer", bearer_token) # API needs format "Bearer <my_token>"
  API_url <- 'https://api.twitter.com/2/tweets'
  
  # Make request to API and parse to list
  parsed_request <- GET(
    API_url, 
    add_headers(Authorization = auth), 
    query = list(
      ids = tweet_id, 
      tweet.fields = 'created_at', # time stamp
      expansions='attachments.media_keys,author_id', 
      # necessary expansion fields for img_url
      media.fields = 'url' # img_url
    )
  ) %>% content('parsed')
  
  # Extract necassary information from list-like structure
  tweet_text <- parsed_request %>% 
    pluck("data", 1, 'text') 
  
  tweet_user <-  parsed_request %>% 
    pluck("includes", 'users', 1, 'username')
  
  # Make file name unique through time-date combination
  # Replace white spaces and colons (:) for proper file names
  tweet_date <- parsed_request %>% 
    pluck("data", 1, 'created_at') %>% 
    lubridate::as_datetime() %>% 
    str_replace(' ', '_') %>% 
    str_replace_all(':', '')
  
  img_urls <- parsed_request %>% 
    pluck("includes", 'media') %>% 
    bind_rows() %>% 
    filter(type == 'photo') %>% 
    pull(url)
  
  # Download image - set mode otherwise download is blurry
  img_names <- paste('tweet', tweet_user, tweet_date, seq_along(img_urls), sep = "_")
  walk2(img_urls, img_names, ~download.file(.x, paste0(.y, '.png'), mode = 'wb'))
  
  # Return list with information
  list(
    url = tweet_url,
    text = tweet_text,
    user = tweet_user,
    file_names = paste0(img_names, '.png'),
    file_paths = paste0(getwd(), '/', img_names, '.png')
  )
}
```

### Fill out Markdown template using extracted information and images

We have our images and the original tweet now. Thanks to our previous
function, we can save all of the information in a list.

``` r
request <- request_twitter_data(tweet_url, bearer_token)
```

So, let's bring all that information into a Markdown file. Here is the
`template.md` file that I have created for this joyous occasion.

``` r
library(readr) # for reading and writing files from/to disk
cat(read_file('template.md'))
## #dataviz #twitter
## 
## ![[insert_img_name_here]]
## 
## ### Original Tweet
## 
## insert_text_here
## 
## Original: insert_URL_here
## 
## ### Original Mail
## 
## insert_mail_here
```

As you can see, I started the Markdown template with two tags `#dataviz`
and `#twitter`. This helps me to search for a specific dataviz faster.
Also, I have already written out the Markdown syntax for image imports
`![[...]]` and added a placeholder `insert_img_name_here`. This one will
be replaced by the file path to the image. Similarly, other placeholders
like `insert_text_here` and `insert_mail_here` allow me to save the
tweet and the mail content into my note taking system too.

To do so, I will need a function that replaces all the placeholders.
First, I created a helper function that changes the image import
placeholder properly, when there are multiple images.

``` r
md_import_strings <- function(file_names) {
  paste0('![[', file_names, ']]', collapse = '\n') 
}
```

Then, I created a function that takes the `request` list that we got
from calling our own `request_twitter_data()` function and iteratively
uses `str_replace_all()`. This iteration is done with `reduce2()` which
will replace all placeholders in `template.md` .

``` r
library(tibble) # for easier readable tribble creation
# Replace the placeholders in the template
# We change original mail place holder later on
replace_template_placeholder <- function(template_name, request) {
  # Create a dictionary for what to replace in template
  replace_dict <- tribble(
    ~template, ~replacement,
    '\\!\\[\\[insert_img_name_here\\]\\]', md_import_strings(request$file_names),
    'insert_text_here', request$text %>% str_replace_all('#', '(#)'),
    'insert_URL_here', request$url
  )
  
  # Iteratively apply str_replace_all and keep only final result
  reduce2(
    .x = replace_dict$template, 
    .y = replace_dict$replacement,
    .f = str_replace_all,
    .init =  read_lines(template_name) 
  ) %>% 
    paste0(collapse = '\n') # Collaps lines into a single string
}

replace_template_placeholder('template.md', request) %>% cat()
## #dataviz #twitter
## 
## ![[tweet_c_gebhard_2022-04-03_082301_1.png]]
## 
## ### Original Tweet
## 
## (#)30DayChartChallenge (#)Day3 - Topic: historical
## 
## Back to the Shakespeare data! Hamlet is is longest play, the comedies tend to be shorter.
## 
## Tool: (#)rstats
## Data: kaggle users LiamLarsen, aodhan
## Color-Scale: MetBrewer
## Fonts: Niconne, Noto Sans (+Mono)
## Code: https://t.co/iXAbniQDCb https://t.co/JCNrYH9uP4
## 
## Original: https://twitter.com/c_gebhard/status/1510533315262042112
## 
## ### Original Mail
## 
## insert_mail_here
```

As you can see, my `replace_template_placeholder()` function also
replaces the typical `#` from Twitter with `(#)`. This is just a
precaution to avoid wrong interpretation of these lines as headings in
Markdown. Also, the original mail has not been inserted yet because we
have no mail yet. But soooon. Finally, we need to write the replaced
strings to a file. I got some helpers for that right here.

``` r
write_replaced_text <- function(replaced_text, request) {
  file_name <- request$file_name[1] %>% str_replace('_1.png', '.md')
  write_lines(replaced_text, file_name)
  paste0(getwd(), '/', file_name) 
}
replaced_template <- replace_template_placeholder('template.md', request) %>%
  write_replaced_text(request)
```

### Shuffle files around on your file system

Awesome! We created new image files and a new Markdown note in our
working directory. Now, we have to move them to our Obsidian vault. This
is the place where I collect all my Markdown notes for use in Obsidian.
In my case, I will need to move the Markdown note to the vault directory
and the images to a subdirectory within this vault. This is because I
changed settings in Obsidian that makes sure that all attachments,
e.g.??images, are saved in a separate subdirectory.

Here's the function I created to get that job done. The function uses
the `request` list again because it contains the file paths of the
images. Here, `vault_location` and `attachments_dir` are the file paths
to my Obsidian vault.

``` r
library(tidyr) # for unnesting
move_files <- function(request, replaced_template, vault_location, attachments_dir) {
  # Create from-to dictionary with file paths in each column
  move_dict <- tribble(
    ~from, ~to,
    request$file_path, paste0(vault_location, '/', attachments_dir),
    replaced_template, vault_location
  ) %>% 
    unnest(cols = 'from')
  # Copy files from current working directory to destination
  move_dict %>% pwalk(file.copy, overwrite = T)
  # Delete files in current working directory
  walk(move_dict$from, file.remove)
}
```

## How to extract URL and other stuff from mail

Let's take a quick breather and recap. We have written functions that

-   take a tweet URL
-   hussle the Twitter API to give us all its data
-   download the images and tweet text
-   save everything to a new Markdown note based on a template
-   can move the note plus images to the location of our note-taking hub

Not to brag but that is kind of cool. But let's not rest here. We still
have to get some work done. After all, we want our workflow to be
email-based. So, let's access our mails using R. Then, we can extract a
Twitter URL and apply our previous functions. Also, this lets us finally
replace the `insert_mail_here` placeholder in our Markdown note.

### Postman gives you access

I have created a dummy mail account at gmail. Using the `mRpostman`
package, we can establish a connection to our mail inbox. After the
connection is established, we can filter for all new emails that are
sent from our list of `allowed_senders`.

``` r
library(mRpostman) # for email communication
imap_mail <- 'imaps://imap.gmail.com' # mail client
# Establish connection to imap server
con <- configure_imap(
  url = imap_mail,
  user = user_mail,
  password = password_mail
)

# Switch to Inbox
con$select_folder('Inbox') 
## 
## ::mRpostman: "Inbox" selected.

# Extract mails that are from the list of allowed senders
mails <- allowed_senders %>% 
  map(~con$search_string(expr = ., where = 'FROM')) %>% 
  unlist() %>% 
  na.omit() %>% # Remove NAs if no mail from a sender
  as.numeric() # avoids attributes
```

### Grab URLs from mail

If `mails` is not empty, i.e.??if there are new mails, then we need to
extract the tweet URLs from them. Unfortunately, depending on where you
sent your email from, the mail text can be encoded.

For example, I send most of the tweets via the share button on Twitter
using my Android smartphone. And for some reason, my Android mail client
encodes the mails in something called `base64`. But sending a tweet URL
from Thunderbird on my computer works without any encoding. Here are two
example mails I have sent to my dummy mail account.

``` r
if (!is_empty(mails)) mail_bodys <- mails %>% con$fetch_text()
cat(mail_bodys[[1]])
## ----_com.samsung.android.email_9939060766922910
## Content-Type: text/plain; charset=utf-8
## Content-Transfer-Encoding: base64
## 
## aHR0cHM6Ly90d2l0dGVyLmNvbS9tYXJpbmExMDE3MjAxMy9zdGF0dXMvMTUxNDI4MDkzNjA5NTI0
## MDE5ND9zPTIwJnQ9TFp5TzI3aWhjZk5kbUFPTHlxbTByUUhlcmUgaXMgdGhlIHNhbWUgdHdlZXQg
## YnV0IHNlbnQgZnJvbSBteSBBbmRyb2lkIHBob25lLg==
## 
## ----_com.samsung.android.email_9939060766922910
## Content-Type: text/html; charset=utf-8
## Content-Transfer-Encoding: base64
## 
## PGh0bWw+PGhlYWQ+PG1ldGEgaHR0cC1lcXVpdj0iQ29udGVudC1UeXBlIiBjb250ZW50PSJ0ZXh0
## L2h0bWw7IGNoYXJzZXQ9VVRGLTgiPjwvaGVhZD48Ym9keSBkaXI9ImF1dG8iPmh0dHBzOi8vdHdp
## dHRlci5jb20vbWFyaW5hMTAxNzIwMTMvc3RhdHVzLzE1MTQyODA5MzYwOTUyNDAxOTQ/cz0yMCZh
## bXA7dD1MWnlPMjdpaGNmTmRtQU9MeXFtMHJRPGRpdiBkaXI9ImF1dG8iPjxicj48L2Rpdj48ZGl2
## IGRpcj0iYXV0byI+SGVyZSBpcyB0aGUgc2FtZSB0d2VldCBidXQgc2VudCBmcm9tIG15IEFuZHJv
## aWQgcGhvbmUuPC9kaXY+PC9ib2R5PjwvaHRtbD4=
## 
## ----_com.samsung.android.email_9939060766922910--
## 
cat(mail_bodys[[2]])
## Here is the first data viz that popped up on my Twitter feed.
## 
## https://twitter.com/marina10172013/status/1514280936095240194?s=20&t=1_WWgRGzcDEVmOpRMK6E8w
## 
## This mail was sent from my computer using Thunderbird.
## 
## Ahoy! (Just a random fun thing I wanted to write)
## 
```

As you can see, the mail sent from my computer is legible but the other
one is gibberish. Thankfully, Allan Cameron helped me out on
[Stackoverflow](https://stackoverflow.com/questions/71772972/translate-encoding-of-android-mail-in-r)
to decode the mail. To decode the mail, the trick was to extract the
parts between `base64` and `----`.

There are two such texts in the encoded mail. Surprisingly, the first
one decoded to a text without line breaks. This is why we take the
second encoded part and decode it. However, this will give us an HTML
text with all kinds of tags like `<div>` and what not. Therefore, we use
`html_read()` and `html_text2()` from the `rvest` package to handle
that. All of this is summarized in this helper function.

``` r
decode_encoded_mails <- function(encoded_mails) {
  # Ressource: https://stackoverflow.com/questions/71772972/translate-encoding-of-android-mail-in-r
  # Find location in each encoded string where actual text starts
  start_encoded <- encoded_mails %>% 
    str_locate_all('base64\r\n\r\n') %>% 
    map(~pluck(., 4) + 1) %>% 
    unlist()
  
  # Find location in each encoded string where actual text starts
  end_encoded <- encoded_mails %>% 
    str_locate_all('----') %>% 
    map(~pluck(., 3) - 1)%>% 
    unlist()
  
  # Use str_sub() to extract encoded text
  encoded_text <- tibble(
    string = unlist(encoded_mails), 
    start = start_encoded, 
    end = end_encoded
  ) %>% 
    pmap(str_sub) 
  
  # Decode: base64 -> raw -> char -> html -> text
  encoded_text %>% 
    map(base64enc::base64decode) %>% 
    map(rawToChar) %>% 
    map(rvest::read_html) %>% 
    map(rvest::html_text2)
}
```

I feel like this is the most hacky part of this blog post.
Unfortunately, your milage may vary here. If your phone or whatever you
use encodes the mails differently, then you may have to adjust the
function. But I hope that I have explained enough details and concepts
for you to manage that if it comes to this.

Recall that I send both plain mails from Thunderbird and encoded mails
from Android. Therefore, here is another helper that decoded mails if
neccessary from both types in one swoop.

``` r
decode_all_mails <- function(mail_bodys) {
  # Decode in case mail is base64 decoded
  is_encoded <- str_detect(mail_bodys, 'Content-Transfer-Encoding')
  encoded_mails <- mail_bodys[is_encoded]
  plain_mails <- mail_bodys[!is_encoded]
  decoded_mails <- encoded_mails %>% decode_encoded_mails()
  c(decoded_mails, plain_mails)
}
```

The remaining part of the code should be familiar:

-   Use `decode_all_mails()` for decoding
-   Grab URLs with `str_extract()`
-   Use `request_twitter_data()` with our URLs
-   Replace placeholders with `replace_template_placeholder()`
-   This time, replace mail placeholders too with another
    `str_replace()` iteration
-   Move files with `move_files()`

The only new thing is that we use our postman connection to move the
processed mails into a new directory (which I called "Processed") on the
email server. This way, the inbox is empty again or filled only with
mails from unauthorized senders.

``` r
if (!is_empty(mails)) {
  # Grab mail texts and URLs
  mail_bodys <- mails %>% con$fetch_text() %>% decode_all_mails
  urls <- mail_bodys %>% str_extract('https.*')
  
  # Remove mails from vector in case s.th. goes wrong 
  # and urls cannot be detected
  mail_bodys <- mail_bodys[!is.na(urls)]
  mails <- mails[!is.na(urls)]
  urls <- urls[!is.na(urls)]
  
  # For each url request twitter data
  requests <- map(urls, request_twitter_data, bearer_token = bearer_token)
  
  # Use requested twitter data to insert texts into Markdown template 
  # and write to current working directory
  replaced_templates_wo_mails <- 
    map(requests, replace_template_placeholder, template = 'template.md') 
  
  # Now that we have mails, replace that placeholder too
  replaced_templates <- replaced_templates_wo_mails %>% 
    map2(mail_bodys, ~str_replace(.x, 'insert_mail_here' ,.y)) %>% 
    map2(requests, ~write_replaced_text(.x, .y))
  
  # Move markdown files and extracted pngs to correct place on HDD
  walk2(
    requests, 
    replaced_templates, 
    move_files, 
    vault_location = vault_location, 
    attachments_dir = attachments_dir
  )
  
  # Move emails on imap server to Processed directory
  con$move_msg(mails, to_folder = 'Processed')
}
```

## Last Step: Execute R script automatically

Alright, alright, alright. We made it. We have successfully

-   extracted URLs from mails,
-   created new notes and
-   moved them to their designated place

The only thing that is left to do is execute this script automatically.
Again, if you don't want to assemble the R script yourself using the
code chunks in this blog post, check out this [GitHub
gist](https://gist.github.com/AlbertRapp/37a2e0993acea9b4e36400037b797391).

On Windows, you can write a VBS script that will execute the R script.
Window's task scheduler is [easily set
up](https://www.windowscentral.com/how-create-automated-task-using-task-scheduler-windows-10)
to run that VBS script regularly, say every hour. For completeness' sake
let me give you an example VBS script. But beware that I have no frikkin
clue how VBS scripts work beyond this simple call.

    Set wshshell = WScript.CreateObject ("wscript.shell")
    wshshell.run """C:\Program Files\R\R-4.0.5\bin\Rscript.exe"" ""D:\Local R Projects\Playground\TwitterTracking\my_twitter_script.R""", 6, True
    set wshshell = nothing

The idea of this script is to call `Rscript.exe` and give it the
location of the R script that we want to execute. Of course, you will
need to adjust the paths to your file system. Notice that there are
super many double quotes in this script. This is somewhat dumb but it's
the only way I could find to make file paths with white spaces work (see
[StackOverflow](https://stackoverflow.com/questions/14360599/vbs-with-space-in-file-path)).

On Ubuntu (and probably other Unix-based systems), I am sure that every
Unix user knows that there is
[CronTab](https://stackoverflow.com/questions/38778732/schedule-a-rscript-crontab-everyminute)
to schedule regular tasks. On Mac, I am sure there is something. But
instead of wandering even further from my expertise, I will refer to
your internet search skills.

## Mind the possibilities

We made it! We connected to Twitter's API and our dummy email to get
data viz (what's the plural here? viz, vizz, vizzes, vizzeses?) into our
note-taking system. Honestly, I think that was quite an endeavor. But
now we can use the same ideas for all kind of other applications! From
the top of my head I can think of more scenarios where similar solutions
should be manageable. Here are two ideas.

-   Take notes on the fly using emails and automatically incorporate the
    emails into your note-taking system.

-   Take a photo from a book/text you're reading and send it to another
    dummy mail. Run a script that puts the photo and the mail directly
    into your vault.

So, enjoy the possibilities! If you liked this blog post, then consider
following me on [Twitter](https://twitter.com/rappa753) and/or
subscribing to my [RSS feed](https://albert-rapp.de/index.xml). Until
next time!
