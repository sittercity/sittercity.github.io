## Local setup
* Install jekyll gem: `bundle install`
* You'll need nodejs if you don't already it: `sudo apt-get install nodejs`
* Clone this repo

## Running
To run this locally, use: `bundle exec jekyll serve`

This will run on port 8080. You can access it via `http://localhost:8080`

## To add posts
Simply add new files to the `_posts` directory. As long as you name it the same
way as the other files are named it will show up the next time we deploy.

If you are currently running the jekyll server it will auto-build so your post
is available

```
---
layout: post
title:  "Welcome to Jekyll!"
date:   2015-01-22 03:40:22
categories: jekyll update
---
```

## Process for adding posts

* Create a branch for yourself and add your incredibly insightful post (or posts).
* Create a PR pointing back to master

That's it! We'll all talk about your additions and decide on when it should be merged.
Once it is merged it will be live to the public.
