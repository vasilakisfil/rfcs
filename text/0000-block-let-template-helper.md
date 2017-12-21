- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Block `let` template helper

## Summary

Deprecate the `with` helper and introduce a new `let` helper without the conditional behaviour.

## Motivation

[RFC #200](https://github.com/emberjs/rfcs/pull/200) introduced the concept of the `let` helper,
laying out the rationale for why it is a useful addition to Ember's templating system.
It introduced both the block form of the helper, as well as the inline form.
However, the inline form of `let`


It also laid out the case for why `with`'s conditional behaviour is confusing.

By conditional behaviour what is meant is that `with` is passed a falsey value,
it will not render the content of the block.
To quote RFC #200:

> Whereas with will not render if it is passed an empty list, null, undefined, false, or an empty string. This counterintuitive behavior leads to nasty suprises and difficult to workaround situations.

```handlebars
{{#with (array) as |list|}}
  {{!does not render}}
  You'll never see me!
{{/with}}
```

### Deprecating the `with` helper

Why introduce a new helper? `with` has conditional semantics, which confused people.


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

## Detailed design

### Implementing `let`

The `let` helper should be implemented as a built-in helper, with the following semantics:

* Only the block form is available
* The block is always rendered
* Every positional argument passed to the helper is yielded back out in the same order
* Inline form issues an error, linking users to documentation

There already exists [an implementation in the codebase](https://github.com/emberjs/ember.js/blob/9536e137b9e1a39411b7fd4e8ca0e7fbb341ef17/packages/ember-glimmer/tests/integration/syntax/experimental-syntax-test.js#L6-L37) that could be used as a basis.

### Deprecating `with`

## How We Teach This

The introduction of the `let` helper brings no new concepts.
It touched on the concepts of block helpers, how to pass arguments to them,
and how to use block parameters (`as |foo|`), which should already be introduced in the literature.

The Guides already possess a section dedicated to Templates, with multiple mentions of helpers.
`let` would likely be documented in the [Built-in Helpers](https://guides.emberjs.com/v2.17.0/templates/built-in-helpers/) guide alonside the others.

If this RFC is approved, the `let` will initially only support the block form.
This means that only the following form is available for users:

```handlebars
{{#let 1 2 3 as |one two three|}}
  A, B, C, easy as {{one}}, {{two}}, {{three}}
```

This could also be enforced by issuing a helpful error when `let` is used in the inline form.

## Drawbacks

As with the addition of any API, we will be increasing the cognitive load of learners and users,
as it is one more piece of information to obtain and retain.

The cost of learning this API is mitigated by the fact that its effects are very localized.
It is a template helper, so it will only affect templates.
It is not required for general usage of Ember, unlike something like `link-to`,
so you can learn the helper at your own pace.
And lastly, if you do use it or encounter it in code, only the markup inside the `{{#let}}{{/let}}` block is affected,
making it easier to think about.

## Alternatives

### Adding named arguments to `with`

[RFC #202](https://github.com/emberjs/rfcs/pull/202) proposes to add named arguments to `with`.

I feel it is less practical to add a new mode to the helper where it always renders,
when its semantics are already confusing to users.
The RFC #202 proposal also presents the problem of bringing back context-switching helpers,
as it proposes omitting block arguments (`as |bar|` in `{{#with foo as |bar|}}`).

### Remove the conditional behavior of `with`

Making the `with` helper unconditionally render the block would be a major breaking change of its semantics,
and would likely affect existing applications in insidious ways.
For this reason, I reject this alternative out of the gate.

## Unresolved questions

### Inline form

As mentioned, this RFC was meant to focus on the block form of the `let` helper.
As such, the discussion and design of the inline form of `let` should be left for a subsequent RFC.

### `if-let`, `let*` and others

RFC #200 also proposes the introduction of `if-let` to mimic the behaviour of `with`,
but with a more explicit name, and `let*` to allow bindings to happen in sequence,
thus being able to use previously declared bindings in the same `let` (`{{let* a=1 b=(sum a 5)}}`).

These could also be addressed in subsequent RFCs,
focused on the specificities of each proposal.
