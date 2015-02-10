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

## Process for adding posts

* Create a branch for yourself and add your incredibly insightful post (or posts).
* Create a PR pointing back to master

That's it! We'll all talk about your additions and decide on when it should be merged.

Once it's merged to master there is another process to follow before it is served
live to the public but that will be covered in a separate (and as yet nonexistent)
section.

## TODO

* Make this look a little nicer
* Update the index page so it doesn't show all posts but maybe just the last 5 or so
* Create an archive page to house anything not shown on the front page
