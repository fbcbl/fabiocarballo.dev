---
title: "How bad domain modeling relates to Hyrum's Law"
date: 2021-10-27T13:18:20+02:00
slug: ""
description: ""
keywords: ["programming", "software", "hyrum", "software engineering"]
draft: false
tags: ["Domain", "Software Engineering"]
math: false
toc: false
---

I recently got to know about *Hyrum's Law*, when reading [*Software Engineering at Google*](https://www.oreilly.com/library/view/software-engineering-at/9781492082781/). This is an informal law that was raised in the early days of Google and states the following:

> *With a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviors of your system will be dependend on by somebody*


This law has several implications in different areas of software engineering, such as testing, deprecation, and domain modeling - and it directly impacts the code "maintainability" - since it indicates that the documented and tested behaviors may be a subset of all the expected behaviors.



<img src="/hyrum_law_set.png" style="width: 50%; margin-top: 16px"/>


This can be troublesome during API refactor and migrations since the APIs have hidden "expected" behaviors that might break during refactors or be eliminated in a migration. To decrease this problem, larger companies perform planned outages to identify unexpected dependencies and usages of entire systems.

It was then fun, when I encountered a really good example of this law in action, and realized how important good domain modeling can help to decrease the possibility of unexpected observable behaviors - which should lead to more predictable software development.

## Example: Loading products by category

In this example, we examine the development of a real-life situation I encountered. The system under test is one where we are *loading a list of products of a given category*. For that, we have three fundamental abstractions:

- `GetProductsUseCase` - a public domain abstraction used by other clients to fetch products
- `ProductsRepository` - an internal abstraction over how we fetch our data
- `ProductsService` - an internal abstraction over the product service (backend)

Below, you can see the high-level picture of how each one depends on the other. It should then be clear that `GetProductsUseCase` is part of our `domain` layer, while `ProductsRepository/Api` is part of our data layer.

<img src="/hyrum_law_example_schema.png" style="margin-top: : 16px" />


These three abstractions definition could be declared as the following:

```kotlin
interface ProductsService {

	fun getProducts(category: String): List<RawProduct>
}

class ProductsRepository(service: ProductsService) {
	
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

By looking, at this API, one would expect the following behavior:

> *given a category, a list of products for that category is returned*. 

However, this wasn't what was observed.

When the system was initially developed, that was the expected behavior - given a category in the form of a `String` a list of products (with that category) was returned. Hence, given the API shown above, one example of usage could be:

```kotlin
val products = getProductsUseCase.run("flowers")
```

 As time went by, more and more clients started using this API under the definition of the initial contract.

<p align="center" width="100%">
    <img src="/hyrum_law_initial_behavior.png" style="width: 50%">
</p>

Ultimately, one of the clients needed a new behavior: 

> *given a list of categories, a list of products for those categories is returned*. 

To accommodate this, the team decided that the `ProductsService` would receive the categories in the form of a comma-separated list of categories `String`. This way, if I wanted the products for categories `flowers` and `books`, we would call `ProductsService` in the following way:

```kotlin
productsService.getProducts("flowers,books")
```

Given that our `domain` layer also expected a `category: String`, the new client just started inputting `"category1,category2"` at that level, and things just worked because now the service was updated to deal with that behavior. However, this is a clear leak of an implementation detail of the server into our app's domain layer. And with this subtle gain of flexibility, suddenly the behavior of our whole system has changed, without a single line change in our codebase. Unfortunately, this change went under the radar, because no tests showcased this behavior nor documentation about it was added.

As a result of this change of implementation details and one wrong usage of the API, we had a breach of the original contract of the API, as highlighted in red in the figure below.


<p align="center" width="100%">
    <img width="50%" src="/hyrum_law_end_behavior.png" style="width: 50%">
</p>


I have realized about this behavioral change once I tried to enforce a more structured domain modeling, by adding a `value class` for `Category`. As I tried to migrate each client I then realized that one client was not using the API as the initial developers intended to.

## Better domain modeling to the rescue

The lesson learned was that if the domain layer had been designed from the start a more "strict" domain we wouldn't end up with such "hidden" behaviors. 

As a practical example, one could have used a `Category` value class this subtle change of API would have been caught and properly dealt with. One could have modeled the `Category` to enforce some basic validations:

```kotlin
@JvmInline
inline class Category(val value: String) {
   init {
     require(value.hasOnlyLetters()) { 
     	"The given value is not a valid category since it has non alphabetic characters" }
   }
}
```

Whenever the faulty client would try to create a `Category("flowers,drinks")`, this error would have been caught and they would be forced refactor the API accordingly to:

```kotlin
class GetProductsUseCase(...) {
  fun run(categories: List<Category>): List<Product>
}
```

It is important to note that an `Enum` wouldn't work for this case -  since our list of categories is not a fixed set of categories. They are provided by the service and can vary at any time.

I hope this scenario provided you with an example of why domain modeling is very important when it comes to API design.


For more on this topic:
- [Domain Modeling Made Functional](https://fsharpforfunandprofit.com/books/)
- [Deep Models](https://marcellogalhardo.dev/posts/2021/10/07/deep-models/)
