---
title: Testing Numerical Applications
description: It's hard.
date: 2018-01-05
draft: true
tldr: Floating point arithmetic, optimizations with convergence criteria, platform implementations.
tags: ["software engineering"]
---

Here I share some of my experience in redesigning a continuous integration system for a large numerical application and the challenges that go with these systems. In particular, CI principles, approaches to handle computationally expensive and long running regression tests, issues that go along with parallelized, non-liner systems, iterative solutions, floating point errors, result tolerances, performance tests and so on.

## Take-home points (TL;DR for hipsters)
- All design decisions derive from one principle: **Dev-CI parity**. Developers and CI run the same commands, in identical environments.
- CI systems should be dumb. How dumb you ask? They should run a single command and parse a results file. Then 
they may show pretty plots if so desired.
- This one command triggers a pipeline, the name of which shall be parameterized.
- The pipeline needs to be implemented as an abstraction, so that arbitrary steps and success criteria can be implemented and plugged into the system.
- Pipelines will contain steps of unit tests, regression tests, performance tests, you name it. They receive the artifact on which to work on through a module, which in turn either builds from source (specific commit) or points to some location.
- A bisection system needs to be implemented, which also works with the same pipeline abstraction.
- Chose success criteria for regression tests wisely - this is where you earn your money, and what constrains any future development. Really know FP errros, tolerances and time series comparison. Know what's important to your customers.

## Challenges 

### User input - the futile task of sanitization and whitelisting

Input to physics based simulators consists of options and arbitrary numeric values in integer and floating point format. While we can sanitize these based on ranges of validity, *e.g.* physical constraints, and blacklist non-sensical combinations of options, *i.e.* combinations of algorithms known to cause issues, rigorous validation of input values and whitelisting of supported workflows seems out of reach. For all intents and purposes, the number of possible combinations of user input is infinite. This is in sharp contrast to web apps with graphical interfaces and their backends, which most of the currently developed CI tools target. There is no *happy path* in most numerical apps or libraries, there is only chaos. The user is exposed to the full wrath that is floating point arithemtic, non-linear algorithms, parallelization, iteration and so on. 

### Parallelization

Floating point arithmetic is sensitive to partitioning - think workload splitting of computations in multiple processes and threads. Results will differ between different parallelizations, unless implemented in invariant manner. In CI systems we usually test for several aspects related to this: scaling and invariance. The former referrs to performance, the latter to results. Tolerance aware success criteria thus need to be applied.

## Pipelines

It is imperative that any pipeline can be run on a dev's machine and that its definition is under version control.

### Implementation

I wish I could tell you here abaout a framework that ticks all the boxes to define continuous integration pipelines an run them locally and in a CI system. Unfortunately I'm not aware of any. Here is a selection of requirements we have for such a system:
- cross platform, win and linux
- knows how to retrieve commits from vcs
- run build commands
- run unittests
- run cluster tests through ssh

Ideally the CI system of your choice supports pipeline definition in some standardized file format like yaml. In this case, it's often easy to write a helper to run the same instructions locally. If not, shell scripts are the way to go, with ideally a single call from the CI system.

### Success criteria

The bad news here is that we can only screw this up. If defined too loosely, your customers will loose confidence in your product if their results change unacceptably (very subjective) every other update. If defined to tightly, you will end up living a OS kernel developers life under the no regression mantra, and seriosly impede agile approaches to development. It may increase the pressure to get things right the first time to unhealthy levels, and may stall development. It may lead to having maintain different options and implementation variants, which again will stall development pace through immense maintenance burden. Success criteria for CI tests have the potential to make or break your product and development approach.

{{< figure src="/img/testing-numapps/ci_hirarchy.png" caption="Interaction of CI systems with code and binaries. The only point of contact is through pipeline definitions, which typically consist of single files." >}}

Prefer qualitative criteria over quantitative ones as much as you can afford it, *e.g.* does the algorithm converge as opposed to does the algorithm converge to this and that value plus minus tolerances. More often than not this won't be an option, but it should be the starting point imo.

## Bisection

Long-running tests ('20') will necesserily test on an amalgamation of commits. Upon test failures, a bisection algorithm will be needed to identify culprits. This search algorithm is aware of the VCS, to walk code versions, and the pipeline in question - that's it.

## General points
If building on Linux, use `ninja` instead of `make`, if only for the advantage of not having to specify the number of jobs to run in parallel. This will make swapping agents and playing with different instaces on cloud VMs easier.

Put nothing in path, make all resources to be used explicit. Always prefer configuration files over environment variable. If a build/test setting is not under version control in sync with the source code, results loos reproducability and also loose their ephemeral nature, i.e. they start to be tightly coupled with the environment.

## More on simulation applications

Here scaling tests will be performed on changing spatial discretizations of the same physical objects. This requires special care in defining comparison functions, since relying on integrals/cumulatives can be a serious pitfall.

... :construction:

# Closing thoughts

[Technical dept of ML systems](https://research.google.com/pubs/pub43146.html):
> Unit testing of individual components and end-to-end tests of running systems are valuable, but in the face of a changing world such tests are not sufficient to provide evidence that a system is working as intended. Live monitoring of system behavior in real time is critical

Automated tests are lighttowers. Development and refactoring of numerical applications is a fight against the five-headed hydra that is floating point errors, parallelization, non-linear systems, iterative solutions, and an infinite number of combinations of user input.

Automated tests come mostly in two forms, unit tests and regression tests. The former dos not suffer from nearly as much ambiguity as the latter. Also unit tests are usually cheap to run, common place and thus don't need special attention here. Unfortunately, unit test often are not sufficient, and I'll try to share my 0.02$ on why that is. In a nutshell
- Iterative solutions
- Ill-conditioned systems
- Floating point arithmetic
- Infinite combinations of user input

Non-linear systems are notoriously more sensitive to small changes in distribution outliers than relatively larger shifts in the mean.
