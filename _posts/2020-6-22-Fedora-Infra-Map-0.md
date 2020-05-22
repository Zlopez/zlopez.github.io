---
layout: post
title: Fedora Infra Map - First Steps
tags: fedora clean_architecture fedora_infra_map
---

A few weeks ago I started to learn about [clean architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) and wanted to try this approach in Python, I decided to start working on a new app. As a source of working with clean architecture in Python I used book [Clean Architectures in Python](https://leanpub.com/clean-architectures-in-python) by Leonardo Giordani.

## What is Fedora Infra Map?
[Fedora Infra Map](https://github.com/Zlopez/fedora-infra-map) is work in progress application that should allow users to interactively look at the map of Fedora Infrastructure. You should be able to find any application that interests you in the map and see plenty of information about the application, together with any relationship to other applications. 

As a Fedora Infrastructure team we don't have anything like this and it will be a very nice app for us and the users who want to know more. Inside our team we talked about this a plenty of times that it will be nice to have the diagram of your applications or some kind of map. And here it is. The work is in it's beginnings and it will take a lot of time and coding to have something usable, but the work already started.

## Why clean architecture?
The more I read about the clean architecture, the more I want to try it and use it. But why one should chose the clean architecture and why it's called clean? The reason is relatively simple, design is much more cleaner, because interfaces to external systems are separated from core code. This is done by splitting your code to layers, in our case I decided to go with three layers. In core you have domain layer, which contains the simple building stones of the whole application. Then you have use cases layer, which contains internal implementation of use cases for the applications and requests/responses definition to communicate with external systems. The most outer layer are wrappers around external systems. This makes the application easily maintainable and testable. You can also switch the external system without need the to rewrite internal implementation of your application.

I decided to work on fedora-infra-map using the TDD (Test Driven Development) together with clean architecture. As this allows you to have very reliable code from start and to have a really stable core of your application.

To summarize the answer, I chose the clean architecture to have both stable and easily maintainable app and also, because I want to try something new. :-)

## First steps
I started by creating the project using [Cookiecutter](https://github.com/cookiecutter/cookiecutter). This tool is really helpful when you need to start a new project from scratch. I plan to create my own template in future, but for now I decided to use [template from audreyr](https://github.com/audreyr/cookiecutter-pypackage). There were a few caveats, like the Travis webhook, which is not working great on GitHub anymore and it took a few hours to actually show the check on each pull request on GitHub. At the end I just needed to delete the webhook and only use the GitHub Travis app. And also some links were generated wrongly, but I managed to fix those issues. But the setup of the project itself was really quick, even with the setup of Travis and ReadTheDocs build.

Next thing I started to work on was the [list of requirements](https://fedora-infra-map.readthedocs.io/en/latest/requirements.html#list-of-requirements). This is a small list that describes what the applications should do and what features it should support. It is rather simple, good for first clean architecture application. And because of the clean architecture design it should be much more easier to add another use cases in the future.

Based on the requirements I started to work on [design document](https://fedora-infra-map.readthedocs.io/en/latest/design.html). In this document I described every class and source file, so anyone who will work with it should be able to orient in the application easily. It also helps to have some vision of the application before you start working on it.

After I had those two documents I finally started hacking the code.

## Entities implementation
As I wrote above I decided to go with TDD. This means that you write test first and then write code that will make the test pass. To use TDD to it's fullest you shouldn't write more than one test at once. So the work itself is slower then the quick prototyping. But you can be sure that your code works from very beginning. Also one of the benefit of clean architecture is that the internal implementation doesn't need any dependencies at all. Everything is written using only standard python libraries, which makes the code much easier to maintain.

I decided to name the core layer of fedora-infra-map application entities, it contains implementation of two basic classes Edge and Node. These represents the relationships and applications on the map.This layer also contains serializers for those entities. So you can easily serialize them to JSON when working with them outside of the internal layers. This will also make much more easier to implement any API response, which should contain any of those entities.

The work on the Edge and Node implementation is now done. I will continue working on JSON serializers when I will get back to it. The next step after that is to start work on use cases layer.
