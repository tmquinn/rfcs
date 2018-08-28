- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Tracked Properties

## Summary

This RFC introduces the concept of tracked properties. Tracked properties are just like normal properties but are the mechanism for opting into change detection. Tracked properties are exposed by Ember as decorators.

As an example this is what that would look like.

```js
import { tracked } from '@ember/object';

export default class Example {
  @tracked firstName;
  @tracked lastName;

  @tracked
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  set fullName(fullName) {
    let [ first, last ] = fullName.split(' ');
    this.firstName = first;
    this.lastName = last;
  }

  changeFirstName(firstName) {
    this.firstName = firstName;
  }

  changeLastName(lastName) {
    this.lastName = lastName;
  }
}
```

If you are not familar with how change tracking has evolved in Ember please see the appendix.

## Motivation
Due to the flexibility of the reference system introduce by the Glimmer VM, we have been able seamlessly model the semantics of the notification-based change tracking system in terms of the pull-based change tracking system. While it was a goal to ensure backwards compatability, references can actually be used to help simplify the programming model. Below are some areas of oppurtunity.

### Opt-in Mutations

Today, any value that this referenced in the templating layer is automatically opted into observing changes. If you attempt to mutate those properties without `set` you will get an error like the following:

```
You must use set() to set the `someProp` property (of custom-object) to `foo-bar`.
```

This is typically refered to as the "mandatory setter" error. While it is possible to set values using normal javascript assignment, it is considered a best practice to always use `set` since it is not immediately obvious as to what properties are being observed or not. With tracked properties, only changes to properties that have been annotated with the `@tracked` decorator will be observable through out the system.

### Remove `set`

Since the `@tracked` encapsulates the change tracking details within the installed property descriptor, we are able to drop `set` for assigning and just javascript assignment.

### Remove Need For Dependent Keys

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

## Appendix

Historically, Ember has used `get` and `set` for accessing and setting properties on a class. This was largely due to the fact that Ember's browser matrix included browsers where accessors were not available. As of Ember 3.0, this is no longer the case.

[RFC#281](https://github.com/emberjs/rfcs/blob/master/text/0281-es5-getters.md) made it possible for applications to use JS `PathExpression`s for accessing properties by installing ES5 getters. This is now available in Ember 3.1. While we loosened the constraint on `get`, RFC#281 did not attempt to loosen the requirement of `set`. This is largely because `set` is the entry point into the change tracking system.

### Notification-based Change Detection

Since the beginning, Ember has used a notification-based system under the hood to perform change detection within an application. Effectively, this means that the system has push-based semantics since we "push" out notifications to subscribers to keep values up to date. To illustrate how this works in practice, Lets consider the following example.

```js
import Component from '@ember/component';
import { computed } from '@ember/object';

export Component.extend({
  init() {
    this._super(...arguments);
    this.firstName = undefined;
    this.lastName = undefined;
  },
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstname} ${this.lastName}`;
  }),

  actions: {
    updateFirstName(first) {
      this.set('firstName', first);
    },
    updateLastName(last) {
      this.set('lastName', last);
    }
  }
});
```

In the example above we have a computed property which concats `firstName` and `lastName`. When the `updateFirstName` and/or `updateLastName` actions are called we call `set`. Calling `set` on the properties will set the value on the instance and then notify any dependents to invalidate. This means that the computed property will be invalidated and its value will be recomputed the next time it is accessed. While this system works, notifications are eager by nature and can lead to excessive churn. Furthermore, notification based systems typically lead to more memory consumption.

### Pull-based Change Detection

With the introduction of the Glimmer VM we also introduced a new change tracking system known as [references](https://github.com/glimmerjs/glimmer-vm/blob/master/guides/04-references.md). References are the underlying data structure that back `MustacheStatement`s in a template and keep the UI up to date. Unlike notification-based system that backs APIs like `Computed` and `Observable`, references have pull-based semantics. This means that instead of having subcribers and notifiers, we must have a descrete signal to "capture" the underlying values. This can be best illustrated by the following example:

```js
let foo = 1;
let bar = 2;

let fooReference = {
  value() {
    return foo;
  }
};

let barReference = {
  value() {
    return bar;
  }
};

let fooPlusBarReference = {
  value() {
    return fooReference.value() + barReference.value();
  }
};

fooPlusBarReference.value(); // => 3

foo = 2;

fooPlusBarReference.value(); // => 4

bar = 3;

fooPlusBarReference.value(); // => 5
```

As you can see, `fooPlusBarReference` composes `fooReference` and `barReference` instead of accessing the variables directly. As `foo` and `bar` change over time, the `fooPlusBarReference` stays up-to-date and returns the correct result of `foo + bar`.

While references allow us to model arbitrarily complex computation, they require a mechanism to ensure that we only recompute values if the underlying data has changed. To solve this references use revision tags to determine the freshness of the computation.

### Revision Tags

The revision tag system is based around the idea of a global revision counter. The global revision counter is a monotonically increasing sequence, which is just a fancy way of saying that it's a global number that only increases but never decreases.

Conceptually, a discrete system can be modeled as a series of state transitions. The global revision counter is incremented by one every time the system undergoes a state transition. In other words, the global revision counter is incremented every time a variable is changed in the system.

In addition to the global counter, each (observable) object in the system has an internal "last modified" revision counter. Every time an object is modified, this counter will be set to the current value of the global revision counter (after the global revision counter has been incremented).

The easiest way to implement this is with a collaborating object model. For example, all observable changes in Ember are already required to go through the `set` function. This is a perfect opportunity to increment both the global and per-object revision counters.

These primitives lay out the foundation for the revision tag system. If we can assume each observable object in the system has a `lastModified` counter, then it would be possible to construct an entity tag for each object where the validation ticket is the current value of the `lastModified` counter. This tag will guarantee the freshness of any first-level path lookups on that object. In other words, all property lookups on an object will have the same result so long as the `lastModified` remains unchanged.

```js
let $REVISION_COUNTER = 1;

class DirtyableTag  {
  constructor() {
    this.lastRevision = $REVISION_COUNTER;
  }

  value(): Revision {
    return this.lastRevision;
  }

  validate(ticket): boolean {
    return ticket === this.lastRevision;
  }

  dirty() {
    this.lastRevision = ++$REVISION_COUNTER;
  }
}

function set(object, property, value) {
  object.tag.dirty();
  return object[property] = value;
}

///

let person = {
  tag: new DirtyableTag(),
  name: 'Godfrey Chan'
};

person.tag.value(); // => 1

set(person, 'name', 'Yehuda Katz');

person.tag.validate(1); // => false
person.tag.value(); // => 2
```