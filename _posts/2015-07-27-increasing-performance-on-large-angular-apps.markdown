---
layout: post
title: angular-fng - Improve the performance of large angular 1.x apps, by using faster event directives
excerpt: "How to avoid root scope digests using faster angular events, which mimic the functionality of the existing ng-event directives, but have a feature that allows them to be called in a desired scope, rather than trigger a root scope digest."
modified: 2015-07-27
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

The larger an angular app, the slower it becomes. Watchers in large apps can exceed over 1,000 and dirty checking (digests) can take 5ms or longer. If a few digests occur within close proximity or are triggered one after another, this will cause the UI to freeze and the app to become unresponsive.

One way to solve this is to reduce unnecessary calls to the dirty checking mechanism for watchers that haven't updated - specifically *root scope digests* - which is the global digest that queries every watcher in an app.

<figure>
    <img src="{{ site.url }}/images/fng-directives/scope-tree.gif" alt="Scope tree">
    <figcaption>Angular's scope tree. A root scope digest visits every child scope, checks its watchers and sees if any data has been modified.</figcaption>
</figure>

> Triggers of a root scope digest:
>
> * User events (ng-keypress, ng-keydown, ng-click, etc)
> * $q service
> * $http service
> * $timeout
> * $evalAsync
> * during animations
> * and manually triggered ($apply, $digest)

## Root scope digests aren't always desirable

When building larger apps, code is separated into [modules](/a-better-module-structure-for-angular/). When changes occur inside a module that has no side effects on other modules on the same page, a full digest isn't needed. For example, when an item on a page is updating 5 times a second, but the rest of the app remains the same, there is no need to refresh everything. Doing so would slow the UI down.

When a module does affect another, it is better to trigger updates via explicit [APIs](/a-better-module-structure-for-angular/#api) between modules. The default root scope digest mechanism can also be used, but it can become problematic in large apps.

## Problematic digests

Take a live search component as an example. It is an input field that performs a live search and displays the results in a list after every keypress.

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

Because of the ng-event directives (ng-keypress, ng-focus, etc..), just entering the live search input field, typing one character and exiting the field will require 5 full root scope digests. In a large app, the UI could be frozen for 25ms or longer as the 5 digest cycles trigger one after another.

Unfortunately, you cannot avoid triggering root scope digests, because all the default event directives cause them to occur.

A work around involves creating several angular apps on a page, with individual root scopes. This ensures the modules are isolated from each other, but it causes a lot of problems with DI and other behaviour. There is a simpler solution...

## Angular-Fng - Faster angular (fng)

Faster angular event directives mimic the functionality of the existing ng-event directives, but can be called in a desired scope.

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/ng-event-anim.gif" alt="Scope tree">
    <img src="{{ site.url }}/images/fng-directives/fng-event-anim.gif" alt="Scope tree">
    <figcaption>Left: Using ng-events root scope digest. Right: Using fng events in a large app</figcaption>
</figure>

<br />

The ng-events example to the left receives the key presses, but it takes a long time for the UI to unblock and letters to appear. This is due to root scope digests consecutively being called.

With the fng-events, there is no perceptible input lag and the text is entered as typed. It is not calling a root scope digest, but a partial (or local) digest where there are considerably less watchers. As a result of this, the digest times drop to a fraction of the time of the global root scope digest.

Both examples are very similar in terms of code, but the HTML for the right hand demo replaces 'ng' with 'fng':

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

To bring the partial digest functionality to life, there is one change required in the JS. A property is set on one of its parent scopes:

{% highlight js %}
    $parentScope.$stopDigestPropagation = true;
{% endhighlight %}



### How it works

The fng events are opt-in directives, which behave *the same* as an ng event directive. However, it differs in one important way. When triggered (e.g. fng-click) it bubbles up the scope tree and searches for a defined $stopDigestPropagation property.

When found it will call a $digest in the scope where $stopDigestPropagation is set and checks all the child scopes as shown below:

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/scope-tree-local.gif" alt="Scope tree local">
    <img src="{{ site.url }}/images/fng-directives/scope-local-digest.gif" alt="Scope tree">
</figure>

<br />

If $stopDigestPropagation property isn't found, it will fallback to the default behaviour and act **the same** as the ng-event directives, calling a root scope digest:

<figure class="half">
    <img src="{{ site.url }}/images/fng-directives/scope-tree.gif" alt="Scope tree local">
    <img src="{{ site.url }}/images/fng-directives/scope-full-digest.gif" alt="Scope tree">
</figure>

Because they work the same as the existing ng-event directives, they can be dropped in and used as a replacement.
That means all ng-keydowns can be converted to fng-keydowns, and so forth.


###How to chose where to digest

It is not recommended that these are used at low levels, such as in individual components. The live search component mentioned before would not implement $stopDigestPropagation property. It should be implemented at the module level, or higher. Such as a group of modules that relate to a major aspect of functionality on a page.

---

The code, further reading and installation instructions can be found on github: [fng-event-directives](https://github.com/AdamCraven/fng-event-directives).

I hope you've found this useful. Want to get in touch? You can find me at @Adam_Craven on twitter.

<a href="https://github.com/AdamCraven/fng-event-directives"><img style="position: absolute; top: 0; right: 0; border: 0;" src="https://camo.githubusercontent.com/38ef81f8aca64bb9a64448d0d70f1308ef5341ab/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f6769746875622f726962626f6e732f666f726b6d655f72696768745f6461726b626c75655f3132313632312e706e67" alt="Fork me on GitHub" data-canonical-src="https://s3.amazonaws.com/github/ribbons/forkme_right_darkblue_121621.png"></a>







