## Local setup
* Install jekyll gem: `gem install jekyll`
* You'll need nodejs if you don't already it: `sudo apt-get install nodejs`
* Clone this repo

## Running
To run this locally, use: `jekyll serve`

This will run on port 8080. You can access it via `http://localhost:8080`

## To add posts
Simply add new files to the `_posts` directory. As long as you name it the same
way as the other files are named it will show up the next time we deploy.

If you are currently running the jekyll server it will auto-build so your post
is available

To enable comments on your post add 'comments: true' to the metadata at the top,
like so:

```
---
layout: post
title:  "Welcome to Jekyll!"
date:   2015-01-22 03:40:22
categories: jekyll update
comments: true
---
```
