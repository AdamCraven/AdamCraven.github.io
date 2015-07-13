---
layout: post
title: Improving performance of enterprise angular apps by using better directives
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

The larger your angular app, the slower it gets. Watchers in enterprise apps can easily reach over 1,000 and dirty checking (digests) can take 10ms or longer. If a few digests occur within close proximity or are triggered one after another, this will cause the UI to freeze and the app to become unresponsive.

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
    <figcaption>Angular's scope tree. A rootscope digest visits every child scope and checks its watchers to see if any data has been modifed.</figcaption>
</figure>


## Not everything requires a rootscope digest

When building larger apps, code should be separated into [modules](/a-better-module-structure-for-angular/). Changes inside that module often have no side effects on other modules on the same page. This is a case when a full digest isn't needed. It is better to trigger updates via explicit [APIs](/a-better-module-structure-for-angular/#api) between modules in large apps.

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

Because of the ng-event directives (ng-keypress, ng-focus, etc..) attached, just entering that input field, typing one character and leaving the field will require 5 full rootscope digests. In a large app, the UI would be frozen for 50ms as the 5 digest cycles trigger one after another.

Unfortunately, it's hard to avoid triggering rootscope digests, because all the default event directives cause them to occur, which cannot be prevented.

To get around this, you could create a lot of different rootscopes in angular and isolate the modules from each other, but this causes a lot of problems with DI and other behaviour. There is a simpler solution:


## Faster angular events directives - (fng)

Faster angular event directives mimic the functionality of the existing ng-event directives, but can be called in a desired scope.

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/ng-event-anim.gif" alt="Scope tree">
    <img src="{{ site.url }}/images/fng-directives/fng-event-anim.gif" alt="Scope tree">
    <figcaption>Left: Using ng-events rootscope digest. Right: Using fng events in a large app</figcaption>
</figure>

<br />

In the ng-events example to the left it receives the keyboard inputs, but it takes a long time for the UI to unblock and letters to appear.

With the fng-events, there is no perceptible input lag and the text is entered as typed. Even though there are the same amount of watchers in the app, there are considerably less watchers in the scope it is digesting in, so the digest times will drop to a fraction of the time that the global one will.

### How it works

The fng are opt-in directives, they behave the same as a normal ng event directive. When triggered (e.g. fng-click), it will check that scope and then its parents for a true $stopDigestPropagation property. When found it will call a $digest in that scope.

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/scope-tree-local.gif" alt="Scope tree local">
    <img src="{{ site.url }}/images/fng-directives/scope-local-digest.gif" alt="Scope tree">
</figure>

If $stopDigestPropagation property doesn't exist on any parents, it will fall-back to the default behaviour and act the same as the default ng-events directives and call $rootScope.digest.

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/scope-tree.gif" alt="Scope tree local">
    <img src="{{ site.url }}/images/fng-directives/scope-full-digest.gif" alt="Scope tree">
</figure>

As they work the same way as the existing ng-event directives, they can be dropped in and used as a replacement.
That means all ng-keydowns can be converted to fng-keydowns, and so forth.

So the HTML for the fng-events look almoost same as the ng-events:

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

To bring the fng directives to life, and allow $digest to occur in the chosen scope, there is one change required in one of the parents scopes:

{% highlight js %}
    $parentScope.$stopDigestPropagation = true;
{% endhighlight %}


###How to chose the where to digest

It is not recommended that these are used at low levels, such as in individual components. The live search component mentioned earlier would not implement $stopDigestPropagation property. It should be implemented at the module level, or higher. Such as a

But to allow the $digest to occur in the desired scope, there is one change required in the scope of the live search directive:

{% highlight js %}
    $liveSearch.$stopDigestPropagation = true;
{% endhighlight %}




The directives can be installed from here: https://github.com/AdamCraven/fng-event-directives






