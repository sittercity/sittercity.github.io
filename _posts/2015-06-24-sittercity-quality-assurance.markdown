---
layout: post
title:  "Sittercity Quality Assurance"
date:   2015-06-24 11:00:00
categories: qa
author: "Bob Vonderau"
---
![Sittercity QA](/assets/sittercity_qa_logo.png)

Over the past several years, the QA team at Sittercity has gone through a drastic transformation.  We have moved from a completely manual testing process, to a nearly fully automated testing platform that has helped streamline our product delivery pipeline.

This post will take you through the journey of the QA team throughout the past few years, and outline where we plan to go in the future.

<!-- More -->

## Past

As of 2010, nearly all of the testing done by the QA team at Sittercity was done manually.  We had a few unreliable [Selenium IDE](http://www.seleniumhq.org/projects/ide/) scripts, that only help test a few scenarios.  This was not ideal, as manually testing all new features as well as manually regression testing before each release became very time consuming and could lead to delays in releasing if any last minute issues were found.  

This left the QA team in the following state:

  - All manual testing.
  - Very slow QA process, regression testing took several days for each release.
  - Minimal system/testing documentation.

## Present

In 2012, when Sittercity began a project of re-platforming from PHP to Ruby on Rails, the QA team took this as an opportunity to completely revamp the QA process from the ground up.  The goals of this change included:

  - Begin to introduce automated testing into our regression testing process.
  - Improve documentation.
  - Streamline the QA process to allow for quicker/safer and more reliable releases.

Once the re-platform was complete in 2013, we ended up exceeding all of those goals.  

As of today, the QA testing process includes:

  - Nearly fully automated regression testing.
  - Regression testing time is reduced to a few hours.  Allowing frequent/reliable releases.
  - Streamlined QA process to deliver quality products on time.
  - Cross device/cross platform automated tests.
  - Extensive system/testing documentation.
  - In house mobile testing lab
  - Drastic decrease in bugs reported by users - automated testing was able to catch a majority of regression issues before they reached users.

### Automated Testing Stats:
  - Over 1000 automated test cases.
  - Over 10000 automated testing steps.
  - In 2014 over 100,000 test cases were executed.

### How do we accomplish this?
We are able to accomplish all of this with a variety of tools and processes.

![Sittercity QA](/assets/qa_tools.jpg)

 - [Cucumber](https://cukes.info/) has been an excellent acceptance criteria framework used across our product and development teams to describe desired functionality.  These are easily translated into unit and full stack acceptance tests that the development and QA team can build and maintain.
 - [Calabash](http://calaba.sh/) and [Appium](appium.io) are used for our automated mobile testing.  They can be used across iOS and Android platforms.  They are tightly integrated with Cucumber.
 - [WatirWebdriver](http://watirwebdriver.com/) is the engine behind our full stack Cucumber tests.  WatirWebdriver allows our tests to run in actual browsers, locally or using cloud based virtual machines.
 - [Applitools](https://applitools.com/) is a new tool in our testing arsenal.  This allows us to do machine based UI testing, allowing us to test pixel level differences between pages.  QA engineers no longer have to manually inspect each page, on every release.

### Typical Day:

Having all of these processes and tools at our disposal allows the QA team members to be exposed to lots of different projects, technologies and opportunities each day.  

A typical day can be broken down into the following:

![Daily Routine](/assets/sittercity_qa_daily_routine.jpg)

## Future

Just as the Sittercity Tech Team uses cutting edge technologies, the QA team must keep pace, ensuring all of Sittercity's systems are fully operational, production ready and as bug free as possible.  

As we continue to grow and adapt, we look forward to the following:

 - Continuously improving/refactoring automated tests.
 - Using new testing platforms, tools and technologies.
 - Leveraging intelligent/machine learning testing frameworks and tools.
 - Additional cross platform/cross devices testing.
 - Decreased regression testing time.
 - Mobile device lab growth.

 As we increase our level of automated test coverage, utilize intelligent automation tools/machine learning capabilities and increasing our testing capabilities across mobile devices and platforms, the future of QA at Sittercity is what I like to call the: "Rise of the Machines"

 ![Sittercity QA](/assets/terminator_three_rise_of_the_machines_ver2.jpg)
