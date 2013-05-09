# Backbone.BaseView

Backbone.BaseView is a subclass of Backbone.View that helps with the most common tasks of managing views.

Generally most Backbone views have a similar lifecycle, it usually consists of rendering a template, responding to DOM events, updating elements and managing several subviews. BaseView is designed to remove some of the boilerplate associated with these tasks while retaining the flexibility that Backbone is known for.


## Getting started

BaseView is inherited like any other Backbone class by using the `extend` function. Below is an example of the most common methods you might want to override. It's a good place to start if you want to copy and paste an example to get up and running.

##### Example usage

```javascript
var MyView = Backbone.BaseView.extend({
  // initialize is usually a good place to set up model listeners and create subviews
  initialize: function() {
    this.listenTo(this.model, 'change', this.render);
    this.formView = new FormView(); // FormView also inherits from BaseView
  },

  // A hash of references to DOM elements, available to use after rendering (eg: this.$message)
  elements: {
    '$message': 'p',
    '$formContainer': '.formContainer'
  },

  // Must return a compiled template function, eg from Underscore, Handlebars, Mustache, etc
  template: function() {
    return Handlebars.compile('<p>Hello {{user.name}}, you are {{age}} years old</p><div class=".formContainer"></div>');
  },

  // The data object that'll be sent to the template when rendering
  templateData: function() {
    return {
      user: this.model,
      age: (new Date()).getFullYear() - this.model.dob.getFullYear()
    };
  },

  // Called automatically after rendering
  afterRender: function() {
    // Use the element references to access DOM elements
    this.$message('color', 'red');
    // Attach the subview to the container element, this will also automatically render it
    this.formView.appendTo(this.$formContainer);
  }
});
```

```javascript
// Create a new instance of the view and give it a model
var myView = new MyView({
  model: new Backbone.Model({
    name: 'Matt',
    dob: new Date('1980-01-01')
  });
});

// Attach the view to the DOM, this will also automatically render it
myView.appendTo('body');
```

Several concepts are introduced in this example. The most important thing to recognise is there's no `render` function - the rendering process is automated and uses the template information provided in the class definition. The other important point to make is that views are attached to the DOM using `appendTo`, which is a function provided by BaseView that helps with nested view management.


## Specifying templates and data

Out in the wild the majority of Backbone views tend to render a single template, and it's generally inserted into `View.$el`. This usually leads to the same rendering code being written over and over again. BaseView is designed to reduce this repetition and add a bit of structure:

```javascript
var MyView = Backbone.BaseView.extend({
  template: function() {
    // Must return a compiled template function, i.e. from Underscore, Handlebars, etc.
    return Handlebars.compile('<p>Hello</p>');
  },
  templateData: function() {
    // Return the object you want sent to the template when it's rendered
    // (this is an optional function, the template can still be rendered without it)
    return {
      user: this.model,
      age: (new Date()).getFullYear() - this.model.dob.getFullYear()
    };
  }
});
```

When you call `render()` it will automatically use the values you've provided and render the template into `View.$el`. Here's what the default `render` function in BaseView looks like:

```javascript
// Internally BaseView.render() does something like this:
var template = this.template();
var templateData = this.templateData();
this.$el.html(template(templateData)));
```

The actual code is a little more complex to account for null values, but this is the general gist.

For the majority of views this is usually enough, but obviously there'll be circumstances where you need to do something different. See the section on [Overriding render](#overriding-render) for how to customize things.

### Decorating templateData

You may be wondering why `template` and `templateData` are functions, and not just assigned actual values. This is deliberate, and an important part of how BaseView works.

`template` and `templateData` are designed to be *lazy*, and are called at the last moment, just before `render` actually renders stuff. This gives you the chance to change the value that's returned.

It turns out this is incredibly useful for `templateData`. If you're using a logic-less templating language, such as Handlebars, you can't compute values directly in your templates. You must either write a Handlebars helper or pre-compute the value before sending it to the template. BaseView promotes the second approach, and encourages you to wrap your data in an object alongside other values.

In the following example, `age` is not a property of the user model, so we calculate it before sending it to the template:

```javascript
templateData: function() {
  return {
    // The user model only has date of birth (dob) so we pre-calculate the age
    // of the user so we can easily use it in our template.
    age: (new Date()).getFullYear() - this.model.dob.getFullYear(),
    user: this.model
  }
}
```

And then in the template:

    <p>Hello {{user.name}}, you are {{age}} years old</p>

### Swapping templates

You can also use this feature for the `template` function:

```javascript
template: function() {
  // Return a different template based on some internal view state
  if (this.state === 'open') {
    return '<p>Hello {{user.name}}, welcome to the store</p>';
  } else {
    return '<p>Sorry, we are closed</p>';
  }
}
```

Because both `template` and `templateData` are called just before `render()` it means decisions about the rendering process are made when they're needed, and not pre-emptively. It also encourages you to keep the logic for your templates in two discreet locations, and not peppered throughout the class in other functions and callbacks.


## Before and after render

Every time you call `render()` there are two methods that are called automatically: `beforeRender` and `afterRender`. Overriding either is optional. `afterRender` is the more useful of the two, and is most commonly used to change parts of the newly rendered view.

```javascript
afterRender: function() {
  // Useful point at which to modify your rendered view
  this.$('.closeButton').css('color', 'red');
}
```

**Note:** Try to avoid using `beforeRender` to change the `template` and `templateData` properties. Instead you should place the logic inside the `template` and `templateData` functions themselves, as described in the [previous section](#decorating-templatedata).

**Also:** `afterRender` is a common place to cache selectors to DOM elements inside the view. Consider using the [Elements](#elements) hash as an alternative.


## Overriding render

There may be times when you want more control over how your view is rendered. If this is the case then just override `render` and write your own implementation:

```javascript
var MyView = Backbone.BaseView.extend({
  template: function() {
    return Handlebars.compile('Hello {{name}}');
  },
  templateData: function() {
    return { name: 'Matt' };
  },
  render: function() {
    // We'll use the existing template and templateData methods,
    // but we'll render into a different element:
    var template = this.template();
    var templateData = this.templateData();
    this.$('.container').html(template(templateData));
    return this;
  }
});
```

Another reason you might want to override `render` is because you're not using templates to generate your view.

**Note:** `beforeRender` and `afterRender` will still be called even if you override `render`. This might not seem necessary now that you have full control of rendering, but it can still be useful if you're keen on maintaining code consistency.


## Elements

Sometimes you need to modify or read values from DOM elements within your view. The most common approach is to cache references to them after the view is rendered, for example:

```
// Common method of working with elements inside a Backbone view:
render: function() {
  //...render the view...
  
  this.$title = this.$('.title'); // Store a reference to the element
},
onAlert: function() {
  this.$title.css('color', 'red'); // Use the reference later
}
```

This generally isn't a difficult task, but BaseView tries to add some consistency by defining a single place to put all your references. It will also keep them up-to-date as the view is re-rendered and dispose of them when it's removed.

Elements are defined in a hash that contains a *name* and *selector*. The name will be available to your view after it's rendered:

```javascript
var MyView = Backbone.BaseView.extend({
  elements: {
    '$title': '.title',
    '$bodyText': 'p'
  },
  template: function() {
    return Handlebars.compile('<h1 class="title">Welcome</h1><p>How are you?</p>');
  },
  afterRender: function() {
    this.$title.text('Hello');
    this.$bodyText.css('color', 'green');
  }
});
```

Here `.title` is a jquery selector, and `$title` is the name of the property that becomes available on your view after rendering. The `$` sign is an optional prefix, these examples use it to share consistency with the built-in `View.$el`.

Elements are automatically updated every time `render` is called. The first place you can use them is in `afterRender`. You can also force a manual refresh:

    // Force manual refresh of the elements, can be called at any time
    this.updateElements();

Element references are automatically disposed of when you call `View.remove()`.


## Dealing with nested views

Dealing with nested views is an aspect of Backbone that causes more confusion than anything else. It's usually the first problem new users face when trying to build an application, and usually the first thing they'll abstract into some sort of reusable code. Many of these solutions eventually get turned into plugins, just like this one.

### The problems

It's worth taking some time to understand the problems associated with managing nested views in Backbone. Even if you don't end up using BaseView then hopefully the following will give you a good idea of how to tackle the problem yourself.

**It's extremely easy to unbind DOM events by accident**

Watch out for jquery's `html()`, using it will unbind the events of any attached subviews. This means events inside the subview will cease to work.

```javascript
initialize: function() {
  this.welcomeView = new WelcomeView();
},
render: function() {
  // On the second time round this will remove the subview and unbind its events...
  this.$el.html(template(templateData));

  // ...so we need to attach the subview and re-bind events every time
  this.welcomeView.$el.appendTo(this.$('.container'));
  this.welcomeView.delegateEvents();
  this.welcomeView.render(); // You may also want to call this, to cascade render
}
```

Internally `.html()` calls `.empty()` which unbinds events on *all* child nodes. The solution above simply uses the built-in `delegateEvents()` to re-bind the events. You can also try temporarily detaching the subview from the DOM while you render.

**The order in which you attach/render your subviews can make a difference**

Sometimes a subview will size itself proportionally based on the dimensions of it's parent. If the subview isn't attached to the parent at the point you call `render` then you won't be able to read any meaningful dimensions from it. While not a common problem it's certainly one that's very frustrating if you've ever encountered it. It can be especially tricky if you've perpetuated the problem down a deep view hierarchy.

The following example shows how to fix this:

```javascript
// Common approach, however the view won't be attached to the parent during render
this.$el.append(this.mySubview.render().$el);

// Easily fixed by just attaching the view before rendering
this.$el.append(this.mySubview.$el);
this.mySubview.render();
```

**Remembering to dispose of your subview references**

Clearing up your subview references isn't hard, but it's easy to forget, and if you don't remove all the references then you might run into some memory management issues down the road.

Backbone does a good job of automatically cleaning up events in your views when you call `remove()` - wouldn't it be nice if it did the same for subviews?

### The solution

BaseView attempts to break these issues down into two separate solutions:

1. [Attaching subviews](#attaching-subviews) - Solves event binding and the attach-then-render problem
2. *TODO:* Storing subviews - Solves keeping track of subviews and automatically disposing of them

## Attaching subviews

BaseView provides several methods to help you attach views to the DOM:

* `myView.appendTo(element)` - Appends `myView.$el` to the given jquery `element`
* `myView.prependTo(element)` - Prepends `myView.$el` to the given jquery `element`
* `myView.replace(element)` - Replaces `element` with `myView.$el`

Internally each of these functions also re-delegates events and ensures the view is attached *before* it's rendered. This solves two of the [previously discussed problems](#the-problems).

Here's an example of their usage:

```javascript
var SiteView = Backbone.BaseView.extend({
  initialize: function() {
    // WelcomeView is a subclass of BaseView
    this.welcomeView = new WelcomeView();
  },
  template: function() {
    return '<h1>My Site</h1><div class=".container"></div>';
  },
  afterRender: function() {
    // Attach the subview to the container div, this will also render it
    this.welcomeView.appendTo(this.$('.container'));
  }
});

var siteView = new SiteView();
siteView.appendTo('body'); // Also use appendTo to attach and render the top-most view
```

**Note:** You should try to use `appendTo()` for every view, even the top-most one. This ensures you're taking advantage of the attach-then-render functionality for the entire view hierarchy.

It's worth mentioning that these methods don't store any subview references, you'll still be responsible for keeping track and disposing of them manually.

These methods are designed to maintain consistency with the Backbone API. As you may know Backbone already provides `remove()`, which is a wrapper around `this.$el.remove()` that also performs some additional tasks (such as cleaning up events). BaseView simply adds wrappers around other jquery methods (eg: `appendTo`) and performs it's own tasks.
