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

Summary: Digests can be slow, the standard directives trigger them often. Use more perf orientated directives


Summary: Digests can be slow. They happen often. Local digests are better. But you need something else to get the real performance edge


The larger your angular app, the slower it gets. Watchers in enterprise apps can easily creep over 1,000 and dirty checking (digests) can take 10ms or longer. If a few digests occur within close proximity of each other or are triggered one after another, this will cause the UI to freeze and the app to become unresponsive.

One way to solve this is to reduce the amount of rootscope digests happening in the application.


> Triggers of rootscope digest:
>
> * User events (ng-keypress, ng-keydown, ng-click, etc)
> * $q service
> * $http service
> * $timeout
> * $evalAsync
> * during animations
> * and manually triggered ($apply, $digest)

<figure>
    <img src="{{ site.url }}/images/fng-directives/scope-tree.gif" alt="Scope tree">
    <figcaption>Angular's scope tree. On a rootscope digest Every scope is visited and every watcher checked. Angular's dirty checking mechanism, it checks all the scopes in the app, calling every watcher to see if any data has been modified.</figcaption>
</figure>


### Not everything requires a rootscope digest

When building larger apps, code is separated into [modules](/a-better-module-structure-for-angular/). Changes inside that module often have no side effects on other modules on the same page. In this case a full digest cycle isn't needed. It is better to trigger via explicit [APIs](/a-better-module-structure-for-angular/#api) between large modules in the app.

### When digests become a problem

Take an example of a live search component. It is an input field that performs a live search and displays the results in a list after every keypress.

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

Entering that input field, typing one character and leaving the field will require 5 full rootscope digests. In our large app, the UI would be frozen for 50ms as the 5 digest cycles trigger.

Unfortunately, it's hard to avoid triggering rootscope digests, because all the default event directives (ng-click, ng-focus) cause them to occur, which cannot be prevented.

What if there was a way to prevent the whole app refreshing? This would mean the live search component could avoid triggering rootscope digest and instead trigger a more localised digest within the module it's within. And because there will be considerably less watchers at the module level, the digest times will drop to a fraction of

Fortunately angular is built in a way that means you don't have to rootscope digest all the time.


## Faster angular directives - (fng)

Faster angular events mimic the functionality of the existing ng-event directives, but have a feature that allows them to be called in the local scope.

The demo below shows both in action on a simulated large app:

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/ng-event-anim.gif" alt="Scope tree">
    <img src="{{ site.url }}/images/fng-directives/fng-event-anim.gif" alt="Scope tree">
    <figcaption>Left: Using ng-events rootscope digest. Right: Using fng events, localscope digest</figcaption>
</figure>

<br />

The ng-events receives the keyboard inputs, but it takes a long time for the UI to unblock and letters to appear.

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

The fng are opt-in directives, they behave the same as normal event directives.  Once $stopDigestPropagation on a scope.

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







