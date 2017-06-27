---
layout: post
title:  "Composable HTML Views in Swift"
date:   2017-06-27
categories: swift html dsl
author: Brandon Williams
summary: "We define a view as a function from data to HTML nodes, and show how this results in three different types of compositions of views."


image: TODO
---

[Last time]({% post_url 2017-06-22-type-safe-html-in-swift %}) we defined a DSL in Swift for creating HTML documents. We accomplished this by creating some simple value types to describe the domain (nodes and attributes) and some helper functions for generating values. In the end it looked like this:

```swift
header([
  h1(["id" => "welcome"], ["Welcome!"]),
  p([
    "Welcome to you, who has come here. See ",
    a(["href" => "/more"], ["more"]),
    "."
  ])
])
```

to generate this HTML:

```html
<header>
  <h1 id="welcome">Welcome!</h1>
  <p>
    Welcome to you, who has come here. See <a href="/more">more</a>.
  </p>
</header>
```

Now we are going to tackle a problem that goes up one layer in the web-server request lifecycle: creating views. These are the things responsible for taking some data, say a list of articles, and creating the HTML to represent that data. It may sound easy, but there are a lot of difficult problems to solve with views. We want our views to be composable so that we can create small sub-views focused on rendering one piece of data and be able to reuse it. We also want our views to be flexible enough for future developments that are hard to see right now.

## The View Function

Like most topics discussed on this site, we are going to define view as a plain ole function. Also, like most topics in computer science, a monoid is involved. Luckily we’ve talked about these topics a lot on this site ([here](%{ post_url 2015-02-17-algebraic-structure-and-protocols %}) and [here](%{ post_url 2017-04-18-algbera-of-predicates-and-sorting-functions %})), and so there’s a lot of material to pull from. We will assume you are familiar with most of this content, and quickly recap the code that defines these objects:

```swift
precedencegroup Semigroup { associativity: left }
infix operator <>: Semigroup

protocol Semigroup {
  static func <>(lhs: Self, rhs: Self) -> Self
}

protocol Monoid: Semigroup {
  static var e: Self { get }
}

extension Array: Monoid {
  static var e: Array<Element> { return  [] }
  static func <>(lhs: Array, rhs: Array) -> Array {
    return lhs + rhs
  }
}
```

Here we have defined two protocols that define what a semigroup and monoid are, and made `Array` conform because that is the monoid we are going to be most interested in for this article.

We are going to define a view, roughly, as a function from some data to an HTML node (which we defined [last time]({% post_url 2017-06-22-type-safe-html-in-swift %})). It turns out that it helps to actually map into an array of nodes, `[Node]`, since that aids composition by stacking views on top of each other. It also helps to generalize `[Node]` to be any monoid, not just `[Node]`. We will do this by making a struct that wraps a function:

```swift
struct View<D, N: Monoid> {
  let view: (D) -> N

  init(_ view: @escaping (D) -> N) {
    self.view = view
  }
}
```

All of our views will be mapping into `[Node]`, but that lil extra bit of generality will pay dividends later. With this simple type we can cook up a few views. Below I have roughly recreated some of the HTML on this very site:

```swift

let headerContent = View<(), [Node]> { _ in
  [
    h1(["Few, but ripe..."]),
    menu([
      a([href => "#"], ["About"]),
      a([href => "#"], ["Hire Me"]),
      a([href => "#"], ["Talks"])
      ])
  ]
}

struct FooterData {
  let authorName: String
  let email: String
  let twitter: String
}

let footerContent = View<FooterData, [Node]> {
  [
    ul([
      li([.text($0.authorName)]),
      li([.text($0.email)]),
      li([.text("@\($0.twitter)")]),
      ]),
    p(["Articles about math, functional programming and the Swift programming language."])
  ]
}
```

Notice that the header view has a `()` data parameter because it doesn’t need any data to construct its nodes. The footer however does need some data, which we packaged up into a struct. Let’s also make a view that renders a list of articles:

```swift
struct Article {
  let date: String
  let title: String
  // more fields, e.g. author, body, categories, ...
}

let articleListItem = View<Article, [Node]> { article in
  [
    li([
      span([.text(article.date)]),
      a([href => "#"], [.text(article.title)])
      ])
  ]
}

let articlesList = View<[Article], [Node]> { articles in
  [
    ul(
      articles.flatMap(articleListItem.view)
    )
  ]
}
```

Note that we first created a helper view `articleListItem` for rendering a single item, and then we used it to render the full list of articles. We have to `flatMap` onto `articleListItem.view` since it returns an array of nodes.

We can now bring this all together to create a homepage view that composes these views together. Since the


```swift
struct HomepageData {
  let articles: [Article]
  let footerData: FooterData
}

let homepage = View<HomepageData, [Node]> {
  [
    html(
      [
        body(
          [ header(headerContent.view(())) ]
            + [ main(articlesList.view($0.articles)) ]
            + [ footer(footerContent.view($0.footerData)) ]
        )
      ]
    )
  ]
}
```

It isn’t the prettiest code right now, but we’ll make it better soon. And it’s not _that_ bad right now! Some things to note:

* It’s just a pure function mapping an array of articles to some HTML nodes, which can be rendered to a string and then tested in a unit test.

* We got to leverage Swift features to aid in composition. For example, here we used array concatenation with `+` to stack our three views on top of each other. It doesn’t look great right now because we had to wrap each of our subviews in semantic tags.

* We were able to use subviews to clean up this view and make it clear that we are just stacking a header on top of main content on top of a footer.


We can take the homepage for a spin by creating some homepage data and rendering. The `render(node:)` function we made [last time](%{ post_url 2017-06-23-rendering-html-dsl-in-swift %}) only works on nodes, not views. We can write a `render` for views like so:

```swift
func render<D>(view: View<D, [Node]>, with data: D) -> String {
  return view.view(data).map(render(node:)).reduce("", +)
}
```

And now creating some data and rendering looks like this:

```swift
let data = HomepageData(
  articles: [
    Article(date: "Jun 22, 2017", title: "Type-Safe HTML in Swift"),
    Article(date: "Feb 17, 2015", title: "Algebraic Structure and Protocols"),
    Article(date: "Jan 6, 2015", title: "Proof in Functions"),
    ],
  footerData: .init(
    authorName: "Brandon Williams",
    email: "mbw234@gmail.com",
    twitter: "mbrandonw"
  )
)

render(view: homepage, with: data)
```

which will output the following HTML.

```html
<html>
  <body>
    <header>
      <h1>Few, but ripe...</h1>
      <menu>
        <a href="#">About</a>
        <a href="#">Hire Me</a>
        <a href="#">Talks</a>
      </menu>
    </header>
    <main>
      <ul>
        <li><span>Jun 22, 2017</span><a href="#">Type-Safe HTML in Swift</a></li>
        <li><span>Feb 17, 2015</span><a href="#">Algebraic Structure and Protocols</a></li>
        <li><span>Jan 6, 2015</span><a href="#">Proof in Functions</a></li>
      </ul>
    </main>
    <footer>
      <ul>
        <li>Brandon Williams</li>
        <li>mbw234@gmail.com</li>
        <li>@mbrandonw</li>
      </ul>
      <p>Articles about math, functional programming and the Swift programming language.</p>
    </footer>
  </body>
</html>
```

This is complicated enough view that was quite simple to compose from smaller pieces! (todo: more)

## View Composition

Since `View` is a function, we would expect that there are some nice ways to compose them. And indeed, there are at least 3 ways!

### View Composition #1 – Map

The first type of composition we will discuss is called `map`. It is closely related to `Array`’s `map`, so let us recall that definition. `Array<A>` has a method called `map` that takes a function `f: (A) -> B` and returns an `Array<B>`. You can think of `f: (A) -> B` has transforming `Array<A>`s to `Array<B>`s by just applying `f` to each element of the array.

`View<D, N>` also has such a function, but it transforms the `N` part of the view. Let us first write the signature of how such a function would look:

```swift
extension View {
  func map<S>(_ f: @escaping (N) -> S) -> View<D, S> {
    ???
  }
}
```

We know that this method returns a `View<D, S>`, which is really just a function `(D) -> S`, so we can fill in a bit of this function body:

```swift
extension View {
  func map<S>(_ f: @escaping (N) -> S) -> View<D, S> {
    return View<D, S> { d in
      ???
    }
  }
}
```

Now we have at our disposal `d: D`, `f: (N) -> S`, and `self.view: (D) -> N`. Seems like we can just compose these two functions and feed `d` into em:

```swift
extension View {
  func map<S>(_ f: @escaping (N) -> S) -> View<D, S> {
    return View<D, S> { d in
      f(self.view(d))
    }
  }
}
```

This function allows us to perform a transformation on a view’s nodes without knowing anything about the data that is involved. We already did something like this (3 times) when we wrapped our subviews in semantic tags, for example:

```swift
[ footer(footerContent.view($0.footerData)) ]
```

If we wanted to store this in another view, it would be a bit cumbersome in the naive way:

```swift
let siteFooter = View { [ footer(footerContent.view($0)) ] }
```

Our function `map` allows us to do this quite succinctly:

```swift
let siteFooter = footerContent.map { [footer($0)] }
```

Even better, if we take a few nods from [Haskell](https://www.haskell.org), [PureScript](http://www.purescript.org) and other purely functional languages, we could rewrite this as:

```swift
precedencegroup ForwardComposition { associativity: left }
infix operator >>>: ForwardComposition

// Left-to-right function composition, i.e. (f >>> g) = g(f(x))
func >>> <A, B, C>(f: @escaping (A) -> B,
                   g: @escaping (B) -> C) -> (A) -> C {
  return { g(f($0)) }
}

// Wrap a single value in an array.
func pure<A>(_ a: A) -> [A] { return [a] }

let siteFooter = footerContent.map(footer >>> pure)
```

This line says that `siteFooter` is made by mapping over `footerContent`’s nodes, wrapping those nodes in a `footer` tag, and then wrapping that in an array. We can use this for all 3 of our subviews:

```swift
let siteHeader = headerContent.map(header >>> pure)
let mainArticles = articlesList.map(main >>> pure)
let siteFooter = footerContent.map(footer >>> pure)
```

That is starting to look really nice! We can now have our subviews concentrate _only_ on the content they provide, and not worry about what their enclosing tag should be. For example, `articleListItem` rendered an `li` tag with the article date and title in it. We could generalize this so that we could potentially reuse that view in places that are not necessarily a list:

```swift
let articleCallout = View<Article, [Node]> { article in
  [
    span([.text(article.date)]),
    a([href => "#"], [.text(article.title)])
  ]
}
```

And then it’s `articlesList` to properly wrap that subview in `li` tags:

```swift
let articlesList = View<[Article], [Node]> { articles in
  [
    ul(
      articles.flatMap(articleCallout.view >>> li >>> pure)
    )
  ]
}
```

This allows us to maximize reusability of our subviews. We will also soon be able to use our other two forms of composition to even simplify that!

### View Composition #2 or #3 – Contramap



### View Composition #2 or #3 – Monoid

The next form of composition we will encounter is from our requirement that `N` be a monoid in the definition of `View<D, N>`. Recall from a [previous article](%{ post_url 2017-04-18-algbera-of-predicates-and-sorting-functions %})) we showed that the type of functions from a type into a monoid also forms a monoid. This means that `View` is a monoid, and so let’s implement its conformance:

```swift
extension View: Monoid {
  static var e: View {
    return View { _ in N.e }
  }

  static func <>(lhs: View, rhs: View) -> View {
    return View { lhs.view($0) <> rhs.view($0) }
  }
}
```

This function allows us to combine two views by stacking them, as long as the type of data they each take matches up. For example, say we were building a view for a particular article on this site. It consists of a header (title, date, author), body (paragraphs, code snippets), and footer (contact info). All of those views can be rendered from an `Article` value:

```swift
let articleHeader = View<Article, [Node]> { ... }
let articleBody = View<Article, [Node]> { ... }
let articleFooter = View<Article, [Node]> { ... }
```

These views can be composed together with `<>` to obtain a new view that just stacks the views:

```swift
let fullArticle: View<Article, [Node]> =
  articleHeader
    <> articleBody
    <> articleFooter
```

And then later we could `map` on this view in order to wrap it in an `article` tag:

```swift
fullArticle.map(article >>> pure)
```




## Bonus

Using `get` helper we can use keypaths...



.

## References

* http://www.fewbutripe.com/swift/html/dsl/2017/06/22/type-safe-html-in-swift.html
* http://www.fewbutripe.com/swift/math/algebra/monoid/2017/04/18/algbera-of-predicates-and-sorting-functions.html
* http://www.fewbutripe.com/swift/math/algebra/2015/02/17/algebraic-structure-and-protocols.html