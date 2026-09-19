---
title: My Android PR Review Philosophy; 7 Years of Mobile Development Insights
date: 2025-06-13 23:50:00 +0200
categories: [Development, Code Review]
tags: [Development]
---

After seven years building Android apps, two of which spent leading engineering teams, i've learned that good code reviews can make or break a project, maybe not break it dramatically but throw a punch that can put a crack into it.

Whether i'm reviewing contributions to code i maintain, evaluating team members during promotion cycles, or just trying to keep our production apps from crashing, i've developed four core principles that consistently help me spot the difference between code that ships and code that survives.

Honestly, i used to overthink this process. i'd create massive checklists (more on why that's a terrible idea later), stress about every edge case, and turn reviews into interrogations.

But over thousands of pull requests ranging from tiny bug fixes to major architectural overhauls, i've found that focusing on these four areas gives me the most bang for my buck.

## Can Someone Else Actually Maintain This Code?

Let's be real: you're not going to remember why you wrote that clever bit of code six months from now, or maybe even earlier.

And neither will your teammates. i've inherited codebases where the original authors left the company years ago, taking all the context with them. It's... not fun.

![Look at me, i am the documentation now](/assets/img/pr_philosophy/documentation_joke.jpg)

The Android ecosystem moves fast:

- Jetpack Compose replaced traditional Views
- Kotlin became first-class (and thank you JetBrains, even though i was just getting comfortable with Java)
- Architectural patterns keep evolving from MVP to MVVM to MVI

When i review code, my first question is: will this survive the next major platform update, trend or some other library that influences the code?

Here's what i actually look for:

- **Method organization**: Are functions doing one thing well, or is there a 300-line `doSomething()` that handles everything from UI setup to network calls? (i was surprised to see that sometimes even seniors fall into this)
- **Naming that tells a story**: `processUserData()` tells me nothing, but `validateAndCacheUserPreferences()` gives me actual context
- **Comments that explain the "why"**: Not `// increment counter` but `// Retry with exponential backoff because the API rate limits after 3 consecutive failures`
- **Architectural clarity**: Does something start to run and never get to finish because it's cancelled because of wrong scope? Does the data layer abstract implementation details? is dependency injection structured to support feature modules or whatever setup we have?

Clear separation of concerns isn't academic, it's survival (wow that was dramatic).

When Google introduced Compose and then mandatory scoped storage, well architected apps adapted smoothly while monolithic ones required complete rewrites. i've been through both scenarios, and trust me, you want to be in the first camp.

## Does This Code Respect Mobile Reality?

Now with KMP, desktop applications can afford inefficiencies that mobile apps absolutely cannot. During reviews, i'm constantly thinking about the constraints our users actually face: dying batteries, memory pressure, spotty network connections, and devices that are definitely not the latest Shamesung/Pixel or whatever has top performance.

Key questions i tend to ask:

- **Compose performance**: Does this code trigger unnecessary recompositions?
- **Memory management**: Are we holding references that prevent garbage collection?
- **Background processing**: Will this work properly with Android's increasingly aggressive battery optimizations and how do we handle process death recovery?
- **Third-party dependencies**: Will this external library behave as we expect it to, or are we about to add 50MB to our APK and introduce transitive dependencies that sometimes can just crash without an easily traceable cause?

Network calls deserve special attention. Whether you're using Ktor, Retrofit, or raw HTTP clients, the configuration matters. Caching strategies, timeout handling and how you handle retries, and offline scenarios directly impact user experience. Usually you set this once and forget it, but the setup is critically important.

I've debugged too many ANRs caused by seemingly innocent changes that worked fine in the emulator but failed spectacularly on real hardware.

**I'm looking at you ncnn.**

If your code is performant and works well on a really cheap and poorly optimized phone, it will for sure be blazing fast on a more powerful device.

## Are We Testing the Right Things?

Android testing is... complex. You've got unit tests for business logic, instrumentation tests for Android components, UI tests for user flows and screenshot tests for visual regressions. Each serves a purpose, but not every change needs every type of test.

My testing philosophy has evolved from "test everything" to "test what matters":

- **Core business logic**: Solid unit test coverage with fake dependencies implementation testing almost production data
- **Complex UI components**: Screenshot tests that catch visual regressions across different devices. Now foldables and big screens enter the picture and i think screenshot testing will become even more relevant
- **Critical user journeys**: End-to-end tests that simulate real device conditions
- **Integration points**: Tests that verify your code works with actual Android components, not just fakes

The key insight: tests should fail when users would notice problems. A test that passes while the app crashes on device for example during orientation changes isn't protecting anyone.

I also found that almost no one tests process death scenarios when earning some of those sweet integration points but very often comes back to do once an unexpected behavior is met, if of course caught.

Some of you might find it weird why i didn't mention mocking? Recently one colleague mocked the behavior in favor of his code and missed some important scenarios that he could've avoided if using fakes, also c'mon, verifying that a function was called instead of calling it an expecting a result, it boggles my mind that i've advocated mocking for 2 years.

## Can We Debug This When It Breaks in Production?

Mobile apps present unique monitoring challenges.

Users report bugs weeks after crashes occur or sometimes never, often with minimal reproduction steps like "it just stopped working." and it's not their fault, they're not technical. Sometimes they're your regular neighbor who knows everything about how to build a car engine but struggles when using a smart phone.

Effective instrumentation helps bridge this gap between development and reality.

Essential observability questions:

- **Crash reporting**: Does this feature include appropriate tags and context for Firebase Crashlytics, Sentry or Bugsnag?
- **Performance metrics**: Are we capturing timing data for slow code paths? (this is one of the hard parts alongside non descriptive ANRs)
- **User journey tracking**: Can we trace user flows through analytics events? (if applicable)
- **Business metrics**: How will we measure if this feature actually provides value? (if applicable)

These questions become critical when debugging issues reported by users across different devices, OS versions, and network conditions. Mobile analytics tools like Firebase provide rich insights, but only if we instrument thoughtfully from the beginning.

## Accessibility: Not Optional Anymore

With the European Accessibility Act coming into effect, it's legally required for many apps. During reviews, i now check:

- **Content descriptions**: Are Image and custom components properly labeled?
- **Content fluidity**: Is this code properly announcing things when they're in two different rows, i.e is the order sorted?
- **Focus management**: Can users navigate with TalkBack or external keyboards?
- **Color contrast**: Does the design meet WCAG standards or whatever you're aiming for?
- **Dynamic text support**: Does the UI adapt to user-preferred text sizes?

I've seen too many apps scramble to add accessibility retrofits when they could have built it in from the start. It's much easier to review for accessibility during development than to audit and fix an entire app later.

I've only started this journey recently as when the EU law forced it for one of our app to be compliant, knowing that we have users who utilize TalkBack, you want them to enjoy a good UX.

## Growing the Team Through Reviews

Here's something i learned during my team lead years: code reviews aren't just about catching bugs, they're mentoring opportunities. When i find well-implemented solutions, i highlight them explicitly. "Nice use of sealed classes here, this makes the state management really clear" helps reinforce good patterns.

This approach has been especially valuable when leading teams. Junior developers gain confidence when their good decisions are acknowledged. Senior developers appreciate recognition for thoughtful implementations. The feedback loop strengthens team skills over time and makes the review process less painful for everyone.

## Why I Abandoned Checklists

Early in my career, i tried comprehensive checklists covering every possible Android pitfall: memory leaks, lifecycle awareness, security considerations, accessibility compliance, performance optimizations... The lists grew to multiple pages and became counterproductive.

Here's what happened: either i'd mindlessly check boxes without really thinking about the code, or i'd get overwhelmed trying to apply every item to every change. A networking library PR needs different scrutiny than a UI animation fix. Checklist-driven reviews either become superficial box-checking exercises or paralyze reviewers with irrelevant concerns.

Instead, these four principles adapt to any Android codebase while keeping reviews focused and thorough. They free up mental capacity to actually think about the specific change instead of running through a generic list.

## Putting It All Together

These standards guide both my review process and my own development work. When contributing to open source projects or implementing features for production apps, i evaluate against these same criteria.

The mobile development landscape continues evolving with new Compose APIs, updated architecture guidance, platform changes that break half your assumptions. But the fundamental principles of maintainable, performant, observable, and well-tested code remain constant.

After thousands of pull requests across multiple teams and projects, this approach consistently identifies the changes that will succeed in production and age well over time. Plus, it makes the review process more collaborative and less adversarial, which benefits everyone involved.

Keep in mind that there are static and dynamic analysis tools that can help you in this journey, linters are your your semi-best friend, PR templates etc...

Also being up to date is sometimes a big big benefit for everyone in the team, knowing some existing issues with libraries used in the project or knowing an alternative to a library.

Your future self (and your teammates) may thank you for applying these principles consistently and as always, whatever works best for your team, some times you might need to be the change in your team, so do it.

## Closing Words

Thank you for your attention, i hope this brings value to you and your team.

Stay safe, drink enough water and apply sun screen, i'm recovering from a pretty serious leg injury and didn't have much time to put out something shiny/technical, but this is something i've been writing for one week now, until next article.
