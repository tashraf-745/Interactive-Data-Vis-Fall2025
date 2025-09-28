---
title: "Lab 0: Getting Started"
toc: true
---

This page is where you can iterate. Follow the lab instructions in the [readme.md](./README.md).


## Writing HTML in Markdown
<!-- simple divs -->
<div>
  <p>You can write pure HTML in <a href="https://commonmark.org/">markdown files!</a></p>
</div>

<!-- hidden context -->
<details>
  <summary>Click me</summary>
  This text is not visible by default.
</details>

<!-- css grid styles -->
<div class="grid grid-cols-4">
  <div class="card"><h1>A</h1></div>
  <div class="card"><h1>B</h1></div>
  <div class="card"><h1>C</h1></div>
  <div class="card"><h1>D</h1></div>
</div>

<!-- leveraging pre-made cards -->
<div class="grid grid-cols-4">
  <div class="card">
    ${resize((width) => `This card is ${width}px wide.`)}
  </div>
</div>

## Writing JS in Markdown
Use JavaScript to render charts, inputs, and other dynamic, interactive, and graphical content on the client. JavaScript in Markdown can be expressed either as fenced code blocks or inline expressions. You can also import JavaScript modules to share code across pages.

You can write javascript in js blocks... 
<!-- function definition -->
```js
const addition = (x, y) => x + y;
```

...which won't show up in the dashboard unless referred to in its own block...
<!-- function reference -->
```js
addition
```

<!-- display -->
...or leveraging the `display` function.
```js
display(addition)
display(addition(2, 3))
```

There are some helpful functions already baked into Observable Framework, like now, width, or the responsive display. 
`now`:
```js
now
```

`width`:
```js
width
```

`resize()`:
```js
resize((width) => `I am ${width} pixels wide.`)
```