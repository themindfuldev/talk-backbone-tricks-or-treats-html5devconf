<!--

WARNING!! DON'T EDIT THE FILE README.md on the root of the project, that one is a GENERATED FILE!

You should just edit the source file at src/README.md - the one which stars with ## @@title

-->

## @@title

<br/><br/><br/><br/><br/><br/>

@@author @ [HTML5DevConf](http://html5devconf.com)

*@@site*

@@date

---

## Tiago Garcia

<img src="http://www.gravatar.com/avatar/5cac784a074b86d771fe768274f6860c?s=250" class="avatar">

- Tech Manager at [Avenue Code](http://www.avenuecode.com) 
- Tech Lead at [Macys.com](http://www.macys.com).
- Organizer of the [Backbone.js Hackers meetup in SF](http://www.meetup.com/Backbone-js-Hackers).


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
 - Data binding
 - Components
 - Mocking async calls

---

## Prerequisites

- [Backbone.js](http://slides.com/avenuecode/boosting-the-client-side-with-backbone-js#/)
- [Design patterns for large-scale javascript](http://slides.com/avenuecode/design-patterns-for-large-scale-javascript#/)
- Curiosity
- Opinion

---

## AJAX Rant

- Think twice before saying AJAX ever again.
- AJAX = *A*synchronous *J*avascript *A*nd *X*ML.
- AJAJ = *A*synchronous *J*avascript *A*nd *J*SON.
- It is hardly because of *XMLHttpRequest* object -> it can transfer in any format.
- RESTful APIs became popular -> I don't quite see XML responses ever since.
- XML lacks out-of-the-box support on MV* frameworks.
- However, *AJAJ* doesn't sound as cool as *AJAX* so people prefer not using the proper term.
- *ASYNC*, FTW! Who will join me?

----

## ASYNC

<img src="img/async.jpg">

---

## The jQuery Way

- Backbone depends on jQuery\*, but *depending !== abusing*.
- Newcomers tend to adopt jQuery-based solutions instead of taking advantage of Backbone.js structures:
  - *Backbone.Model* takes care of async calls so they **SHOULDN'T** be coded like *`$.ajax()`*.
  - *Backbone.View* takes care of DOM events binding so they **SHOULDN'T** be coded like *`$(el).click(...)`*.
- A common scenario in code migrations to Backbone, but simple to fix. Just put Models and Views to do their work.
- Follow [Step by step from jQuery to Backbone](https://github.com/kjbekkelund/writings/blob/master/published/understanding-backbone.md) to shed some light on this process.

---

## Views and Memory leaks

- Backbone leaves much of the code structure for the developer to define and implement.
- Bad designs easily lead to memory leaks.
```javascript
  var MyView = Backbone.View.extend({
    initialize: function() {
      this.model && this.model.on('change', this.render, this);
    },
    render: function() {
      alert('Rendering the view');
    }
  });
```
- *MyView* gets instantiated twice: 1st will never be Garbage Collected, as the model holds its reference -> *Zombie View*.
- This can cause *side effects* -> alert box will appear twice.

----

## Manual approach

- To fix it, we just need a method to *unbind* the View:
```javascript
    close: function() {
      // Unbind the events that this view is listening to
      this.stopListening();
    }
```
- However, we must remember to manually call this method whenever we destroy a *MyView* instance.
- Good practice: use a *Manager* to maintain the current View:
```javascript
    showView: function(view) {
      this.currentView && this.currentView.close();
      this.currentView = view;
      this.currentView.render();
      $('#content').html(this.currentView.el);
    }
```

----

## Marionette.js

<img src="img/marionette.png" class="icon" />

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
- By using *listenTo()* instead of *on()* for binding events, you no longer need to manually invoke a *close* method.
```javascript
  var MyView = Marionette.ItemView.extend({
    template: '#my-ujs-template', // Underscore.js template
    initialize: function() {
      this.listenTo(this.model, 'change', this.render);
    }

    // No render() anymore!! :)
  });
```

----

## Marionette's Region

- *Marionette.Region* is a Views container and manager.
- It properly displays Views on the DOM and disposes them -> no more Zombie Views.
```javascript
    var myRegion = new Marionette.Region({
      el: '#content'
    });

    var view1 = new MyView({ /* ... */ });
    myRegion.show(view1); // renders view1 and appends to #content

    var view2 = new MyView({ /* ... */ });
    myRegion.show(view2); // disposes view1, renders/append view2
```

---

## Overwhelming the DOM

- In a View which renders a Collection, we normally render a child View for each item and append them to the parent:
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
- Say the Collection has N items, this code will make N appends, which is *expensive*. What if N = 1000?


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

- Common view nesting:
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
- This can easily get messy for multiple inner views.

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

- Global vars are evil. Nesting global vars (*namespaces*) is a common alternative, but it can easily get slow/messy.
- Non-AMD applications usually have a global var to represent the app, hold the declarations and initialize it.
- Introducing *Marionette.Application*, without nesting:
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
    var privateData = 'I am private'; // Private var
    this.publicFunction = function() { // Public method
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

## Backbone.Radio

- [Backbone.Radio](https://github.com/marionettejs/backbone.radio) also implements a Pub/Sub:
```javascript
  MyApp.module('alerter', function() {   // Subscriber
    this.addInitializer(function() {
      Backbone.Radio.channel('alerter').on('sayIt', function() {
        alert('May the force be with you.');
      });
    });
  });

  MyApp.module('invoker', function() {   // Publisher
    this.addInitializer(function() {
      Backbone.Radio.channel('alerter').trigger('sayIt');
    });
  });

  MyApp.module('alerter').start();
  MyApp.module('invoker').start();
```
- This will be replacing *Backbone.Wreqr* on Marionette.js.

---

## Data binding

- Backbone.js data binding is primitive while other MV* frameworks (Ember.js, Knockout.js, AngularJS) excel on it.
- This code re-renders the whole View for each Model's change:
```javascript
  var MyView = Backbone.View.extend({
    initialize: function() {
      this.model && this.model.on('change', this.render, this); 
    },

    render: function() { ... }
  });
```
- Ideally, one attribute change on the Model should just re-render that attribute's representation on the DOM and not the whole View.

----

## Epoxy.js

<img src="img/epoxy.png" class="icon" />

- *Epoxy.js* is a data binding library for<br/>
Backbone.js based on Knockout.js<br/>
and Ember.js, featuring:
  - Declarative View Bindings
  - Automatic View Updates
  - Computed Model & View Attributes
- It connects each Model's attribute with a DOM element.
- Can be used together with Marionette's views.
- Can also compute results from the attributes's data.

----

## Epoxy.js

```html
  <!-- Plain HTML template -->
  <div id="my-form">
    <label>Name:</label>
    <input type="text" class="name">
    <span class="name"></span>
  </div>
```

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

---

## Components

- Views reusability for real? Think components.
- *Web Components* spec is right around the corner.
- *React* is a View framework about:
<img src="img/react.png" class="icon" />
  - Components! (abstraction & composition)
  - Elements (DOM) vs. Templates (strings)
  - Virtual DOM 
  - Reactive data flow (or *ReactLink* for Two-way)
  - Bonus: works on the server-side!
- *React* can replace the whole *Backbone.View* layer.
- It just takes a mixin to be integrated with Backbone, or something as [react.backbone](https://github.com/clayallsopp/react.backbone).

----

## React.js

```javascript
  var bindModel = new Backbone.Model({
    name: 'Lando Calrissian'
  });

  var BindingView = React.createBackboneClass({
    render: function() {
      return (
        React.DOM.span( 
          { className: 'name' }, this.getModel().get('name')
        )
      );
    }
  });

  React.renderComponent(
    BindingView({ model: bindModel }), document.getElementById('my-form')
  );
```

---

## Benchmark

- <a href="http://jsfiddle.net/tiagorg/5L9qxnsq/" target="_blank">JSFiddle</a>
- Yes, structure and data binding have a price....
- ...so as the chaos of not having them!

---

## Mocking async calls

- How to unit test Models that consume data from a server?
- Using a real server can be slow. The server must also be up at all times, otherwise the tests will probably fail.
- Using *data mocks* frees us from server dependency. 
- However, it deviates from the way the Model will actually be used in Production, which creates challenges for testing.
- What about replacing the browser async mechanism with a *mock-powered proxy*?

----

## Sinon.JS

<img src="img/sinon.png" class="icon" />

- Test spies, stubs and mocks for async.
- Framework-agnostic.
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
- Goes well with [Leche](https://github.com/box/leche) for mock creation.

---

## Conclusion

- Backbone.js is not a complete application structure framework, thus many details are left for the developer.
- In order to avoid problems and keep up with the good practices, *Marionette.js* comes very handy.
- Data binding can be made easy with *Epoxy.js*.
- *React.js* can greatly componentize your View layer.
- Mocking async calls with *Sinon.JS* provides a solid way to test Backbone.js integration.

---

## Learn more

1. [Structuring jQuery with Backbone.js](http://www.codemag.com/Article/1312061)
1. [Step by step from jQuery to Backbone](https://github.com/kjbekkelund/writings/blob/master/published/understanding-backbone.md)
1. [Zombies! RUN! (Managing Page Transitions In Backbone)](http://lostechies.com/derickbailey/2011/09/15/zombies-run-managing-page-transitions-in-backbone-apps/)
1. [Developing Backbone.js Applications](http://addyosmani.github.io/backbone-fundamentals)
1. [Marionette.js](https://github.com/marionettejs/backbone.marionette)
1. [Reducing Backbone Routers To Nothing More Than Configuration](http://lostechies.com/derickbailey/2012/01/02/reducing-backbone-routers-to-nothing-more-than-configuration/)
1. [Backbone.Radio](https://github.com/marionettejs/backbone.radio)
1. [Epoxy.js](http://epoxyjs.org)
1. [React](http://facebook.github.io/react/)
1. [Sinon.JS](http://sinonjs.org)
1. [Leche](https://github.com/box/leche)

---

## Meetups

1. [Backbone.js Hackers](http://www.meetup.com/Backbone-js-Hackers/) - San Francisco
1. [Dancing with Marionette](http://www.meetup.com/Dancing-with-Marionette-js/) - New York
1. [ReactJS](http://www.meetup.com/ReactJS-San-Francisco/) - San Francisco

---

## Challenge

1. Fork my [Quiz App](https://github.com/tiagorg/quiz-app) or come up with a brand new Backbone.js application which requires interaction with the user.
1. Use *Marionette.js* and either *React.js* or *Epoxy.js*. Explore them well and use their features as much as possible.
1. Evaluate the pros and cons of your solution regarding the adoption of such frameworks, in terms of code organization, verbosity, scalability and robustness.
1. Send me your analysis and project in a GitHub repo.
