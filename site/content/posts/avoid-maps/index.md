---
title: Another Reason to Avoid Maps
description: They tend to make tests flaky.
date: 2020-04-13
tldr: Maps are generally avoided in performance critical code. They often pop up in orchestration parts. One reason to avoid them there as well is testability. The fact that iteration order is not guaranteed tends to result in flaky tests.
tags: ["production", "data structures"]
---

don't leak errors. use them to communicate to user
