---
layout: post
title:  "Implementing Majordomo"
author: "Phil Trimble"
date:   2016-02-15 12:00:00
categories: technology, majordomo
---

Like most technology outfits we have work that we would like to perform 'off-cycle'. This just means that we don't want make a customer wait for a particular task that is not critical for their experience at that moment. This is a very normal problem and there are many solutions out there in the world, so many in fact that it would take forever to list them all.

At Sittercity we have a handful of existing solutions. Our main platform has traditionally been something we call `Stormtrooper`. This is a custom, homegrown library written in `Ruby` and backed by `RabbitMQ`. It works pretty well but we have intermittently experienced issues with lost messages and ballooning memory usage. As we have grown we have come to realize that it just wasn't cutting it anymore. We would need to look for a new solution. 

<!-- More -->

We brainstormed a few high-level requirements that we would need to keep in mind during our search:

* We didn't have the developer bandwidth for a complete rewrite. Any new solution would need to fit into our existing stack seamlessly.
* We could explore new languages but since most of our stack was in Ruby any new solution would need to incorporate that fact.
* We would prefer to use something open-source or something we could write ourselves. Nothing proprietary or requiring some outside contract.

## Starting the process

A few months ago we received new project requirements that would necesitate off-cycle work. Based on the high-level requirements above we began to focus on [ØMQ](http://zeromq.org/) and decided to attempt to implement the [Majordomo protocol](http://rfc.zeromq.org/spec:7).

With `ØMQ` and Majordomo we saw the following positives:

* `ØMQ` is a mature technology
* There are a lot of libraries and guides for reference
* `ØMQ` sockets are simple and lightweight
* We have big plans to scale and Majordomo/`ØMQ` fit the bill
* It could implement both traditional request/reply synchronous messages as well as asynchronous eventing
* Man would it be cool to learn this (we didn't really use this one when pitching it to our bosses, but come on, we were all thinking it)

We also listed out all of the cons that we could see:

* Sockets are freaking hard, man
* The Majordomo protocol (front to back) is inherently multithreaded. That's always difficult, especially in our environment where we strive for 100% test coverage. Testing async stuff can be a major pain in the ass.
* No one at Sittercity had ever done this before. That's scary!

In the end we decided that the major thing holding us back was fear. In our experience it's never a good idea to base your decisions entirely on fear. We decided to go for it and started the process of writing our own in-house Majordomo implementation in `Go` and `Ruby`.

## Wait wait wait, why did you want to write it in-house?

We looked around for existing implementations before deciding to do any work in-house. What we found was that while there was an 'official' Github repo it was unfortunately written primarily in `C`. As we have said on this blog previously we are a `Ruby` and `Go` house so bringing in a `C` codebase wasn't really an option. We just wouldn't be able to properly support it.

We also found a handful of reference implementations written in various languages (_including_ `Go` and `Ruby`). However, there was no supported implementation of the entire protocol out in the wild. The reference codebases were just that: examples to show how Majordomo could work. None of it was production-ready as far as we could tell.

## Implementing Majordomo

The [Majordomo protocol](http://rfc.zeromq.org/spec:7) has three major components:

* Client - this emits an event containing the work request
* Broker - this is the middleman, accepting request from clients and routing them to relevant workers
* Worker - this is the actual thing performing the work (duh)

Here is a nice visual overview of the whole thing (stolen from the RFC):

[<img src="http://zeromq-rfc.wdfiles.com/local--files/spec:7/figure1.png">](http://rfc.zeromq.org/spec:7)

In a nutshell, clients send requests (either synchronously or asynchronously) to a broker, which then routes those requests to available workers. The work is performed by the worker and passed back to the broker, which then passes it back to the original client. Both workers and clients are responsible for reporting their own health to the broker, which tracks all available components.

I won't go into the nuts and bolts of Majordomo itself here. The idea is that requests can be passed based on abstract service names and each set of components needs to know nothing about the implementation details of other components. As long as they all use the right protocol we're good!

In the beginning we had a simple [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call) requirement. This is the basic request/reply model. We were starting from scratch, basically, so where do we begin?

As mentioned before (multiple times) most of our existing stack is written in `Ruby`. We knew that we would need both clients and workers to probably be written in `Ruby`, since we would be submitting requests from a `Ruby` application and probably wanted to run work based on existing business logic also written in `Ruby`. We decided initially to bypass that part and think about the broker. That piece could be reused by multiple applications. We needed to think long and hard about its design.

In the end we decided to write it from scratch in `Go`. Two senior engineers decided to pair and took a week and a half to create an intial implementation of the broker based on the RFC. They also wrote a worker and client in `Ruby` that were specific for the project in question since we were not 100% convinced that this would be our long-term solution (more on that in a bit).

The end result? It all went pretty smooth! We rolled it out and pretty much have no complaints. The broker performs very, very well. We now have fine-grained visibility into our eventing stack and know when and how off-cycle work performs.

## What we learned

During the implementation process we learned a few key facts:

* The performance is AWESOME. No exact benchmarks at this stage but it's running really, really well with a small memory footprint.
* `Go` works really well for stuff like this. It easily replaces `Ruby`.
* We had a LOT to learn about `ØMQ` but it performed really, really well for us.
* We really needed to think long and hard about logging and reporting for this to work so we could monitor the health of all of the individual components. This was more difficult than a straightforward in-process algorithm.
* The ability to write each component in a unique language really helped us integrate slowly into our stack, allowing us to avoid a huge rewrite. Our Majordomo implementation is running side-by-side with our existing applications in Ruby.

But what about the negatives?

* Sockets are HARD, man! Goddamn.
* Our desire to reach 100% test coverage can really slow us down when implmenting asynchronous logic of this nature. It's doable but difficult and something to keep in mind for future work efforts.

## What about those workers and clients in Ruby?

As I mentioned in the beginning we did not want to go whole-hog on this. We wanted to test it out and see how things went. The initial client and workers were both written in `Ruby` inside of existing `Ruby` applications. When a second application required off-cycle work we wanted to use our existing Majordomo broker but couldn't spend the time making the existing client/worker code generic enough to be shared. Rather than take a chance of poor abstractions we copy/pasted the code from the existing workers into our new project. As [Sandy Metz says](http://www.sandimetz.com/blog/2016/1/20/the-wrong-abstraction): `duplication is far cheaper than the wrong abstraction`.

Copy and pasting code obviously isn't a substainable practice, though. We recently received more project requirements that needed off-cycle workers decided that it was time to finally write a standardized worker implementation. We decided to (again) write it in `Go`. This was completed in December and rolled out to production in mid-January and so far has been performing well.

## Hey, you skipped over the client!

Most of our client applications (the pieces emitting the requests for work) are still written in `Ruby` and it is likely to remain that way. Because of that we wanted to keep that code as-is. We do have `Go` services that will need to emit work to our Majordomo broker in the future but we can cross that bridge when we come to it.

Our main `Ruby` eventing library is another homegrown project that we have released called [Arbiter](https://github.com/sittercity/arbiter). That gem is our attempt to create an abstraction that allows the business logic to emit events without worrying about implementation details. It is battle tested and used across our stack.

## Are you going to release this stuff?

Yes! Our broker accepts requests from multiple applications across our production stack and we feel confident in its stability. In addition our new worker is processing requests, although since that piece is much newer we would like to continue to improve it before releasing. 

We plan on combining the two pieces into one repository and then pushing it to our [Github](https://github.com/sittercity/) organization in the coming months.

## Final thoughts

We are very happy internally with how this all turned out. A few thoughts:

* When condemplating a shift like this make sure to choose something that can be integrated slowly. Big bang rewrites are very difficult!
* `Go` once again performed very well for us and we'll continue to use it more and more internally.
* `ØMQ` is solid and great for stuff like this but we cannot stress enough how much effort it is to fully understand it. Socket programming is very difficult compared to other solutions so make sure the gains you are looking for are worth the additional complexity headaches.

Look for future posts about our releases of the broker and worker implementations in `Go` and for future updates to `Arbiter`! Hit us up on [twitter](https://twitter.com/sittercitytech) if you want to chat about our experiences.
