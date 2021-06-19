---
title: Infinite scroll pagination with Rails and Stimulus
description: This description will be used in the meta tags
date: 2021-06-19
minutes_read: 10
tags: [rails, javascript]
---

In this article, we'll learn how to build an infinite scroll pagination system using only a few lines of code. We will create a very simple Rails application and implement the infinite scroll feature in a Stimulus Controller that you can reuse to paginate all the resources of your app. We will do this step by step so let's begin!

## Creating the Rails application

Let's start by creating a new Rails application with Stimulus installed:

```bash
rails new infinite-scroll-article --webpack=stimulus
```

We'll start by building a pagination feature that works without any Javascript. Let's first create a model `Article` with a string title and a text content.

```bash
rails g model Article title content:text
rails db:migrate
```

Now that we have our `Article` model, let's create a seed that creates 100 articles for us to paginate.

```rb
# db/seeds.rb

puts "Remove existing articles"
Article.destroy_all

puts "Create new articles"
100.times do |number|
  Article.create!(
    title: "Title #{number}",
    content: "This is the body of the article number #{number}"
  )
end
```

To persist those 100 articles in the database, let's run the command:

```bash
rails db:seed
```

We are good to go for the model part, let's now create a controller with only the `#index` method and the corresponding view to display those 100 articles.

```rb
rails g controller articles index
```

In the routes file, let's make our articles list the home page:

```rb
# config/routes.rb

Rails.application.routes.draw do
  root "articles#index"
end
```

In the controller, let's query all the articles from the database:

```rb
# app/controllers/articles_controller.rb

class ArticlesController < ApplicationController
  def index
    @articles = Article.all
  end
end
```

Finally, let's display all of our  100 articles in the view.

```html
<!-- app/views/articles/index.html.erb -->

<h1>Articles#index</h1>

<% @articles.each do |article| %>
  <article>
    <h2><%= article.title %></h2>
    <p><%= article.content %></p>
  </article>
<% end %>
```

You can now launch your local server `rails s` and webpack server `webpack-dev-server` and see on the homepage the list of 100 articles we just created!

We're now ready to add a very simple pagination as a second step.

## Adding pagination without the infinite scroll

For the pagination, we will use a very simple gem created by the Basecamp team called [geared pagination](https://github.com/basecamp/geared_pagination). It is very small (less than 50 commits at the time I write this article) and very well written.

Let's add the gem to our Gemfile and install it. Don't forget to restart your server after that!

```bash
bundle add geared_pagination
bundle install
```

Using the gem is very easy, we just have to use the `set_page_and_extract_portion_from` method in the controller like this:

```rb
# app/controllers/articles_controller.rb

class ArticlesController < ApplicationController
  def index
    # Note that we specify that we want 10 articles per page here with the
    # `per_page` option
    @articles = set_page_and_extract_portion_from Article.all, per_page: [10]
  end
end
```

In the view, we simply have to add the pagination logic at the end of the page:

```html
<!-- app/views/articles/index.html.erb -->

<h1>Articles#index</h1>

<% @articles.each do |article| %>
  <article>
    <h2><%= article.title %></h2>
    <p><%= article.content %></p>
  </article>
<% end %>

<% unless @page.last? %>
  <%= link_to "Next page", root_path(page: @page.next_param) %>
<% end %>
```

The pagination works! Click on the next page link to see the page changing. But that's not what we want! What we want is an infinite scroll and that's the most interesting part of this article!

## Adding the infinite scroll pagination with Stimulus

The infinite scroll will work as follow:

1. Every time the viewport intersect the hidden next page link, we will trigger an AJAX request to get the additional articles
2. We will then append those articles to the list and replace the current next page link with the next one
3. We will then repeat the process until we reach the last page!

Are you ready? Let's go!

First, let's create a pagination controller with Stimulus and connect it to our articles index page.

Let's add a `nextPageLink` target and log it in the console when the controller initialized.

```js
// app/javascript/controllers/pagination_controller.js

import { Controller } from "stimulus"

export default class extends Controller {
  static targets = ["nextPageLink"]

  initialize() {
    console.log(this.nextPageLinkTarget)
  }
}
```

To make it work, we also need to update our HTML by adding `data-controller="pagination"` to the articles list and `data-pagination-target="nextPageLink"` to the next page link. Our index code now looks like this:

```html
<!-- app/views/articles/index.html.erb -->

<div data-controller="pagination">
  <% @articles.each do |article| %>
    <article>
      <h2><%= article.title %></h2>
      <p><%= article.content %></p>
    </article>
  <% end %>

  <% unless @page.last? %>
    <%= link_to "Next page",
                root_path(page: @page.next_param),
                data: { pagination_target: "nextPageLink" } %>
  <% end %>
</div>
```

Refresh your page and you should now see the next page link logged into your console.

Now that everything is wired correctly, we are ready to add our feature. The first thing we are going to do, is to `console.log("intersection")` when the viewport intersects the next page link.

How do you do this?

With a Javascript object called `IntersecionObserver`! The [Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) provides a way to asynchronously observe changes in the intersection of a target element with an ancestor element or with a top-level document's viewport.

Let's add this in our Stimulus controller:

```js
// app/javascript/controllers/pagination_controller.js

import { Controller } from "stimulus"

export default class extends Controller {
  static targets = ["nextPageLink"]

  initialize() {
    this.observeNextPageLink()
  }

  // private

  async observeNextPageLink() {
    if (!this.nextPageLinkTarget) return

    await nextIntersection(this.nextPageLinkTarget)
    console.log("intersection")
  }
}

const nextIntersection = (targetElement) => {
  return new Promise(resolve => {
    new IntersectionObserver(([element]) => {
      if (!element.isIntersecting) {
        return
      } else {
        resolve()
      }
    }).observe(targetElement)
  })
}
```

Wow! That's the most complicated part of the feature! Let's break it down.

First, when the Stimulus controller is initialized, we start observing the next page link.

```js
initialize() {
  this.observeNextPageLink()
}
```

If there is no next page link on the page, then the controller does nothing. However, if there is a next page link, we'll wait for the intersection and then `console.log("intersection")`. Note that this process is asynchronous: we don't know when the next intersection is going to happen!

How do we do asynchronous Javascript? With async / await and promises!

The function `observeNextPageLink` is asynchronous for this reason. See how it reads like plain English now? Wait for the next intersection with the next page link and then `console.log("intersection")`.

```js
async observeNextPageLink() {
  if (!this.nextPageLinkTarget) return

  await nextIntersection(this.nextPageLinkTarget)
  console.log("intersection")
}
```

Last but not least, the `nextIntersection` function has to return a `Promise` that will resolve when the next page link intersects the viewport. This can be done easily by creating a new `IntersectionObserver` that will observe the next page link.

```js
const nextIntersection = (targetElement) => {
  return new Promise(resolve => {
    new IntersectionObserver(([element]) => {
      if (!element.isIntersecting) {
        return
      } else {
        resolve()
      }
    }).observe(targetElement)
  })
}
```

Now that our mechanic is in place, we need to replace our `console.log("intersection")` with something useful. Instead of logging "intersection" in the console, we will fetch the articles from the next page and append them to the list of articles we already have!

To do AJAX requests with Rails, we will use the brand new [rails/request.js](https://github.com/rails/request.js) library that was created in June 2021. This library is a wrapper around `fetch` that you'll normally use to do AJAX requests in Javascript. It integrates nicely with Rails, for example, it automatically sets the `X-CSRF-Token` header that is required by Rails applications, this is why we'll use it!

Let's add it to our package.json using yarn:

```bash
yarn add @rails/request.js
```

Now let's import the `get` function in our Pagination Controller and replace the `console.log("intersection")` with the actual logic. The code now looks like this:

```js
import { Controller } from "stimulus"
import { get } from "@rails/request.js"

export default class extends Controller {
  static targets = ["nextPageLink"]

  initialize() {
    this.observeNextPageLink()
  }

  async observeNextPageLink() {
    if (!this.nextPageLinkTarget) return

    await nextIntersection(this.nextPageLinkTarget)
    this.getNextPage()
  }

  async getNextPage() {
    const response = await get(this.nextPageLinkTarget.href) // AJAX request
    const html = await response.text
    const doc = new DOMParser().parseFromString(html, "text/html")
    const nextPageHTML = doc.querySelector(`[data-controller~=${this.identifier}]`).innerHTML
    this.nextPageLinkTarget.outerHTML = nextPageHTML
  }
}

const nextIntersection = (targetElement) => {
  return new Promise(resolve => {
    new IntersectionObserver(([element]) => {
      if (!element.isIntersecting) {
        return
      } else {
        resolve()
      }
    }).observe(targetElement)
  })
}
```

The only changes here are the `import { get } from "@rails/request.js"` that we use to make a get AJAX request to our server and the `console.log("intersection")` that was replaced by `this.getNextPage()`. Let's understand this last method.

```js
async getNextPage() {
  const response = await get(this.nextPageLinkTarget.href) // AJAX request
  const htmlString = await response.text
  const doc = new DOMParser().parseFromString(htmlString, "text/html")
  const nextPageHTML = doc.querySelector(`[data-controller=${this.identifier}]`).outerHTML
  this.nextPageLinkTarget.outerHTML = nextPageHTML
}
```

First, we issue a get request to the server, wait for the response and store it in the `response` variable. Then we extract the text from the response and store it in the `htmlString` variable. As we want to use querySelector on this `htmlString`, we first need to parse it to make it an HTML document with `DOMParser`. We then store this document in the `doc` variable. We then extract the next page articles and the next page link from this document and append them to our articles list by replacing the current next page link.

Our infinite scroll is now working, but only for one iteration! We need to make it recursive. Every time new articles are added to the page, a new next page link is also added! We need to observe this new next page link to be able to have a read *infinite* scroll!

Let's add this recursion!

**Here is the final controller:**

```js
import { Controller } from "stimulus"
import { get } from "@rails/request.js"

export default class extends Controller {
  static targets = ["nextPageLink"]

  initialize() {
    this.observeNextPageLink()
  }

  async observeNextPageLink() {
    if (!this.nextPageLinkTarget) return

    await nextIntersection(this.nextPageLinkTarget)
    this.getNextPage()

    await delay(500) // Wait for 500 ms
    this.observeNextPagelink() // repeat the whole process!
  }

  async getNextPage() {
    const response = await get(this.nextPageLinkTarget.href)
    const html = await response.text
    const doc = new DOMParser().parseFromString(html, "text/html")
    const nextPageHTML = doc.querySelector(`[data-controller~=${this.identifier}]`).innerHTML
    this.nextPageLinkTarget.outerHTML = nextPageHTML
  }
}

const delay = (ms) => {
  return new Promise(resolve => setTimeout(resolve, ms))
}

const nextIntersection = (targetElement) => {
  // Same as before
}
```

Here, we only changed the two last lines of the `observeNextPageLink` function by waiting 500ms to avoid scrolling too fast, and then, we observe the new next page link if there is one, thus repeating the whole process we just went through!

The last think you can do is hide the next page link on the page to make it a real infinite scroll.

```html
<% unless @page.last? %>
  <%= link_to "Next page",
              root_path(page: @page.next_param),
              data: { pagination_target: "nextPageLink" },
              style: "visibility: hidden;" %>
<% end %>
```

You did it, you built a real infinite scroll with Rails and Stimulus!

## Takeaways and useful resources

- [rails/request.js](https://github.com/rails/request.js) is a library that provides a wrapper around fetch. It is very useful when working with Rails applications because it sets a few headers under the hood for you that are required by your Rails application.
- [basecamp/gearder_pagination](https://github.com/basecamp/geared_pagination) is a very small pagination gem (less than 50 commits). You should read the code if you want to learn a few tricks in Ruby / Rails!
- When working with asynchronous actions in Javascript, you should work with promises and async / await. The [Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) can help you trigger actions based on the viewport intersecting other elements on the page.

## Credits

This article is heavily inspired by [hey.com](https://www.hey.com/index.html)'s infinite scroll. Thanks to the Basecamp team for [leaving the source maps open](https://m.signalvnoise.com/paying-tribute-to-the-web-with-view-source/). It was really helpful when I had to build a similar feature!

## Did you like this article?

You can [follow me on Twitter](https://twitter.com/alexandre_ruban) to get notified when I publish new articles. I sometimes do when I work on interesting features like this infinite scroll!
