---
layout: post
title:  "QA Automation on Chime"
date:   2016-03-30 15:00:00
categories: qa
author: "Nick Fagen"
---
![Sittercity QA](/assets/sittercity_qa_logo.png)

So far, 2016 has been a great year. Our small, but rockstar, QA team has been making huge strides towards improving automated regression suites across all of our projects. We have tackled the endless QA dilemma regarding automation, how to do it effectively and efficiently. There are always obstacles when automating any app. For example, let's say you have a new release and bug fixes ready to ship. How will you ensure that the new bug fix(es) have not introduced any new issues in previous working functionality? Manually testing all of the functionality of an application for every single fix or new feature is an option when a project is just starting.  However, as a project grows, large manual test regression plans are daunting and in my opinion a red flag. With larger regression plans you might be able to accomplish it, but then you are not going to be testing effectively in terms of company cost, resources, time, etc. More and more companies are trying to move toward continuous delivery, making a need for QA automation.

<!-- More -->

As Bob V. stated in the [last QA post](http://tech.sittercity.io/articles/2015/06/24/sittercity-quality-assurance), we have made a lot of changes. The last few years the QA department has accomplished a lot. What would take the QA team days to accomplish can now occur daily. In the past, if there were bugs, the release would have to wait until the next week because of the regression feedback loop. Now, for our newest product [Chime](https://hellochime.com) we deliver to prod multiple times a day, up to every 20 minutes for the varying underlying services. It is obviously not just QA's effort that has made this possible though; it has been a team effort across all departments (Development, Devops, and QA). Any company can accomplish continuous delivery if they have a determined team that can work well with one another.

## Deciding what tools to use for Chime

The newest Sittercity product, Chime, gave the QA team the opportunity to really work towards improving automation. There are many automation tools available. The foundational code a site uses will factor into what option is most appropriate. We primarily use ruby but also have a lot of different languages used in various projects. These may not work specifically for you, but they have met our needs so far: Watir, Cucumber, Appium, Applitools, and Jenkins.

### Watir

We chose to use [Watir](http://watir.com/) for the web automation. We have previously used it on our other [Sittercity](https://www.sittercity.com/) automation suites. It is easy to learn and very powerful in the right hands.

### Cucumber
Cucumber (and the Gherkin DSL) has many benefits for us, chief among them being that it is how we strive to structure our acceptance criteria. Another big benefit is that Cucumber works really, really well with Watir.

♥ Cucumber and Watir-WebDriver sitting in a tree ♥  

This combination allows us to automate the testing of our web app and at the same time make the features easily readable to stakeholders, without any layers of abstraction. It's win-win-win for all involved.

### Appium
We have decided to try [Appium](http://www.appium.com) for our automated mobile testing. Appium lets us automate our app as a user would actually use it. Actually registering users and performing actions and on REAL devices. We use Calabash-iOS and Calabash-Android for our other mobile apps. However, we have run into issues with them not being maintained for our needs in certain instances. This was why we decided to take a look at Appium. So far it's working well. We are looking forward to learning more about this tool and what opportunities it can provide for us.

### Applitools

Eyes_selenium is a gem by [Applitools](https://applitools.com/). This allows us to do automated UI testing. This allows us to test pixel and general layout level differences between pages. We take screenshots of the app on the web and on mobile devices in order to compare them against baseline images. Awesome! QA doesn't have to go through the app and scrutinize every page for visual defects that normal automation does't catch. This is a great tool to use to make sure we don’t introduce any unpleasant visual bugs. Applitools works for both  web and mobile automation.

### Jenkins

We utilize Jenkins to its full extent to a QA team. It allows seamless continuous integration. Thanks to Jenkins we can automatically kick off automated regression suites for specific parts of our apps upon being deployed to an environment. QA is not required to start and monitor a build every time a deploy is done, which is awesome. The results of the suites are automatically sent into chat rooms so everyone is aware when releases can continue to go out (or not). This allows QA to focus on more automation and do more exploratory testing of the app (which, in my opinion, is the end goal of all QA teams).

## Automate all the things!

By leveraging these tools together, along with [parallelizing](https://github.com/grosser/parallel_tests) the tests themselves, we have helped the team gain greater confidence in making breaking changes as well as being able to deliver a lot in a short amount of time.

With all of these pieces combined we deliver the following to our company:

* We release to production multiple times a day.
* Run thousands of tests on a daily basis.
* Have drastically reduced regression suite running times.
* We are continuously improving/refactoring the automation suites.
* Make sure to leave time for ping pong

Over the past year we have made many improvements, especially in automation, but know there is still a lot to accomplish. These goals have been reached through hard work and dedication.  Fortunately for me, it hardly feels like work when you enjoy what you do.

“Pleasure in the job puts perfection in the work”. – Aristotle.
