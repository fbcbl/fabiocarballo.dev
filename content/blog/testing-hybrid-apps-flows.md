---
title: "Testing Hybrid Apps Flows with emptyComposeRule"
date: 2021-10-07T11:17:22+02:00
slug: "Testing Hybrid App Flows"
description: "Leverage the emptyComposeRule"
keywords: ["android", "compose", "testing", "jetpack", "emptyComposeRule"]
draft: false
tags: ["Compose", "Testing"]
math: false
toc: false
---

Recently, we have migrated one of our app screens to Compose - in an experiment that reduced the UI layer codebase by 60%. However, by the time we wanted to test our app we found a problem.

At work, we have a small set of critical tests that we run as pure E2E tests. Therefore, every test starts by launching the app and traverse several different screens to verify that its expectations are met. In some scenarios we do start the `Activity` to be tested, allowing us to skip some harder to set-up steps.

By looking at the general-purpose [documentation](https://developer.android.com/jetpack/compose/testing) to testing with Compose, there are some mentions to the `createAndroidComposeRule()` for the cases you want to test an `Activity` that hosts `Compose` content (which was our case).

In a general case, this approach is simple enough: you specify the activity you want to launch in the rule, and then you use the same rule to perform assertions on the Compose content:

```kotlin
@get:Rule
val rule = createAndroidComposeRule<FeatureActivity>()

@Test
fun testExpectedBehavior() {
	rule.onNodeWithText("My text")
	    .assertExists()
}
```

However, for our situation it didn't fit our needs for two distinct reasons:

1) Our testing infrastructure is still relying on the deprecated `ActivityTestRule` - while the `createAndroidComposeRule` works on top of `ActivityScenario`. To accommodate this, we would have to perform a migration and it would block the Compose initiative.

2) For the cases where we want to launch an `Activity` with a specific start `Intent`, we weren't able to found an API that allows us to specify the start `Intent`. 

```kotlin
val rule = createAndroidComposeRule<FeatureActivity>() 

// how can I define different starting `Intent` data for each test?
```

The traditional `ActivityScenario.launch(Intent)` would cover this scenario, but the public API of the `createAndroidComposeRule` didn't offer this option.


## emptyComposeRule()

Inspecting the source code, I then found the `emptyComposeRule()`: 

> A typical use case on Android is when the test needs to launch an Activity (the compose host)  after one or more dependencies have been injected.

This rule suited our needs since it allows an independent launch/setup of the `Activity` to be launched.  This way all we have to do is to declare it, define how you want to launch your `Activity`, and then use the rule to assert your `Compose` views.

A good example of usage would be:

```kotlin
@get:Rule
val composeRule = createEmptyComposeRule()

@Test
fun testExpectedBehavior() {
   val targetContext = InstrumentationRegistry.getInstrumentation().targetContext
   val intent = Intent(targetContext, FeatureActivity::class.java)
       .putExtra("id", "12345")
   val scenario = ActivityScenario.launch(intent)
   
   composeRule
      .onNodeWithText("My Text")
      .assertItExists()
}
```

I hope this helps you.

Follow me on [Twitter](https://twitter.com/@fabiocarballo)

