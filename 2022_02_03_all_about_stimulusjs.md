Notes from the [official handbook](https://stimulus.hotwired.dev/handbook/origin)

## The Origins of Stimulus

- No to making javascript "applications". 
- All of Basecamp has server-side rendered HTML
- Supports multiple platforms this way, including native mobile apps. With a single set of models/controllers/views. Only have to update in one place this way
- Don't want to confine server-side application to producing JSON for the javascript-based client application to consume.
- But a benefit of single-page JavaScript application *is* the faster, more fluid user interfaces (set free from full-page refresh). Turbo and Stimulus were designed to get those benefits (without going full client-side rendered route)
- Stimulus use the browser's [MutationObserver API](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) to detect DOM chagnes.

### Where Turbo fits in

"Turbo up high, Stimulus down low"

- Turbo descends from pjax (from github)
- The reason full-page refreshes often feel slow is **not** because the browser has to process a bunch of HTML sent from a server. No, browsers are fast at that. It's because CSS and JavaScript has to be reinitialized and reapplied to the page again. Even if the files are cached (so no network cost but still page performance cost)
- So Turbo's job then is to get around this reinitialization. It does this by maintaining a persistent process (similar to SPA, but an 'invisible one). It intercepts links and loads new pages via Ajax. Server returns fully formated HTML documents.
- Javascript (on-the-page) still needed for modern web app behaviors like show/hide elements, add item to a todo list, etc. Basecamp did this in inconsistent ways before with JQuery, vanilla JS, etc. All with explicit event handling hanging off a `data-behavior` attribute. 

So made Stimuls to bring consistency to this Javascript.

### Core concepts of Stimulus

Controllers, actions, and targets are the 3 core concepts

```html
<div data-controller="clipboard">
  PIN: <input data-clipboard-target="source" type="text" value="1234" readonly>
  <button data-action="clipboard#copy">Copy to Clipboard</button>
</div>
```

Some nice things about the above code:
- You can get an idea of what's going on by looking at the HTML alone, without looking at the `clipboard` controller code. (This is different from other HTML where an external JS file applies event handlers to it.)
- Stimulus does not bother itself by *creating* the HTML. That's still rendered on the server either on page load (first hit or via Turbo) or via Ajax request that changes the DOM.
- Stimulus is concerned with manipulating the existing HTML document. By adding a CSS class that hides, animates, highlights an element. By rearranging elements in groupings. By manipulating content of an element like UTC times that can be cached into local times and displayed.
- Stimulus *can* create new DOM elements and that's allowed. But that's minority case. The focus is on manipulating not creating elements.

How Stimulus differs from mainstream JavaScript frameworks:
- Other frameworks are focused on turning JSON into DOM elements via template language
- Other frameworks maintain *state* within JavaSripts objects.
For Stimulas, state is stored in the HTML, so that controllers can be discarded between page changes, but still reinitialize as they were when the cached HTML appears again. [question mark]

The approach of Stimulus + Turbo makes sense in many cases. But client-side rendering is sometimes called for and Basecamp does use it too. 

> At Basecamp, we have and do use several heavier-duty approaches when the occasion calls for it. Our calendars tend to use client-side rendering. Our text editor is Trix, a fully formed text processor that wouldn’t make sense as a set of Stimulus controllers.

## Introduction

Stimulus is designed to enhance *static* or *server-rendered* HTML by connecting JavaScript objects to elements on the page using simple annotations.

These JavaScript objects are called *controllers* and Stimulus monitors the page waiting for HTML `data-controller` attributes to appear. Each attribute's value is a controller class name. Stimulus finds that class, creates a new instance of that class and connects it to the element. 

Just like `class` attribute is a bridge connecting HTML to CSS. `data-controller` attribute is a bridge connecting HTML to JavaScript.

In addition to controllers, 3 other major Stimulus concepts are:

actions - which connect controller methods to DOM events using `data-action` attributes
targets - which locate elements of significance within a controller [question mark]
values - which read/write/observe data attributes on the controller's element [question mark]

## Hello, Stimulus

This was an exercise to print a greeting when user clicks a button, along with the name that was typed into a text box. Demonstrates how *actions* and *targets* are used in the code pretty well.

```html
<body>
  <div data-controller="hello">
    <input data-hello-target="name" type="text">
    <button data-action="click->hello#greet">Greet</button>
  </div>
</body>
```

The **`data-controller`** connects this HTML to a class in hello_controller.js file. (and Stimulus also auto initializes this controller object).

The **`data-action`** means when this button is clicked, execute the code inside the `greet` action/method of the `hello` controller.

The value `click->hello#greet` is called an *action descriptor*.

I noticed that it works without the `click->` part, so just `data-action="hello#greet"` works too. Some Stimulus syntacic sugar. 

The **`data-[controller-name]-target`** is a way to connect this HTML element to the controller such that it's value can be accessed inside the controllr. In this case `data-hello-target`. This is what the code looks like inside `hello_controller.js`:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  
  static targets = [ "name" ]
  
  greet() {
    const element = this.nameTarget
    const name = element.value
    console.log(`hello, ${name}!`)
  }
}
```

We create a property for the target by adding `name` to our controller’s list of target definitions. Stimulus *will automatically create* a `this.nameTarget` property which returns the first matching target element. We can use this property to read the element’s `value` and build our greeting string.

## Building Something Real - Copy to Clipboard

The HTML looks like this:
```html
<body>
  Example: Copy To Clipboard
  <div data-controller="clipboard">
    PIN: <input data-clipboard-target="source" type="text" value="1234" readonly>
    <button data-action="clipboard#copy">Copy to Clipboard</button>
  </div>
  More than once instance of the clipboard controller on the page
  <div data-controller="clipboard">
    PIN: <input data-clipboard-target="source" type="text" value="5678" readonly>
    <button data-action="clipboard#copy">Copy to Clipboard</button>
  </div>
  Use other HTML elements like link and textarea (instead of button and input)
  <div data-controller="clipboard">
    PIN: <textarea data-clipboard-target="source" readonly>3737</textarea>
    <a href="#" data-action="clipboard#copy" class="clipboard-button">Copy to Clipboard</a>
  </div>
</body>
```

The `clipboard_controller.js` looks like this:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  
  static targets = [ "source" ]

  //v1 - with a button, using the browswer Clipboard API
  copy_old() {
     navigator.clipboard.writeText(this.sourceTarget.value)
  }

  //v2 - copy action attached to <a> link, input from a <textarea>
  copy(event) {
    event.preventDefault()
    this.sourceTarget.select()
    document.execCommand("copy")
  }
```

Some interesting things to learn from the above example:

**What does the `static targets` line do?**

When Stimulus loads our controller class, it looks for a static array with the name `targets`. For each target name in the array, Stimulus adds three new properties to our controller. For the "source" target name above, we get these 3 properties -- `this.sourceTarget`, `this.sourceTargets`, and `this.hasSourceTarget`

**We can instantiate the same controller more than once on a page**

Stimulus controllers are reusable. Any time we want to provide a way to copy a bit of text to the clipboard, all we need is the markup on the page with the right `data-` annotations. And it just works.

In the HTML above, we have the exact same `div` for copying PINs duplicated twice. The 2nd copy has a different value so we can test that both the copy button just work and copy the right thing. The thing that's implicit here is that we have two different instances of the controller class, and each instance it's own `sourctTarget` property with the correct `value`. This is how we keep them separate the copy the corresponding value (and don't get the values mixed up with the other `input` element annotated with `data-clipboard-target="source"` on the page. It's because the *controller* is scoped to the `<div>`

This implies that if we put *two* buttons inside the same `<div>`, things would not work as expect. The below will always copy the value in the *first* text box:

```html
<div data-controller="clipboard">
    PIN: <input data-clipboard-target="source" type="text" value="1234" readonly>
    <button data-action="clipboard#copy">Copy to Clipboard</button>
    PIN: <input data-clipboard-target="source" type="text" value="this won't get copied" readonly>
    <button data-action="clipboard#copy">Copy to Clipboard</button>
</div>
```

**Actions and Targets can go on any HTML elements**

So we don't have to use a `<button>` for the copy to clipboard functionality, we could also use a link `<a>` tag. (In which we want to make sure to preventDefatult()).

We can also use a `<textarea>` instead of the `<input type="text">`. The controller only expects it to have a `value` property and a `select()` method.

## Designing for Resiliance

This is about building in support for older browsers as well as considering what happens to our application when there are network or CDN issues.

It may be tempting to write these things off as not important but often it’s trivially easy to build features in a way that’s gracefully resilient to these types of problems.

This resilient approach, commonly known as **progressive enhancement**, is the practice of delivering web interfaces where the basic functionality is implemented in HTML and CSS. Tiered upgrades to that base experience are layered on top with CSS and JavaScript, progressively, when supported by the browser.

With the clipboard API the idea is to hide the `Copy to Clipboard` button unless the browser has support for the clipboard API. We do this by adding classes to the HTML, adding a bit of CSS to high the button, and adding a *feature check* in our JavaScript controller to toggle the class to show the button if the browser supports clipboard API.

The HTML looks like this:

```html
<div data-controller="clipboard" data-clipboard-supported-class="clipboard--supported">
    PIN: <input data-clipboard-target="source" type="text" value="1234" readonly>
    <button data-action="clipboard#copy" class="clipboard-button">Copy to Clipboard</button>  
</div>
```

And we add a `connect()` method to the `clipboard_controller.js`

```javascript

static classes = [ "supported" ]
  
  connect() {
    navigator.permissions.query({ name: 'clipboard-write' }).then( (result) => {
      if (result.state == "granted") {
        this.element.classList.add(this.supportedClass)
      }
    })
  }
```

**An issue I ran into locally on firefox with clipboard-write**

This code runs happily on Chrome and does the progressive enhancement. On firefox, I get the error in console:
```
Uncaught (in promise) TypeError: 'clipboard-write' (value of 'name' member of PermissionDescriptor) is not a valid value for enumeration PermissionName.
```

So even the code to check whether a given browser has access to a feature, in this case clipboard API, itself has browser specific issues.(sigh, that's the beauty of broswers and javascript lol).

## Managing State - Slideshow Controller

Most JavaScript frameworks encourage you to *keep state in JavaScript* at all times. They treat the DOM as a write-only rendering target (using client-side templates after consuming JSON from the server).

Stimulus takes a different approach. A Stimulus application’s state lives as *attributes in the DOM*; controllers (i.e. the JavaScript parts) are largely **stateless**. This approach makes it possible to work with HTML from anywhere—the initial document, an Ajax request, a Turbo visit, or even another JavaScript library.

We build a slideshow controller that keeps the index of the currently selected slide in an attribute, to learn how to store values as state in Stimulus.

**Lifecycle call-backs in Stimulus**

Stimulus lifecycle callback methods are useful for setting up or tearing down associated state when our controller enters or leaves the document.

These methods are invoked by Stimulus:

`initialize()` - Once, when the controller is first instantiated
`connect()` - Anytime the controller is connected to the DOM
`disconnect()` - Anytime the controller is disconnected from the DOM

**Using Values in Stimulus**

The concept of *values* is another core thing to Stimulus, similar to the concept of *controllers*, *actions*, and *targets*.

Stimulus controllers support typed `value` properties which automatically map to data attributes (`value` is an object while `targets` and `classes` are arrays). When we add a value definition to our controller class like this `static values = { index: Number }`, Stimulus creates a `this.indexValue` controller property associated with a `data-slideshow-index-value` attribute (and handle the numeric conversion for us).

**Value change callback**

In the code below, notice how we are having to manually call the `this.showCurrentSlide()` method each time we change the value in `this.indexValue`. Actually Stimulus will automatically do this for us if we add a method with this name `indexValueChanged()`. This method will be called at initialization and in response any change to the `data-slideshow-index-value` attribute (including if we make changes to it in the web inspector). Once we add `indexValueChanged()` we can also remove the `initialize()` method altogether.

The HTML code looks like this:

```html
<div data-controller="slideshow" data-slideshow-index-value="1">
    <button data-action="slideshow#previous"> ← </button>
    <button data-action="slideshow#next"> → </button>

    <div data-slideshow-target="slide">🐵</div>
    <div data-slideshow-target="slide">🙈</div>
    <div data-slideshow-target="slide">🙉</div>
    <div data-slideshow-target="slide">🙊</div>
  </div>
```

The `slideshow_controller.js` looks like this:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = [ "slide" ]

  static values = {index: Number}

  initialize() {
    this.showCurrentSlide()
  }

  next() {
    this.indexValue++
    this.showCurrentSlide()
  }

  previous() {
    this.indexValue--
    this.showCurrentSlide()
  }

  showCurrentSlide() {
    this.slideTargets.forEach((element, index) => {
      element.hidden = index != this.indexValue
    })
  }
}
```

We can use the web inspector to confirm that the controller element’s `data-slideshow-index-value` attribute changes as you move from one slide to the next. And that the `hidden` attribute is added and removed from each of the slide elements as we navigate.

**Exercise: Wrap around the index at 0 and max values**

## Working With External Resources - HTTP Requests and Timers

Sometimes our controllers need to track the state of external resources, where by external we mean anything that isn’t in the DOM or a part of Stimulus.

This example build a simple email inbox where the html for new messages is loaded asychronously (in this messages.html is just a static file but normally the server would return this html) using `fetch` and then plopped into the `innerHTML` of the controller's `div`. We then also use a timer to refresh and load new messages every 5 seconds.

This timer is started and stopped in the life-cycle methods, `connect()` and `disconnect()`, respectively.

The HTML placeholder looks like this, annotated with Stimulus attributes:

```html
<div data-controller="content-loader" data-content-loader-url-value="/messages.html" data-content-loader-refresh-interval-value="5000"></div>
```

The `content_loader_controller.js` looks like this:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { url: String, refreshInterval: Number }

  connect() {
    this.load()

    if (this.hasRefreshIntervalValue) {
      this.startRefreshing()
    }
  }

  disconnect() {
    this.stopRefreshing()
  }

  load() {
    fetch(this.urlValue)
      .then(response => response.text())
      .then(html => this.element.innerHTML = html)
  }

  startRefreshing() {
    this.refreshTimer = setInterval( () => {
      this.load()
    }, this.refreshIntervalValue)
  }

  stopRefreshing() {
    if (this.refreshTimer) {
      clearInterval(this.refreshTimer)
    }
  }
}
```

### Using content-loader controller on multiple elements

**params**

So far we have seen the concepts of *controllers*, *actions*, *targets*, and *values*. *params* is another Stimulus feature. *params* are associated with the element and not 'attached' at the controller level, unlike *values* and *targets* (i.e. there is not `static params = `)

Here is an example:

```html
<div data-controller="content-loader">
    <a href="#" data-content-loader-url-param="/messages.html" data-action="content-loader#load">Messages</a>
    <a href="#" data-content-loader-url-param="/comments.html" data-action="content-loader#load">Comments</a>
</div>
```

That `-url-param` can accessed in the controller's `load` action with `params.url`, like this:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  load({ params }) {
    fetch(params.url)
      .then(response => response.text())
      .then(html => this.element.innerHTML = html)
  }
}
```

**What happens if you add the same data-controller to nested HTML elements?**

I made a goofy mistake of adding `data-controller="content-loader"` to that 2nd `<a>` tag above, in addition to it being on the parent `<div>` already. And got to see some wonderfully weird results. The entire index.html loaded over and over again on the page, I could see the calls piling up in the network tab and the page's scroll bar getting smaller an smaller. Not sure exactly why. Perhaps I can think through this and use it a way to play around with the internal workings of Stimulus. This specific thing was further convoluted by the fact that the above `load` method was done in parallel with another `load` method from the original example of getting inbox messages loaded with a 5 second interval timer. 
