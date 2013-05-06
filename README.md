# Backbone.Baseview

Backbone.BaseView is a subclass of `Backbone.View` that helps with the most common tasks of managing views.

Generally most Backbone views have a simple lifecycle, it usually consists of rendering a template, responding to DOM events, and updating some elements within the view. BaseView is designed to remove some of the boilerplate associated with this, while retaining the flexibility that Backbone is known for.


## Getting started

BaseView is inherited like any other Backbone class by using the `extend` function. Below is an example of the most common methods you might want to override. It's a good place to start if you want to copy and paste an example to get up and running.

##### Example structure

```javascript
var MyView = Backbone.BaseView.extend({
  initialize: function() {
    // Usually the best place to set up model/collection listeners
    this.listenTo(this.model, 'change', this.render);
  }
  elements: {
    // A jquery selector that will be available after render() using: this.$msg
    '$msg': '.msg'
  },
  template: function() {
    // Must return a compiled template function
    return Handlebars.compile('<p class="msg">Hello {{user.name}}, you are {{age}} years old</p>');
  },
  templateData: function() {
    // Decorate templateData with additional data
    return {
      user: this.model,
      age: (new Date()).getFullYear() - this.model.dob.getFullYear()
    };
  },
  afterRender: function() {
    // Called automatically after render()
    this.$msg('color', 'red');
  }
});
```

```javascript
var myView = new MyView({
  model: new Backbone.Model({
    name: 'Matt',
    dob: new Date('1980-01-01')
  });
});

myView.$el.appendTo('body');
myView.render();
```

Several concepts are introduced in this example. The most important thing to recognise is there's no `render` function - the rendering process is automated and uses the template information provided in the class definition. The following sections will explain everything in more detail.


## Specifying templates and data

Out in the wild the majority of Backbone views tend to render a single template, and it's generally inserted into `View.$el`. This usually leads to the same rendering code being written over and over again. BaseView is designed to reduce repetition and add structure:

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

When you call `render()` it will automatically use the values you've provided and render the template into `View.$el`.

```javascript
// Internally BaseView.render() does something similar to this:
var template = this.template();
var templateData = this.templateData();
this.$el.html(template(templateData)));
```

The actual code is a little more complex to account for null values, but this is the general gist.

For the majority of views this is usually enough, but obviously there'll be circumstances where you need to do something different. See the section on [Overriding render](#overriding-render) for how to customize things.

### Decorating templateData

You may be wondering why `template` and `templateData` are functions, and not just assigned actual values. This is deliberate, and an important part of how BaseView works.

`template` and `templateData` are designed to be *lazy*, and are called at the last moment, just before `render` actually renders stuff. This gives you the chance to change the value that's returned.

This is incredibly useful for `templateData`. If you're using a logic-less templating language, such as Handlebars, you can't compute values directly in your templates. You must either write a Handlebars helper or pre-compute the value before sending it to the template. Wand promotes the second approach, and encourages you to wrap your data in another object alongside other values.

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

And then in your template:

    <p>Hello {{user.name}}, you are {{age}} years old</p>

### Swapping templates

Using a function for `template` is also useful, it allows you to change the template that's rendered:

```javascript
template: function() {
  // Return a different template based on some internal view state
  if (this.state === 'open') {
    return '<p>Hello {{user.name}}, welcome to the store</p>';
  } else {
    return '<p>Sorry, we are not open</p>';
  }
}
```

Because both `template` and `templateData` are called just before `render()` it means decisions about the rendering process are made when they're needed, and not pre-emptively. It also encourages you to keep the logic for your templates in two discreet locations, and not peppered throughout the class in other functions and callbacks.


## Before and after render

Every time you call `render()` there are two methods that are called automatically: `beforeRender` and `afterRender`. Overriding either is optional. `afterRender` is the more useful of the two, and it's most commonly used to change an element that's become available since rendering:

```javascript
afterRender: function() {
  // Useful point at which to modify your newly rendered view
  this.$('.closeButton').fadeIn();
}
```

**Note:** Try to avoid using `beforeRender` to change the `template` and `templateData` properties. Instead you should place the logic inside the `template` and `templateData` functions themselves, as described in the previous section.

**Also:** `afterRender` is a common place to cache selectors to DOM elements inside the view. Consider using the `elements` hash as an alternative. Check out the Elements section for more info.


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
  }
});
```

**Note:** `beforeRender` and `afterRender` will still be called even if you override `render`. This might not seem necessary now that you have full control of rendering, but it can still be useful if you're keen on maintaining code consistency.


## Elements

Elements is a property of `BaseView` that helps you manage DOM elements that sit inside your view. These are elements that don't warrant an entire View to themselves, but you still want to access them.

Each entry in the `elements` object specifies a *name* and a *selector*. The following example shows you how it's used:

```javascript
var MyView = Backbone.BaseView.extend({
  elements: {
    '$title': '.title'
  },
  template: function() {
    return Handlebars.compile('<h1 class="title">Welcome</h1>');
  },
  afterRender: function() {
    this.$title.text('Hello');
  }
});
```

Here `.title` is a jQuery selector, and `$title` is the name of the property that will become available on your view after `render` is called. The `$` sign is an optional prefix. These examples use it to share consistency with the built-in `View.$el` property.

Elements are updated every time `render` is called, and they're available by the time `afterRender` is called. Behind the scenes BaseView loops over the `elements` object setting properties directly on your view. You can also force a manual refresh:

    // Force manual refresh of the elements, can be called at any point
    this.updateElements();

Elements references are automatically disposed of when you call `View.remove()`.
