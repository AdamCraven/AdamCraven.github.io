---
layout: post
title: Improving performance of enterprise angular apps
excerpt: "Why digests can be slow. How you can leverage local digests to make it faster combining with custom directives for a performance edge"
modified: 2015-03-20
tags: [angular, digest]
comments: true
image:
  feature: sample-image-4.jpg
  teaser: sample-image-3.jpg
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


Summary: Digests can be slow. They happen often. Local digests are better. But you need something else to get the real performance edge

<figure>
    <img src="{{ site.url }}/images/fng-directives/scope-tree.gif" alt="Scope tree">
    <figcaption>Angular's scope tree. Every scope is visited on a digest to check watchers</figcaption>
</figure>
The core of angular's dirty checking mechanism, it checks all the scopes in the app, calling every watcher to see if any data has been modified.

With a good JIT JS engine and decent hardware spec, you would expect a complete digest to take less than a millisecond for small apps.

In large enterprise apps with over 1,000 watchers. It can creep above 10ms or more. This is a long time to halt the browser and with other tasks occurring on the same thread, such as rendering, it will cause application performance to dip below 60fps.


* User events (ng-keypress, ng-keydown, ng-click, etc)
* $q service
* $http service
* $timeout
* $evalAsync
* during animations
* and manually triggered ($apply, $digest)

It is very easy for many $rootscope digests to compound together to bring the UI to a halt. An example is below, again based on a real production component. Here we have a live search component, it is an input field that performs a live search and displays the results in a list:

Digests may seem infrequent, but they are triggered by many things:

{% highlight html %}
{% raw %}
<input class="live-search"
    ng-keypress="ctrl.keypress()"
    ng-keyup="ctrl.keyup()"
    ng-keydown="ctrl.keydown()"
    ng-focus="ctrl.focus()"
    ng-blur="ctrl.focus()"
/>
<ul>
    <li ng-repeat="item in ctrl.searchResults()">{{item.name}}</li>
</ul>
{% endraw %}
{% endhighlight %}

Entering that input field, typing one character and leaving the input field will require 5 full rootscope digests. 6, when the response from a live search service returns. In our large app, that could be 72ms of the time spent digesting and the UI would freeze, creating a poor user experience.

What if there was a way to prevent the whole app refreshing? This is especially important if you have components that don't make changes to the rest of the app or if they do interact via an [API](/a-better-module-structure-for-angular/#api) to control the updates to other components.

Fortunately angular is built in a way that means you don't have to rootscope digest all the time.

## The local digest

An angular app can be told to call a digest on its scope and children, only. It will not refresh its parent or sibling scopes.

All it requires is calling the $digest function on the desired scope.

{% highlight javascript %}
{% raw %}
    $scope1.$digest();
{% endraw %}
{% endhighlight %}

<figure>
    <img src="{{ site.url }}/images/fng-directives/scope-tree-local.gif" alt="Scope tree local">
    <figcaption>$scope1.digest() will trigger watchers on itself and children, the rest of the scopes remain unaffected</figcaption>
</figure>


A childscope of a parent is likely to only have a few watches existing on it, far less than the app and digest times will drop to a fraction.

Working inside a [module]({{ site.url }}/a-better-module-structure-for-angular/)

// Talk about when to use it. A few paragraphs


### Back to our example...

There is still a problem, however. Taking our previous example of the live search component, it is still calling the ng-event directives which all trigger the rootscope digests, the most we can reduce the rootscope digests by in that example is from 6 to 5 - When the result received from the server could be executed as a local digest.

Unfortunately there is no way to make the default angular directives execute a local digest without hacking the framework, but we can easily replace the directives with ones that can.

## Faster angular directives - (fng)

Faster angular events mimic the functionality of the exisiting ng-event directives, but have a feature that allows them to be called in the local scope.

The demo below shows both in action on a simulated large app:

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/ng-event-anim.gif" alt="Scope tree">
    <img src="{{ site.url }}/images/fng-directives/fng-event-anim.gif" alt="Scope tree">
    <figcaption>Left: Using ng-events rootscope digest. Right: Using fng events, localscope digest</figcaption>
</figure>

<br />
The ng-events receives the keyboard inputs, but it takes a long time for the UI to unblock and the letters to appear.

With the fng-events, there is no perceptible input lag and the text is entered as typed.

The HTML for the fng-events are

{% highlight html %}
{% raw %}
<input class="live-search"
    fng-keypress="ctrl.keypress()"
    fng-keyup="ctrl.keyup()"
    fng-keydown="ctrl.keydown()"
    fng-focus="ctrl.focus()"
    fng-blur="ctrl.focus()"
/>
{% endraw %}
{% endhighlight %}

There is one change required in the scope of the live search directive:

{% highlight js %}
    $liveSearch.$stopDigestPropagation = true;
{% endhighlight %}

### How it works

The fng are opt-in directives, you have to set the $stopDigestPropagation on a scope.

When a user interacts with a scope by triggering an event, it will check that scope for a truthy $stopDigestPropagation property. If that doesn't exist it will check its parent. When found it will call a $digest in that scope.

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/scope-tree-local.gif" alt="Scope tree local">
    <img src="{{ site.url }}/images/fng-directives/scope-local-digest.gif" alt="Scope tree">
</figure>

When the $stopDigestPropagation property doesn't exist on any parents, it will fallback to the default behaviour of the ng-events and call a $rootScope.digest.

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/scope-tree.gif" alt="Scope tree local">
    <img src="{{ site.url }}/images/fng-directives/scope-full-digest.gif" alt="Scope tree">
</figure>

As they work that same way as the existing ng-event directives, they can be dropped in and used as a replacement.
That means all ng-keydowns can be converted to fng-keydowns, and so forth.







