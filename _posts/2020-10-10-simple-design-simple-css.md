---
title: Achieving simple design with simple CSS
description: This description will be used in the meta tags
date: 2020-10-10
minutes_read: 15
tags: css
---

A few weeks ago, I came across [this article][signal-v-noise-article] from the Basecamp team explaining how they achieve *"simple design"* in both their products Basecamp and Hey. They mention what they call "Fisher Price" design (always choosing clarity over being slick or fancy). I am a big fan and user of both Basecamp and Hey. I love how simple and straight to the point both products are and I kept wondering what the impacts on the CSS and HTML code are. Let's have a look!

To prepare this article, I rebuilt the ["How it works"][hey-how-it-works] page of Hey's marketing website. Here is what I learned.

## You need fewer variables

If you use open source CSS libraries like Bootstrap in some of your web projects, then you know what a huge scss variables file looks like. If you have a look at the [variables file of Bootstrap][bootstrap-variables] you will notice that this file alone is about 1300 lines long! This is a huge amount of work compared to the 50 variables extracted from the Hey source code. If you are working with a small team and want to build a custom design rebuilding a Bootstrap like design system will probably take you ages.

Let's have a closer look at the Hey variables:

```css
:root {
  /* Colors */
  --color-white: #fff;
  --color-black: #222;
  --color-grey: #777471;
  --color-stone: #edeae6;
  --color-yellow: #f5d652;
  --color-canary: #fff5ca;
  --color-canary-tint: #fffae5;
  --color-cobalt: #0074e4;
  --color-sky: #b6dbff;
  --color-sky-tint: #daedfe;
  --color-blurple: #5522fa;
  --color-lavender: #d5d2ff;
  --color-lavender-tint: #eae8fe;
  --color-purple: #7700a2;
  --color-pink: #cb84de;
  --color-pink-tint: #efdaf5;
  --color-salmon: #ec8580;
  --color-peach: #fee5da;
  --color-peach-tint: #fef2ed;
  --color-teal: #5fddc5;
  --color-mint: #b3f4e0;
  --color-mint-tint: #d8f9f0;

  /* Concrete colors */
  --color-shadow: rgba(0, 0, 0, 0.1);
  --color-shadow-dark: rgba(0, 0, 0, 0.2);
  --color-text: var(--color-black);
  --color-background: var(--color-stone);
  --color-sheet: var(--color-white);
  --color-neutral: var(--color-grey);
  --color-link: var(--color-blurple);
  --color-accent: var(--color-teal);

  /* Spaces modular scale */
  --space-x-small: 0.25em;
  --space-small: 0.5em;
  --space-medium: 1em;
  --space-large: 2em;
  --space-x-large: 3em;
  --space-xx-large: 4em;
  --space-xxx-large: 6em;
  --space-navbar: 4em;

  /* Font families */
  --font-sans: 'Source Sans Variable', 'Helvetica Neue', helvetica, 'Apple Color Emoji', arial, sans-serif;
  --font-mono: monospace;

  /* Font size modular scale */
  --type-base: calc(1.6em + 0.5vw);
  --type-xxx-small: 55%;
  --type-xx-small: 65%;
  --type-x-small: 75%;
  --type-small: 85%;
  --type-medium: 100%;
  --type-large: 125%;
  --type-x-large: 150%;
  --type-xx-large: 200%;
  --type-xxx-large: 300%;
}

html {
  font-size: 10px;
}

body {
  font-size: var(--type-base);
}
```

Out of 50 variables, 30 correspond to the colors of the palette, 2 correspond to the fonts, 8 correspond to spaces used for both margin and paddings and 10 correspond to the different font sizes the designer can choose. That's a much smaller variable file! And it is more than enough to create a "simple design".

There are two more interesting things in this variable file:

First, they use a responsive font size. The document font size is set to `10px` in the `html` tag (so 1 rem is always 10px) and then overriden to `var(--type-base)` in the `body` tag. Having a closer look at the `--type-base` variable, we can notice that it is set to `calc(1.6em + 0.5vw)` which means 1.6 \* 10px + 0.5% of the viewport width. This means that the font size will scale with the viewport width and so the `em` also scales with it. In other words, the font size on smaller screen will be smaller than on bigger screens. The font size is changed with media queries for the two breakpoints `45em` and `91em` globally by overriding the `--type-base` variable.

```css
@media (min-width: 45em) {
  :root {
    --type-base: calc(0.9em + 0.9vw);
  }
}

@media (min-width: 91em) {
  :root {
    --type-base: 2.2em; /* font size stops scaling after 91em */
  }
}
```

Second, the unit of the spaces variables is `em` and not `rem`. This means that the spacings also scale with the viewport width. This works the same as for the font size, the spaces will be smaller on smaller screens than on bigger screens. Rather than hardcoding pixels, your design will keep the same *proportions* on different devices. We will see another use of this feature shortly so keep in mind that spaces for margins and paddings are in `em`, this is a great feature of CSS! If you are still using pixels for your spaces units, you should consider switching to ems!

## You need a simpler layout

Now that we have our set of variables in place let's talk about the layout. Hey uses CSS grid to create a *really simple* layout. If you're not familiar with CSS grid, now is the time to learn it's powers! (Note that the indentation of the grid-template-columns property is only there for readability purposes on this blog but is not required).

```css
.grid {
  display: grid;
  grid-template-columns: [max] 1fr
    [l] 1fr
    [m] repeat(3, [m] 28vw)
    [m] 1fr
    [l] 1fr
    [max];
}
```

These 2 lines of code above divides the screen into 7 columns. The 8 lines that divides those 7 columns are named in order `max`, `l`, `m`, `m`, `m`, `m`, `l` and `max` thanks to the `[]` notation (otherwise they would simply be numbered 1 to 8). Out of those 7 columns, 4 of them have a width of `1fr` and the 3 center columns have a width of `28vw`. The `fr` unit stands for *fraction* and it really means a fraction of the available space. The `vw` unit means 1% of the viewport width. If we go further here, the 3 columns in the center will each take `28vw` for a total of `84vw`. There are `100vw` available (100% of the viewport width) so `16vw` to split in 4 fractions means that each fraction is 4% of the viewport width! If this feels a bit complicated, it is illustrated in the picture bellow:

![hey.com CSS grid](/assets/images/grid.png)

The sizes of those 7 columns are overriden for the two breakpoints `45em` and `91em` as shown bellow.

```css
@media(min-width: 45em) {
  .grid {
    grid-template-columns: [max] 1fr
      [l] minmax(11ch, 0.7fr)
      [m] repeat(3, [m] 21.66ch)
      [m] minmax(11ch, 0.7fr)
      [l] 1fr
      [max];
  }
}

@media (min-width: 91em) {
  .grid {
    grid-template-columns: [max] 1fr
      [l] 15ch
      [m] repeat(3, [m] 21.66ch)
      [m] 15ch
      [l] 1fr
      [max];
  }
}
```

The most important thing to learn here is this `minmax` function. Here `minmax(11ch, 0.7fr)` simply means that the column width can shrink until it reaches a width of `11ch` otherwise it will have a width of `0.7fr`. Easy right? This minmax notation is widely used with CSS grid!

We can also learn about 1 more spacing unit called `ch` which corresponds to the width of the zero character (0) in the current font size. It is usually used for paragraph widths as paragraphs are considered very readable at about `70ch` width. This is what I use for this blog!

Now that we have our 7 columns layout, it's time to define CSS classes to fill those columns.

```css
.grid__max {
  grid-column: max / max 2;
}

.grid__wide {
  grid-column: l / l 2;
}

.grid__medium {
  grid-column: m / m 4;
}

.grid > :not([class*="grid"]) {
  grid-column: m / m 4;
}
```

In a grid layout, the element that defines the columns with the property `grid-template-columns` is called the grid container. Its direct descendants are called the grid items and their place on the grid is determined by the property `grid-column`.

The first class `.grid__max` means that the element it is applied to will fill the space from the first grid column named `max` to the second grid column named `max`. If you have a look at the picture of the grid above this basically means the whole width of the grid container.

The same thing applies for the `.grid__wide` class as the element it is applied to will fill the space from the first grid column named `l` to the second grid column named `l`. In the picture above, you can see that the hero picture received this `.grid__wide` class.

Last but not least, the elements with `.grid__medium` class will fill the space from the first grid column named `m` to the fourth grid column named `m`. That's the 3 center columns of the grid and if you have a look at the picture again, this is where the text lives.

Does it mean that we need to specify `.grid__max`, `.grid__wide` or `.grid__medium` on every single direct descendant of the grid container? No! Thanks to the last selector `.grid > :not([class*="grid"])` if a direct descendent of the grid container (a grid item) has no class that contains the string `grid` it will act like a `.grid__medium` by default! Neat!

If you visit [hey.com][hey-website] you will realise that with this set of variables, and this layout in place, almost all of the heavy lifting is already done. This is a very good example of "simple design".

Note that the grid is not completely supported on Internet Explorer. The Hey team wants to use grid on their site, so they will "force" users to upgrade to a more recent browser using the media query `@supports (display: grid)`. Let's have a look at how they do it:

```html
<div class="unsupported">
  <p>
    <strong>
      Sorry, this website uses features that your browser doesn’t support.
    </strong>
    Upgrade to a newer version of
    <a href="https://www.mozilla.org/en-US/firefox/new/">Firefox</a>,
    <a href="https://www.google.com/chrome/">Chrome</a>,
    <a href="https://support.apple.com/downloads/safari">Safari</a>,
    or
    <a href="https://www.microsoft.com/en-us/edge">Edge</a>
    and you’ll be all set.
  </p>
</div>
```

First, they add HTML markup to suggest that the user upgrades his browser to a newer version.

```css
.unsupported {
  position: fixed;
  width: 100%;
  height: 100%;
  top: 0;
  padding: 4em 2em;
  z-index: 10000;
  font-size: 24px;
  font-weight: bold;
  text-align: center;
  color: #000;
  background-color: #fff
}

.unsupported p {
  max-width: 800px;
  margin: 0 auto
}

/* Don't show the '.unsupported' element if the browser supports grid */
@supports (display: grid) {
  .unsupported {
    display:none;
    visibility: hidden;
  }
}
```

Then they add css to make this suggestion visible only to users that does not support grid thanks the media query `@supports (display: grid)`.

## Use BEM or another convention

CSS is often a misunderstood language. There is a lot to learn and it can become a real mess very quickly it you don't have clear rules about what should go where. The Basecamp team uses the [BEM methodology][bem-website] to organise CSS code. Let's have a look at how BEM methodology helps you organise your code by redesigning those boxes.

![How it works boxes](/assets/images/hiw-boxes.png)

First let's have a look at the corresponding HTML (notice the `.grid__wide` from previous paragraph!):

```html
<section class="hiw-box grid__wide">
  <div class="hiw-box__copy txt-center-mobile">
    <h3 class="push-small--top">
      The Imbox is for your important email
    </h3>
    <p>
      When you say “Yes” to someone in The Screener, their email lands in
      the Imbox by default. It’s the place for emails you actually want to
      read, from people and services you absolutely want to hear from.
    </p>
  </div>
  <figure class="hiw-box__image">
    <img src="imbox.jpg" alt="The Imbox">
  </figure>
</section>

<section class="hiw-box hiw-box--blurple hiw-box--reverse grid__wide">
  <div class="hiw-box__copy txt-center-mobile">
    <h3 class="push-small--top">
      The Feed is for your casual, whenever reads
    </h3>
    <p>
      The Feed turns your newsletters, promotional emails, and long-reads into
      a browsable, casual newsfeed. Just scroll, everything’s open already. See
      something you like? Click it and read the whole thing right in place.
    </p>
  </div>
  <figure class="hiw-box__image">
    <img src="feed.jpg" alt="The Feed">
  </figure>
</section>
```

Did you notice the structure? `hiw-box` is the name of the block. It represents the whole card. Then `hiw-box__copy` and `hiw-box__image` are elements of this block. They are prefixed with the name of the block to prevent naming conflicts in CSS. The block name and the element name are separated with a double underscore. Finally `hiw-box--blurple` and `hiw-box--reverse` are called modifiers. They are also prefixed by the name of the block. The block name and the element name are separated with a double "-". Et voilà, you know almost all there is to know about BEM methodology.

Let's have a look at the corresponding CSS:

```css
/* Block */
.hiw-box {
  position: relative;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(17em, 1fr));
  grid-gap: 1em;
  align-items: center;
  direction: rtl;
  margin: 1em 0;
  padding: 1em;
  color: var(--color-cobalt);
  background-color: var(--color-canary);
  border-radius: 0.75em;
}

@media (min-width: 45em) {
  .hiw-box {
    left:-1em;
    transform: rotate(-1deg);
    background: var(--color-canary) url(teal.svg) left -7em center /
      auto 125% no-repeat;
  }
}

/* Elements of the block */
.hiw-box__copy {
  direction: ltr;
  margin: 0 1em;
}

.hiw-box__image {
  margin: 0;
  background-color: var(--color-white);
  box-shadow: 2rem 2rem 6rem var(--color-shadow);
  border-radius: 1rem;
}

/* Modifiers */
@media (min-width: 45em) {
  .hiw-box--reverse {
    left:1em;
    direction: ltr;
    transform: rotate(1deg);
  }
}

.hiw-box--blurple {
  color: var(--color-peach);
  background-color: var(--color-blurple);
}

@media (min-width: 45em) {
  .hiw-box--blurple {
    transform:rotate(1deg);
    background: var(--color-blurple) url(pink-purple.svg) right -6em center /
      auto 126% no-repeat;
  }
}
```

Do you see how easy it is to read? BEM makes the HTML markup very readable and, as it is a popular methodology, a new developper joining your team will understand immediately how to organise CSS in the project.

## Use the full power of CSS to write less code

Remember when I said spacing should be in ems? Now is the time to explain why a bit more in depth. If you have a closer look at [hey.com][hey-website], you will realize that buttons have many different sizes. However in the source code, you'll find only one `.button` class

```css
.button {
  transition: color 0.2s ease, background-color 0.2s ease, box-shadow 0.2s ease;
  display: inline-block;
  padding: 0.4em 1em;
  color: var(--color-sheet);
  font-family: var(--font-sans);
  text-decoration: none;
  text-align: center;
  line-height: normal;
  -webkit-appearance: none;
  background-color: var(--color-link) !important;
  background: linear-gradient(
    90deg, var(--color-link) 0%, var(--color-cobalt) 100%
  );
  border-radius: 1.5em;
  border: 0;
  box-shadow: none;
}
```

The interesting thing to notice here is that padding units are in em which means relative to the element's font size. If you have utilities for font size in you css classes, scaling the size of the button becomes really easy.

```html
<a href="#" class="button txt-small">Small button</a>
<a href="#" class="button">Medium button</a>
<a href="#" class="button txt-large">Large button</a>
```

This of course is a small trick but combined with all the others, it could save a lot of time for you or your team!

## Build amazing animations without Javascript

Hey's navigation bar is amazing and implemented in pure CSS. In this section, we are going to rebuild it and hopefully learn a few more tricks!

![Hey navigation bar](/assets/images/hey-navbar.gif){:class="gif"}

Let's start with the HTML markup:

```html
<header class="header">
  <nav class="header__nav" aria-label="Mail navigation">
    <a href="#" class="header__logo" aria-label="HEY homepage">
      Hey log
    </a>
    <input
      type="checkbox"
      role="button"
      id="hamburger"
      class="header__checkbox hide-screens"
    >
    <label class="header__toggle" for="hamburger">
      <span class="header__toggle-icon" aria-hidden="true"></span>
      Menu
    </label>

    <div class="header__links" role="menu">
      <a class="header__link" href="#" role="menuitem">How it works</a>
      <a class="header__link" href="#" role="menuitem">Tour features</a>
      <a class="header__link" href="#" role="menuitem">Pricing</a>
      <a class="header__link" href="#" role="menuitem">FAQs</a>
      <a class="header__link" href="#" role="menuitem">Sign in</a>
      <a class="header__link button" href="#" role="menuitem">Try it FREE</a>
    </div>
  </nav>
</header>
```

The markup again is very simple here and very readable thanks to BEM methodology. The thing to notice here is that the "Menu" toggle is nested inside a label that corresponds to the hidden checkbox just above. When you click on "Menu", you are actually clicking on the label that checks and unchecks the checkbox. With this technique, we will be able to add styles to the siblings of the checkbox using the general sibling selector `~` in CSS.

Let's have a look at how it works in detail. First, let's setup the general styles for the navigation bar.

```css
.header {
  position: fixed;
  width: 100%;
  z-index: 1000;
  background-color: var(--color-sheet);
}
```

The `.header` block is fixed at the top of the page.

```css
.header__nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  position: relative;
  z-index: 20000;
  margin: 0;
  padding: 0.75rem;
}

@media (min-width: 45em) {
  .header__nav {
    padding: 1rem;
  }
}

.header__logo {
  margin: 0;
  text-align: left;
}
```

Next, we created the inner container in the navigation bar. It's a flex container. On larger screens, the `justify-content: space-between;` makes sure the logo and the navigation links are each on one side of the screen. Let's have a look at the interesting part: the `.header__links` element.

```css
.header__links {
  display: block;
  margin: 0;
  padding: 0;
  font-size: var(--type-medium);
}

@media (max-width: 44.99em) {
  /* They are also stacked vertically */
  /* Links are absolutely positioned outside of the screen */
  .header__links {
    position:absolute;
    top: 8rem;
    right: -25rem;
    display: flex;
    flex-direction: column;
    align-items: flex-end;
    margin: 0;
    padding: 0;
    font-size: var(--type-large);
    -webkit-overflow-scrolling: touch;
  }

  /* Creates the white backdrop when the navigation bar is opened */
  .header__links::after {
    transition: opacity 0.15s ease-in-out;
    content: '';
    position: fixed;
    z-index: -1;
    top: 0;
    right: 0;
    bottom: 0;
    left: 0;
    opacity: 0;
    pointer-events: none;
    background-color: rgba(255, 255, 255, 0.8);
  }
}
```

On a large screen, the wrapper around the navigation links is just a simple block.

On smaller screens, on the other hand, the header links are stacked vertically thanks to `display: flex;` and `flex-direction: column;` and aligned to the right thanks to `align-items: flex-end;`. They are also in `position: absolute;` relatively to the closest parent with `position: relative;` which in this case is the `.header__nav` element. This allows to position those links at 8rem from the top and -25rem from the right (that's outside the screen!). We'll see soon how checking the checkbox can animate those links so that they move inside the screen.

The `.header__links::after` pseudo element is the white transparent backdrop. The `position: fixed;`, `top: 0;`, `right: 0;`, `left: 0;` and `bottom: 0;` properties makes sure this pseudo element takes the full screen and `z-index: -1;` that it appears behind the navigation links.

Now let's have a look at how the animation works by having a look at the `.header__link` itself:

```css
/* On large screens: classic inline-block link */
.header__link {
  transition: color 0.2s ease,
    text-decoration-color 0.2s ease,
    box-shadow 0.2s ease;
  transform: none;
  margin: 0 0 0 0.6em;
  display: inline-block;
  line-height: 1.5;
  font-weight: 700;
}

@media (max-width: 44.99em) {
  /* On small screens: classic inline block link with more style */
  .header__link {
    transition:transform 0.15s ease-in;
    transform: translate(0, 0);
    display: inline-block;
    padding: 0.2em 1em;
    margin: 0 0 0.25em 0;
    line-height: 2;
    text-decoration: none;
    color: var(--color-white);
    background-color: var(--color-teal);
    background: linear-gradient(
      90deg, var(--color-purple) 0%, var(--color-salmon) 100%
    );
    border-radius: 3em;
  }

  .header__link:visited,
  .header__link:hover {
    color: var(--color-white);
    box-shadow: none;
  }

  /* The animation is delayed 0.05 sec for the second header__link */
  .header__link:nth-child(2) {
    transition-delay: 0.05s;
  }

/* The animation is delayed 0.1 sec for the second header__link */
  .header__link:nth-child(3) {
    transition-delay: 0.1s;
  }

/* And it goes on like this + 0.05 sec for the next ones */
  .header__link:nth-child(4) {
    transition-delay: 0.15s;
  }

  .header__link:nth-child(5) {
    transition-delay: 0.2s;
  }

  .header__link:nth-child(6) {
    transition-delay: 0.25s;
  }

  .header__link:nth-child(7) {
    transition-delay: 0.3s;
  }

  .header__link:nth-child(8) {
    transition-delay: 0.35s;
  }
}
```

On large screens, a `.header__link` is a simple inline block element.

On smaller screens, it has a bit more decoration on it with a linear gradient as a background, but the really interesting part here is the transition delay. For each `.header__link`, the transition on the position will start `0.05s` after the previous `.header__link` thanks to the `:nth-child(n)` selector!

And now let's have a look at how to trigger this animation:

```css
@media (max-width: 44.99em) {
  /* When the checkbox is :checked, show the white backdrop */
  .header__checkbox:checked ~ .header__links::after {
    opacity:1;
    pointer-events: auto;
  }

  /* When the checkbox is :checked, translate the links from -27rem! */
  .header__checkbox:checked ~ .header__links .header__link {
    transition-duration: 0.25s;
    transition-timing-function: cubic-bezier(0.34, 1.56, 0.64, 1);
    transform: translate(-27rem, 0);
  }
}
```

Thanks to the `:checked` pseudo-class and to the general sibling combinator `~` we can instruct our links to translate when clicking on the label. You have a beautiful animation without a single line of Javascript. I will write about how to make an animated hamburger menu in another article.

## Takeways

I had a lot of fun digging into [hey.com][hey-website] marketing website. Here are the main takeways from this blog post summarized:

- Simple design requires very few lines of CSS. The whole [hey.com][hey-website] marketing website requires a bit more than 2000 lines of CSS (reboot and variables included) which is very small. It also has a very positive impact on the performance of the website.
- Your set of variables does not have to be more than 100 lines to build your design system.
- You do not need complex layouts to sell your product. A simple design and good copywriting will be more effective than "the three column layout".
- Using a simple methodology like [BEM][bem-website] makes it really easy to organise CSS and makes your HTML markup readable.
- You can build amazing animations with CSS alone. You might not need Javascript for everything.
- The Basecamp's team approach is a great inspiration for small teams and show how to leverage the full power of CSS's features to write very little code.

[signal-v-noise-article]: https://m.signalvnoise.com/how-we-achieve-simple-design-for-basecamp-and-hey/#comments
[hey-how-it-works]: https://hey.com/how-it-works/
[bootstrap-variables]: https://github.com/twbs/bootstrap/blob/main/scss/_variables.scss
[hey-website]: https://hey.com/
[bem-website]: https://en.bem.info/methodology/
