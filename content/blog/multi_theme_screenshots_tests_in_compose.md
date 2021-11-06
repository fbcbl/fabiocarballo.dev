---
title: "Multi-Theme Screenshots in Jetpack Compose"
date: 2021-11-06T17:44:01+01:00
slug: "Scale up your screenshot testing."
description: ""
keywords: ["android", "compose", "testing", "screenshot"]
draft: false
tags: ["Compose", "Testing", "Screenshot"]
math: false
toc: false
---

I have been working a lot with a design system and one of the recent challenges was to add support to screenshot testing while making it easy to scale for
several different themes. In my vision, we would be able to only write one single test, but automatically generate the screenshots for all the themes in one single run

Meanwhile, I reached a solution that can be seen in this sample project created for this purpose. In this project, we have a really simple design system and we can see an example of how to generate screenshot tests that will verify the system's Typography in both _Light_ and _Dark_ theme. 

<br/>

| Light  | Dark |  
|---|---| 
|  ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_label_light](https://user-images.githubusercontent.com/6845042/140617048-d7b5d3b6-3372-49d1-a336-ea748a3ff52b.png) | ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_label_dark](https://user-images.githubusercontent.com/6845042/140617040-a80f60de-5869-4b2b-a860-b035c5e69955.png) | 

P.s - if you want to look into the code you can look at this specific [PR](https://github.com/fabiocarballo/compose-design-system/pull/1) where I add the multi-theme screenshot support.

Let's go step by step of what was the thought process into getting into the final solution:

## 1. Integrating with Shot

To be able to have screenshot tests, we are integrating [Shot](https://github.com/pedrovgs/Shot) as our _screenshot testing tool_.

Any test can have the ability to take screenshot by:
- Extending `shot.ScreenshotTest`
- Calling `compareScreenhot(rule: ComposeTestRule)`.

This way we could easily have a screenshot test in this form:

```kotlin
class TypographyTest {
   
   @get:Rule
   val composeTestRule = createComposeRule()

   @Test
   fun paragraph() {
       composeTestRule.setContent {
          Theme {
              Text(text = "This is a paragraph.", style = Theme.typography.paragraph)
           }
       }
    
       compareScreenshot(rule = composeTestRule)
   }
}
```

By doing this we then have a screenshot generated in our module under `app/screenshots/com.fabiocarballo.designsystem.TypographyTest.label`

![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_paragraph_light](https://user-images.githubusercontent.com/6845042/140614050-73d4d21b-a64d-4865-8c04-c91ea574f18a.png)

## 2. Improving the API

At this point, we know how to make a screenshot test. However, as we grow the number of tests we would be having a lot of structural duplication:

1. Extending `ScreenshotTest`
2. Declaring the `ComposeTestRule`
3. Setting the composable content (and not forgetting to wrap it under our `Theme`)
4. Comparing the results of the screenshot.

We can then extract this structure into a `DesignSystemScreenshotTest`:

```kotlin
abstract class DesignSystemScreenshotTest: ScreenshotTest {

   @get:Rule
   val composeTestRule = createComposeRule()

   fun runScreenshotTest(content: @Composable () -> Unit) {
         composeTestRule.setContent {
             Theme(content = content)
          }
          
          compareScreenshot(composeTestRule)
   }
}
```

You can see that all the structure was then passed into this `abstract class` that every test class should extend.  Another point to note is that all content should be run under the scope of this `runScreenshotTest`. Let's see how our `TypographyTest` would look like now:


```kotlin
class TypographyTest: DesignSystemScreenshotTest() {
    
    @Test
    fun paragraph() = runScreenshotTest {
       Text(text = "This is a paragraph.", style = Theme.typography.paragraph)
    }

    @Test
    fun label() = runScreenshotTest {
        Text(text = "This is a label.", style = Theme.typography.label)
    }

    @Test
    fun display() = runScreenshotTest {
        Text(text = "This is a display.", style = Theme.typography.display)
    }
}
```

The goal here was to make a test being almost as easy as just declare the composable that should be screenshotted.

By now, our screenshots would be:

| Label  |  Paragraph  |  Display  |  
|---|---|---|
| ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_label_light](https://user-images.githubusercontent.com/6845042/140614410-5c4a09d5-dedc-4d54-ba95-73d4e0c06a0f.png)  | ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_paragraph_light](https://user-images.githubusercontent.com/6845042/140614382-51e2a13c-711d-4827-867d-4fdacf293724.png)  |  ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_display_light](https://user-images.githubusercontent.com/6845042/140614391-bafa3c33-6f7b-4991-b0e9-3d27c6c4d665.png)  |  

## 3. Add support to multi-theme

At this point, we have now the capability to easily generate our screenshots for our _Light Theme_. As a next step, we want to use the same codebase and automatically generate screenshots also to _Dark Theme_.

For that, we are going to enrich our `DesignSystemScreenshotTest` with the capability to run parameterized tests. Hence, we are using [Test Parameter Injector](https://github.com/google/TestParameterInjector) to build the parameterized behavior.

What we are going to do first is to declare a private enum with the themes we want to parameterize with:

```kotlin
private enum class ThemeMode { LIGHT, DARK }
```

Then we are going to declare it as a test parameter so that each test will run with both `LIGHT` and `DARK`. This is done by adding a field annotated with `@TestParameter` and by adding integrating with the `TestParameterInjector` test runner.

```kotlin
@RunWith(TestParameterInjector::class)
abstract class DesignSystemScreenshotTest : ScreenshotTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @TestParameter
    private lateinit var themeMode: ThemeMode

    fun runScreenshotTest(
        content: @Composable () -> Unit
    ) {

        composeTestRule.setContent {
            Theme(
                isSystemInDarkMode = themeMode == ThemeMode.DARK,
                content = content
            )
        }

        compareScreenshot(
            rule = composeTestRule,
        )
    }

    private enum class ThemeMode { LIGHT, DARK }
}
```

However, we still have one small problem: `Shot` default behavior is to name the screenshot as _"ClassName_MethodName"_. This way, we record the screenshots for the test in both modes, but the last run always overrides the first one.

To fix that, we are going to generate the screenshot name by ourselves with the help of this helper method:

```kotlin
private fun extractClassAndMethodName(): String {
    val stack = Throwable().stackTrace

    stack.forEach { element ->
        try {
            val clazz = Class.forName(element.className)
            val method = clazz.getMethod(element.methodName)

            if (method.annotations.any { it.annotationClass == Test::class }) {
                return "${clazz.canonicalName}_${method.name}"
            }
        } catch (ignored: NoSuchMethodException) {
            // do nothing
        } catch (ignored: ClassNotFoundException) {
            // do nothing
        }
    }

    error("Couldn't parse the name")
}
```

This method will simply use the `StackTrace` to figure out what is the test class and method name. We can then use this information together with the `ThemeMode` that is being used for the test to generate a test name as:

- _TypographyTest_paragraph_light_
- _TypographyTest_paragraph_dark_

Below you have the final version of the `DesignSystemScreenshotTest`:

```kotlin
@RunWith(TestParameterInjector::class)
abstract class DesignSystemScreenshotTest : ScreenshotTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @TestParameter
    private lateinit var themeMode: ThemeMode

    fun runScreenshotTest(
        content: @Composable () -> Unit
    ) {

        composeTestRule.setContent {
            Theme(
                isSystemInDarkMode = themeMode == ThemeMode.DARK,
                content = content
            )
        }

        val name = "${extractClassAndMethodName()}_${themeMode.name.lowercase()}"

        compareScreenshot(
            rule = composeTestRule,
            name = name
        )
    }

    private enum class ThemeMode { LIGHT, DARK }
}
```

And that is it! When you run your screenshot tests you will generate both _Light_ and _Dark_ mode screenshots in one go. In our example, the generated screenshots are the following:

<br/>

| Label  | Paragraph   | Display  |  
|---|---|---|
|  ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_label_light](https://user-images.githubusercontent.com/6845042/140617048-d7b5d3b6-3372-49d1-a336-ea748a3ff52b.png)| ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_paragraph_light](https://user-images.githubusercontent.com/6845042/140617062-d796727b-1b19-4f3e-8b27-3b28645612d2.png)  |  ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_display_light](https://user-images.githubusercontent.com/6845042/140617012-83d51bce-5608-4811-9f8e-060b9ff34cc1.png)
| ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_label_dark](https://user-images.githubusercontent.com/6845042/140617040-a80f60de-5869-4b2b-a860-b035c5e69955.png) |   ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_paragraph_dark](https://user-images.githubusercontent.com/6845042/140617076-384c5654-ad98-490e-8b14-63dd21e20daf.png)| ![com fabiocarballo designsystem TypographyTest_com fabiocarballo designsystem TypographyTest_display_dark](https://user-images.githubusercontent.com/6845042/140617006-4f679771-0625-4a56-83b9-96cf048fd257.png)|




