- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Block `let` template helper

## Summary

Introduce a block form `let` template helper to bind values and deprecate `with`.

## Motivation

### Introducing the block `let` helper

The idea of having a `let` template helper, and what that entails, was introduced in [`RFC #200`](https://github.com/emberjs/rfcs/pull/200).
This RFC is not intended to supplant it, but to reduce the scope to the block form of the template helper.

Being able to bind values is an important capability of languages, including templating ones.
To use an example from `RFC #200`, if you have the following code:

```handlebars
Welcome back {{concat (capitalize person.firstName) ' ' (capitalize person.lastName)}}

Account Details:
First Name: {{capitalize person.firstName}}
last Name: {{capitalize person.lastName}}
```

You are able to avoid needless repetition with the `let` helper:

```handlebars
{{#let (capizalize person.firstName) (capitalize person.lastName)
  as |firstName lastName|
}}
  Welcome back {{concat firstName ' ' lastName}}

  Account Details:
  First Name: {{firstName}}
  last Name: {{lastName}}
{{/let}}
```

### Deprecating the `with` helper

Why introduce a new helper? `with` has conditional semantics, which confused people.

## Detailed design

> This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How We Teach This

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

### Named arguments and named block arguments

???

