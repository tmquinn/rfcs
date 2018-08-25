- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Tracked Properties

## Summary

This RFC introduces the concept of tracked properties. Tracked properties are just like normal properties but are the mechanism for opting into change detection. Tracked properties are exposed by Ember as decorators.

## Motivation

Historically, Ember has used a notification-based system under the hood to perform change detection within an application. APIs like `get`, `set`, and `Computed` are all built on top of this system. As mentioned in [RFC#281](https://github.com/emberjs/rfcs/blob/master/text/0281-es5-getters.md#motivation) APIs like `get` and `set` were required due to the fact that these APIs are from a time when JavaScript did not have first-class accessors. As of Ember 3.1 `get` is no longer


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
