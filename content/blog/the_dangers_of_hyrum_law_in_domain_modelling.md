---
title: "The dangers of Hyrum's Law in Domain Modelling"
date: 2021-10-27T13:18:20+02:00
slug: ""
description: ""
keywords: ["programming", "software", "hyrum", "software engineering"]
draft: false
tags: ["Domain", "Software Engineering"]
math: false
toc: false
---

Recently, I have gotten to know about *Hyrum's Law*, when reading *Software Engineering at Google*. This is an informal law that was raised in the early days of Google and states the following:

> With a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviors of your system will be dependend on by somebody


This law has quite several implications in different areas of software engineering, such as testing, deprecation, and domain modelling - and it directly impacts the code "maintainability" - since it indicates that the documented and tested behaviors may be a subset of all the expected behaviors.


<img src="/hyrum_law_set.png" style="width: 50%"/>


This can be troublesome during API refactor and migrations, since the APIs have hidden "expected" behaviors that we might break during refactors or eliminate in a migration. For example, largers companies perform planned outages to identify unexpected dependencies and usages of entire systems.

It was then fun, when I encountered a really good example of this law in action, and realized how important good domain modelling can help to decrease the possibility of unexpected observable behaviors - which should lead to more predictable software development.

## Example: Loading products by category

In this example, we examine the development of a real-life situation I encountered. The system under test is one where we are *loading a list of products of a given category*. For that, we have three particularly important abstractions:

- `GetProductsUseCase`
- `ProductsRepository`
- `ProductsApi`

Below, you can see the high-level picture of how each one depends on the other. It should then be clear that `GetProductsUseCase` is part of our `domain` layer, while `ProductsRepository/Api` is part of our data layer.

<img src="/hyrum_law_example_schema.png" style="margin-top: : 16px" />


The classes definition were following these wireframes:

```kotlin
interface ProductsApi {

	fun getProducts(category: String): List<RawProduct>
}

class ProductsRepository(api: ProductsApi) {
	
	fun getProducts(category: String): List<ProductDto> =
		api.getProducts(category)
			.map { ... }
}

class GetProductsUseCase(repository: ProductsRepository) {

	fun run(category: String) =
		repository.getProducts(category)
			.map { ... }
}
```

By looking, at this API, one would expect this fundamental behavior: *given a category, I receive a list of products for that category*. However, this wasn't what was happening after my **observation**.

When the system was initially developed, that was the expected behavior - give a category in the form of a `String` and receive a list of products. Hence, the first developers design the API as I declared above and the usage could be as simple as:

```kotlin
val products = getProductsUseCase.run("flowers")
```

 With time, more and more clients started using this API under the definition of the initial contract.

<p align="center" width="100%">
    <img src="/hyrum_law_initial_behavior.png" style="width: 50%">
</p>

However, at later point in time, one of the clients needed a new behavior: *given a list of categories, I receive a list of products for those categories*. What I imagine it happened, as a series of discussions (along side with time pressure) which resulted in the addition of unexpected behavior to the API and is highlighted in red in the figure below.


<p align="center" width="100%">
    <img width="50%" src="/hyrum_law_end_behavior.png" style="width: 50%">
</p>

The backend would receive the categories in the form of a comma separated list of categories `String`. This way, if I wanted the products for categories `flowers` and `books`, I would call `ProductsApi` in the following way:

```kotlin
productsApi.getProducts("flowers,books")
```

Given that our `domain` layer also expected a `category: String`, the new client just started passing `"category1,category2"` at that level, and things just worked because now the server was updated to deal with that behavior. However, this is a clear leak of an implementation detail of the server into our app's domain layer. And with this subtle gain of flexibility, suddenly the behavior of our whole system has changed, without a single line change in our codebase. Unfortunately, this change went under the radar, because no tests showcased this behavior nor documentation about it was added.

I only have realized about this fact once I try to enforce more structured domain modeling, by adding a `value class` for `Category`. After it, I tried to migrate client by client, to only realize that one of those was not using the API was the initial developers intended to.

## Better domain modelling to the rescue

It is then easy to conclude that if the domain layer had from the start a more "closed" domain by using a `Category` value class this suble change of API would have been caught, and properly dealt with.

One could have modelled the `Category` to enforce some basic validations:

```kotlin
inline class Category(val value: String) {
   init {
     require(value.hasOnlyLetters()) { 
     	"The given value is not a valid category since it has non alphabetic characters" }
   }
}
```

Whenever, the faulty client would try to create a `Category("flowers,drinks")`, this error would have been caught and they would be forced refactor the API accordingly to:

```
class GetProductsUseCase(...) {
  fun run(categories: List<Category>): List<Product>
}
```

I hope this was a good effort to one more reason on why good domain modeling is important.
