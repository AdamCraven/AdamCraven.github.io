---
layout: post
title: ngWhatever - A better structure for Angular modules (part 1 of 3)
excerpt: "How to create a better module structure for angular, to reduce code complexity and stop overloading services and controllers"
modified: 2014-10-07
tags: [angular, architecture]
comments: true
image:
  feature: texture-feature-05.jpg
  credit: Texture Lovers
  creditlink: http://texturelovers.com
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->


ngWhatever - version 1.0.0

This is a module structure introduced to the projects I have been working on recently. It has been very successful when implemented on large teams and is scalable from small applications all the way up to large financial trading platforms.

It clarifies where code should go, improves efficiency of the team by introducing strong conventions, makes unit testing easier and changes an application from an angular application, to an application that happens to use angular.

## Building an app that uses angular, not an angular app.

Angular is an MVW framework - a Model View **Whatever** - It specifically does not tell you how to structure your projects beyond the model view.

It does not provide enough structure to work on large projects. It is a framework whose power lays in the view. It is opinionated on how that should be done, and a lot of the core functionality centers around the view: directives, declarative HTML, controllers, data-binding, scope, etc.

This is what gets most people interested in angular, it is a very productive way of working when first starting out.

Many Angular projects tend to have a simple **core** module structure that looks like this: (The size of the items represent lines of code):

![Standard structure]({{ site.url }}/images/structure/standard-structure.png)
{: .image-pull-right}

The structure shown is one of the most basic angular apps structures and unfortunately remains one of the most used, even on the very large applications I have seen.


## Defining a structure - ngWhatever

ngWhatever is a structure, it doesn't tell the developer how to implement the code, but where it should go. It does not discriminate between methodologies, a functional reactive programming approach is just as valid as a traditional object orientated one.

It encourages a data-driven approach to the User Interface. The state of the application is stored in models, not held in the DOM or in many different functions.

The **core** of the ngWhatever consists of guidance on:

* Communication and parsing of data from the server.
* Storing, manipulation and validation of data.
* Communicating between views, models and the server.
* Configuration shared throughout the module.
* Communicating between others modules.

With the parts of the MV[W]hatever defined, we can break this down to a folder structure which provides a more robust structure for our application, using a pattern most are familiar with: MVC.

![Defined structure]({{ site.url }}/images/structure/better-structure.png)

The services have been removed and replaced with named parts doing a specific task, let's have a look at the structure in-depth. Starting with what already exists in angular:



## Example

An example of an implementation of the structure can be found here:
[https://github.com/AdamCraven/ngWhatever-example](https://github.com/AdamCraven/ngWhatever-example)

---


## Module Controllers

* Communicating between views, models and the server.

The name 'controller' is already used in angular, instead of renaming all the angular controllers and going against convention. It makes sense to rename the traditional controller in MVC to something else; the Module Controller.

The Module Controllers responsibilities are:

* Instantiating models
* Tells the view to update (in angular this is usually just calling a digest).
* Communicates with the endpoints to update models.


**When to use module controller over the controller**. The angular controllers which are assigned to directives or to the view, are view controllers. A view controller handles the view. It does not know about:

* Communicating with server.
* How to handle the data returned from the server.

Therefore, you should use module controllers in the above situations and also to provide module functionality that would not make sense in a view controller. e.g. module wide loading or error states.


## Models

* Storing, manipulation and validation of data.

If you have used Backbone.js before or have done a lot of server side development, the model should be a familiar concept. The model is the store and accessor to all data.

It contains view models and domain models. Much of the application logic will end up here. Also, because models are not a part of angular (they are usually constructor functions), core logic can be easily unit tested.

The view models are accessed via the controllers and/or view templates to retrieve application data. The simple objects that are on the scope, are completely replaced by view models. State is no longer stored in the view controllers, it is represented in the view models. This is a point worth repeating: **Models are created outside of a controller and introduced to it, they are no longer simple objects created in the controller directly on the scope**.

Moving much of the logic to the models, reduces the code in the [view] controllers significantly. They become simple APIs for the view, and separating all view state representation into the models.

There are several ways to call the models:

* Changing properties of the model directly in the view template.
* Using setters/getters to change the model directly in the view template.
* Hide the model from the view template and use methods on the [view] controller to change properties in the model.
* If using IE10+ the uniform access principle to update transparently via both use of properties or setters.

*NOTE: In angular 1.2, ng-model doesn't have the ability to call setter functions. A simple custom directive can do the job to enable the use of setter functions on form elements. Or if using a version of angular greater than 1.2, documentation for this feature can be found here: [ngModelOptions](https://docs.angularjs.org/api/ng/directive/ngModelOptions)*


A data driven approach is the gold standard for UI applications, but it is important to apply good design principles to models (avoid multiple levels of inheritance, however tempting), such as decorators, composition, mixins and more. Ben Teese has a good introduction to structuring rich data models in angular: [rich object models in angular](http://blog.shinetech.com/2014/02/04/rich-object-models-and-angular-js/)

**Validation in the models**

Whilst validation can happen in the view and angular encourages it. It is an anti-pattern in bigger applications, views might not share the same validation logic and should be fairly dumb. As the logic for validation doesn't exist in the model. The models cannot be unit tested to check their allowed states.

Validation should be centered in the model and in advanced use cases - such as multi-field form validation or an update of one model affecting another - a validation model should be created which knows about every model it needs to validate.

**Filtering in the models**

Angular also encourages the use of filters on the view, but it better to filter in the model on larger applications. Once again speeding up the application, relying less on angular and allowing the filtered states to be unit tested.

## Endpoints

* Communication and parsing of data from the server.

The endpoints communicate between the resources - network, mock or local storage. It also defines a contract with the server API. It handles all error states, such as: syntax errors, timeouts, 4xx and 5xx errors. It usually operates via promises or similar, providing a simple interface for the module controllers.

Parsing also happens here, which is decoding/encoding messages to and from the server and converting into a format that is view/domain model ready. A lot of unit tests defining your contract with the server will be centered here.

## Config

* Configuration shared throughout the module.

Config is local constants and values that are used by the module. Such as default values to the models, initial view states and provide a place to put configuration that may eventually be moved to the server.

## API

* Communicating between others modules.

The API exposes methods of the module and broadcasts changes when events happen.
It makes an explicit contract for anything calling or listening to the module. By convention, any internal methods (e.g. on the module controller, models) cannot be accessed unless they are called through the API.

It is important to maintain a strict barrier between your modules internals and externals. Keeping the module decoupled allows the internals to be completely changed without unexpected side effects on the rest of your application.

Usually a higher level controller will link one or more module APIs together, and - on large applications - create an adapter between the APIs if their inputs and outputs are incompatible.

If the module is completely declarative, the directives API can become the API.


---

## Using the structure

![File structure]({{ site.url }}/images/structure/file-structure.png){: .image-pull-right}

To the right is a module structure for a module called 'habit'. It implements the structure. The services folder has been replaced by 4 well defined folders: models, module controllers, endpoints and config.

There is one extra file, uncovered in the structure, it's the *habit-module.js*. It defines initial module bootstrapping, such as assigning controllers, directives and services to the angular module.

Using this module structure, it provides a guide as to how to structure your project. But it is important to understand that it is **flexible** in how it can be used.

For very small modules you might not need a module controller. Sometimes the endpoints aren't needed as data is being passed in via an external module (through the API). For simple modules and applications, data validation may occur in the view and in a large module, it maybe beneficial to have multiple module controllers.

## Further reading

* Part 2 - A walk through of the implementation with code samples (coming soon)
* Part 3 - Advanced implementations (coming soon)

---


Want to get in touch? You can find me at [@Adam_Craven](https://twitter.com/Adam_Craven) on twitter.
