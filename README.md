<!--

WARNING!! DON'T EDIT THE FILE README.md on the root of the project, that one is a GENERATED FILE!

You should just edit the source file at src/README.md - the one which stars with ## Backbone.js tricks or treats

-->

## Backbone.js tricks or treats

<br/><br/><br/><br/><br/><br/>

Tiago Garcia @ [HTML5DevConf](http://html5devconf.com)

*http://tiagorg.com*

Oct 20th, 2014

---

## Agenda

 - The jQuery Way
 - Views and Memory leaks
 - Overwhelming the DOM
 - Nesting views
 - Application and Modules
 - Router vs Controller
 - Cohesion
 - Coupling

----

## Agenda

 - Data binding
 - Mocking AJAX

---

## Prerequisites

- Backbone.js
- Design patterns for large-scale javascript
- Curiosity

---

## The jQuery Way

- Backbone depends on jQuery, but it shouldn't mean abusing.
- Developers coming from strong jQuery background insist on *jQuerizing*, while Backbone provides structure to avoid that:
  - AJAX belongs to the Model and *SHOULD NOT* be coded like *`$.ajax()`*.
  - DOM events binding belongs to the View and *SHOULD NOT* be coded like *`$(el).click(...)`*.
- This is a common scenario in code migrations to Backbone, but simple to fix. Just have the Models and Views to do their work.
- Follow [Step by step from jQuery to Backbone](https://github.com/kjbekkelund/writings/blob/master/published/understanding-backbone.md) to better understand this process.

---

## Views and Memory leaks

- Backbone leaves much of the code structure for to the developer to define and implement.
- Bad designs easily lead to memory leaks.
```javascript
  var MyView = Backbone.View.extend({
    initialize: function() {
      this.model.on('change', this.render, this); // Data binding
    },

    render: function() {
      alert('Rendering the view');
    }
  });
```
- If instantiated twice, the 1st View will be never Garbage Collected, once model keeps a reference for it (*Zombie View*).
- This can cause *side-effects* - alert box will appear twice.

----

## Manual approach

- To fix it, we just need a method to *unbind* the View:
```javascript
    close: function() {
      // Unbind the events that this view is listening to
      this.stopListening();
    }
```
- However, we must remember to manually call this method whenever we destroy a View.
- Good practice: use a *Manager* to maintain the current View:
```javascript
    showView: function(view) {
      if (this.currentView) {
        this.currentView.close();
      }
      this.currentView = view;
      this.currentView.render();
      $('#mainContent').html(this.currentView.el);
    }
```

----

## Marionette.js

<img src="img/marionette.png" class="marionette" />

<ul class="full">
  <li>A Backbone.js composite application library to <br/>provide structure for large-scale Javascript.</li>
  <li>Includes good practices and design & <br/>implementation patterns.</li>
  <li>Reduces code boilerplate.</li>
  <li>Provides a modular architecture framework <br/>with a Pub/Sub implementation.</li>
  <li>And much more...</li>
</ul>

----

## Marionette's ItemView

- *Marionette.ItemView* extends *Backbone.View* and automates the rendering of a single item (Model or Collection).
- It implements *render()* for you, applying a given *template* to a Model/Collection.
- Using *listenTo()* instead of *on()* for binding events, you no longer need to manually invoke a *close* method.
```javascript
  var MyView = Marionette.ItemView.extend({
    template: '#my-ujs-template', // Underscore.js template
    template: Handlebars.compile($('#my-hbs-template').html()), // Handlebars.js template

    initialize: function() {
      this.listenTo(this.model, 'change', this.render);
    }

    // No render() anymore!! :)
  });
```

----

## Marionette's Region

- *Marionette.Region* is a Views container and manager.
- It manages their lifecycles and proper display on a DOM element and closing (no more Zombie Views).
```javascript
    var myRegion = new Marionette.Region({
      el: '#content'
    });

    var view1 = new MyView({ /* ... */ });
    myRegion.show(view1); // myRegion yields view and populates the DOM

    var view2 = new MyView({ /* ... */ });
    myRegion.show(view2); // myRegion yields view and populates the DOM
```

---

## Overwhelming the DOM

- In a View which renders a Collection, we normally use a child View for each item and append it to the parent, like:
```javascript
    var CollectionView = Backbone.View.extend({
      render: function() {
        _.each(this.collection.models, function(item) {
          var view = new MyView({
            model: item
          });

          this.$el.append(view.render().el); // Populating the DOM
        }, this);
      }
    });
```
- If the Collection has N items, this code makes N operations on the DOM, which is *expensive*. Imagine N = 1000?


----

## Manual approach

- A better approach is to append to a *document fragment* instead, and just add the fragment *once* to the DOM:
```javascript
    var CollectionView = Backbone.View.extend({
      render: function() {
        var fragment = document.createDocumentFragment();

        _.each(this.collection.models, function(item) {
          var view = new MyView({
            model: item
          });

          fragment.appendChild(view.render().el); // Appending to fragment
        }, this);

        this.$el.html(fragment); // Populating the DOM
      }
    });
```

----

## Marionette's CollectionView

- *Marionette.CollectionView* renders a Collection and uses a *Marionette.ItemView* for each item renderization. It doesn't need a template for itself.
- Uses a *document fragment* internally.
```javascript
  var MyView = Marionette.CollectionView.extend({
    itemView: MyView

    // No render() anymore!! :)
  });
```

----

## Marionette's CompositeView

- *Marionette.CompositeView* is similar to a *Marionette.CollectionView* but also takes a template for itself. Designed for parent-child relationships.
- Useful to build hierarchical and recursive structures like *trees*.
```javascript
  var MyView = Marionette.CompositeView.extend({
    itemView: MyView,
    template: '#node-template', // Template for the parent
    itemViewContainer: 'tbody' // Where to put the itemView instances into

    // No render() anymore!! :)
  });
```

---

## Nesting views

- Usual view nesting:
```javascript
  var OuterView = Backbone.View.extend({
    render: function() {
      this.$el.append(template);

      // Inner view
      this.innerView = new InnerView();
      this.innerView.render();
      this.$('#some-container').append(this.innerView.$el);
    }
  });
```
- Every call to *render()* will instantiate again the inner view, and rebind the events.
- The previous inner views have potential to be Zombies.
- Inner view is manually created, but never manually disposed.

----

## Manual approach

- A better approach to improve performance & avoid Zombies.
```javascript
  var OuterView = Backbone.View.extend({
    initialize: function() {
      this.inner = new InnerView(); // Instantiated just once
    },

    render: function() {
      this.$el.append(template);
      this.$('#some-container').append(this.innerView.el);
      this.inner.render();
    },

    // Needs to be manually invoked before removing
    close: function() {
      this.inner.remove();
    }
  });
```

----

## Manual approach

- A better approach to improve performance & avoid Zombies.
```javascript
  var InnerView = Backbone.View.extend({
    render: function() {
      this.$el.html(template);

      // Needed to bind events to new DOM
      this.delegateEvents();
    }
  });
```
- This still can be a mess for too many inner views.

----

## Marionette's Layout

- *Marionette.Layout* extends from *Marionette.ItemView* but provides embedded *Marionette.Region*s which can be populated with other views.
```javascript
  var OuterView = Backbone.Marionette.Layout.extend({
    template: '#outer-template',

    regions: {
      inner: '#inner-template'
    }
  });

  var outer = new OuterView();
  outer.render();

  var inner = new InnerView();
  outer.inner.show(inner);
```

---

## Application and Modules

- We all know how bad it is to create global vars, so we came up with *namespaces* in JS object structures.
- Common practice: create a plain JS object that everything is attached to, which also initializes the Backbone application.
- *Marionette.Application* does that and much more:
```javascript
  var MyApp = new Marionette.Application();

  MyApp.addInitializer(function(options) {
    new MyRouter(options.initialRoute);
    Backbone.history.start();
  });

  MyApp.start({
    initialRoute: 'home'
  });
```

----

## Marionette's Application

- Can fire events on before, after and during initialization.
- Holds *Marionette.Region*s for global layout organization.
- Can define, access, start and stop *Marionette.Modules*:
```javascript
  MyApp.module('myModule', function() { // Defining
    var privateData = 'I am private'; // Private member

    this.publicFunction = function() { // Public member
      return privateData;
    };

    this.addInitializer(function() { // Initializer
      console.log(privateData);
    });
  });

  var myModule = MyApp.module('myModule'); // Accessing
  myModule.start(); // Starting
  myModule.stop(); // Stopping
```

---

## Router vs Controller

- Routers commonly violate the *Single Responsibility Principle* (SRP) when used to:
  - instantiate and manipulate views
  - load models and collections
  - coordinate modules
- A Router's main (and only) purpose is to define routes and delegate the flow to different parts of the application.
- According to MVC, *Controllers* should deal with such things as Models, Collections, Views and Modules.
- Even though Backbone.js is MV*, there is nothing wrong on creating Controllers just as any other Module.

----

## Manual Approach

- One approach is to delegate the routes to controllers:
```javascript
  var MyRouter = Backbone.Router.extend({
    routes: {
      '': 'home',
      'home': 'home',
      'product/:id': 'viewProduct',
    },

    home: function() {
      myController.home();
    },

    viewProduct: function(productId) {
      myController.viewProduct(productId);
    }
  });
```

----

## Marionette's AppRouter

- *Marionette.AppRouter* binds the route methods to an external *Controller*:
```javascript
  var MyRouter = new Marionette.AppRouter({
    controller: myController,
    appRoutes: {
      '': 'home',
      'home': 'home',
      'product/:id': 'viewProduct',
    }
  });
```
- Good practice: divide routes into smaller pieces of related functionality and have multiple routers / controllers, instead of just one giant router and controller.

---

## Cohesion

- Backbone.js provides Models, Collections, Views and Routers. It doesn't mean we can't use other components here.
- Why should you write a complex interface logic (full of customized components) on the same file?
- To achieve a good separation of concerns, factor small pieces of related code into *cohesive Modules*:
```javascript
  MyApp.module('vehicleFactory', function() {
    this.createVehicle = function(wheels) {
      switch (wheels) {
        case 2: return new MotorcycleModel();
        case 4: return new CarModel();
        default: return new VehicleModel();
      }
    }
  });
```

---

## Coupling

- Components depending on other components usually create unnecessary *tight coupling*, which can be greatly reduced using a Pub/Sub:
```javascript
  var alerter = {         // Should be a Subscriber
    sayIt: function() {
      alert('May the force be with you.');
    }
  };
  var invoker = {         // Should be a Publisher
    start: function() {
      alerter.sayIt();
    }
  };
  invoker.start();
```
- Back in [Design Patterns for Large-Scale Javascript](http://slid.es/avenuecode/design-patterns-for-large-scale-javascript), a manual Pub/Sub implementation was proposed.

----

## Marionette's Application.vent

- *Marionette.Application.vent* also implements a Pub/Sub:
```javascript
  MyApp.module('alerter', function() {   // Subscriber
    this.addInitializer(function() {
      MyApp.vent.on('sayIt', function() {
        alert('May the force be with you.');
      });
    });
  });

  MyApp.module('invoker', function() {    // Publisher
    this.addInitializer(function() {
      MyApp.vent.trigger('sayIt');
    });
  });

  MyApp.module('alerter').start();
  MyApp.module('invoker').start();
```

---

## Data binding

- Backbone.js data binding is very primitive while other MV* frameworks (Ember.js, Knockout.js and AngularJS) excel on it.
- This code re-renders the whole View for each Model's change:
```javascript
  var MyView = Backbone.View.extend({
    initialize: function() {
      this.model.on('change', this.render, this); // Data binding
    },

    render: function() { ... }
  });
```
- Ideally, one attribute change on the Model should just re-render that attribute's representation on the DOM and not the whole View.

----

## Epoxy.js

- *Epoxy.js* is a data binding library for Backbone.js based on Knockout.js and Ember.js, featuring:
  - Computed Model & View Attributes
  - Declarative View Bindings
  - Automated Dependency Mapping
  - Automatic View Updates
- It connects each Model's attribute with a DOM element.
- Can be used together with Marionette's views.
- Can also compute results from the attributes's data.

----

## Epoxy.js

```javascript
  var bindModel = new Backbone.Model({
    name: 'Lando Calrissian'
  });

  var BindingView = Backbone.Epoxy.View.extend({
    el: '#my-form',
    bindings: {
      'input.name': 'value:name,events:["keyup"]', // Input
      'span.name': 'text:name', // Output
    }
  });

  var view = new BindingView({ model: bindModel });
```

```html
  <!-- Plain HTML template -->
  <div id="my-form">
    <label>Name:</label>
    <input type="text" class="name">

    <span class="name"></span>
  </div>
```

---

## Mocking AJAX

- How to unit test Backbone Models that consume data from a server?
- If we really use a server, this might get slow depending on the number of requests. And the server must work at all times, cause if it isn't working the tests will probably fail.
- If we use mocks, we don't depend on a server, however we need some code to feed the Model with mocks and hence we are not testing it perfectly (just like in Production).
- How does it sound to replace the browser AJAX with a fake AJAX powered with mocks?

----

## Sinon.JS

- Provides test spies, stubs and mocks for JavaScript.
- Works with any unit testing framework and also standalone.
```javascript
  var server = sinon.fakeServer.create();
  server.autoRespond = true;

  server.respondWith(
    'GET',
    '/character/422',
    [
      200,
      { 'Content-Type': 'application/json' },
      JSON.stringify({
        'id': 422,
        'title': 'Lando Calrissian'
      })
    ]
  );
```

---

## Conclusion

- Backbone.js is not a complete application structure framework, thus many details are left for the developer.
- In order to avoid problems and keep up with the good practices, frameworks as Marionette.js and Epoxy.js are very handy.
- Mocking AJAX with Sinon.JS provides a solid way to test Backbone.js integration.

---

## Learn more

1. [Structuring jQuery with Backbone.js](http://www.codemag.com/Article/1312061)
1. [Step by step from jQuery to Backbone](https://github.com/kjbekkelund/writings/blob/master/published/understanding-backbone.md)
1. [Zombies! RUN! (Managing Page Transitions In Backbone Apps)](http://lostechies.com/derickbailey/2011/09/15/zombies-run-managing-page-transitions-in-backbone-apps/)
1. [Developing Backbone.js Applications](http://addyosmani.github.io/backbone-fundamentals)
1. [Marionette.js](https://github.com/marionettejs/backbone.marionette)
1. [Reducing Backbone Routers To Nothing More Than Configuration](http://lostechies.com/derickbailey/2012/01/02/reducing-backbone-routers-to-nothing-more-than-configuration/)
1. [Epoxy.js](http://epoxyjs.org)
1. [Sinon.JS](http://sinonjs.org)

---

## Challenge

1. Pick your *Quiz App* or come up with a brand new Backbone.js application which requires interaction with the user.
1. Implement both *Marionette.js* and *Epoxy.js* on this project. Explore them well and use their features as much as possible.
1. Evaluate the pros and cons of your solution regarding the adoption of such frameworks, in terms of code organization, verbosity, scalability and robustness.
1. Send me your evaluation and the project in a GitHub repo.
