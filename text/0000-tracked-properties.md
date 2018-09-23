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

## Motivation

Broadly speaking, most component libraries have taken one of two approaches:

1. Fine-grained property observation, like Ember.
2. Virtual DOM diffing, like React.

Systems that rely on property observation trade reduced initial render performance for improved updating speed. That's because they must install observers on every property that gets rendered into the DOM, which takes time. But if a property changes, updates to the DOM are very targeted and fast.

However, web users expect pages to render near instantly. Setting up observers adds a lot of overhead, which slows down initial render. Virtual DOM-based libraries like React have become very popular, because they prioritize raw render speed by keeping change-tracking bookkeeping to a minimum.

The tradeoff is that updates require re-evaluating the component tree to figure out how the DOM needs to be mutated. Essentially, that means doing a full render pass on a component and all of its children every time a property changes.

While constructing virtual DOM is fast and applying diff updates can be optimized, it's far from instantaneous, particularly as the size of your application grows. As your virtual DOM-based app grows, you will have to do more work to make sure it stays performant.

Tracked properties intend to take a hybrid approach to strike a balance between simplicity in the programming model and performance.

#### ------- TOM SHOULD ADD MORE HERE --------

## Detailed design

Below specifies the `@tracked` decorator and the behavior of data within the system.

### `@tracked` Syntax

`@tracked` is a [property decorator](https://www.typescriptlang.org/docs/handbook/decorators.html#property-decorators) that opts the property into mutations. It can be used on class property fields or getter accessor methods.

```js
import { tracked } from '@ember/object';
import Component from '@ember/component';

export default class Employee extends Component {
  @tracked name = 'Fred';
  @tracked get fullName() {
    return `${this.name} Brooks`;
  }
}
```

The decorator is **not** a [decorator factory](https://www.typescriptlang.org/docs/handbook/decorators.html#decorator-factories).

```js
import { tracked } from '@ember/object';
import Component from '@ember/component';

export default class Employee extends Component {
  @tracked name = 'Fred';
  @tracked() age = 87; // Error
  @tracked get fullName() {
    return `${this.name} Brooks`;
  }
}
```

Doing so will result in the following error:

```
You attempted to use "@tracked" as a factory function in the "Employee" component which is not supported. Please change "@tracked()" to "@tracked".
```

### `@tracked` Runtime Semantics

Tracked properties act like normal javascript properties. This means that accessing and assigning just works without the need for `Ember.get` or `Ember.set`. While these APIs are no longer required, they are compatible and will continue to work.


```js
import { tracked } from '@ember/object';
import Component from '@ember/component';

export default class Employee extends Component {
  @tracked name = 'Fred';
  changeName(newName) {
    this.name = newName;
  }

  @tracked get fullName() {
    return `${this.name} Brooks`;
  }

  set fullName(newFullName) {
    let [first] = newFullName.split(' ');
    this.name = first;
  }
  // ...
}
```

Just like calling `Ember.set`, assigning a value to a tracked property will schedule a rerender. Setting the same property multiple times within the same render will result in the following error:

```
You modified Employee.name twice in a single render.
```

In the event you need to re-assign a value within the same render you must schedule the set within a runloop.

```js
import Component from '@ember/component';
import { once } from '@ember/runloop';

export default class Employee extends Component {
  @tracked placeHolderHeight = 100;

  didInsertElement() {
    once('afterRender', () => {
      this.placeHolderHeight = this.element.clientHeight;
    });
  }
}
```

Semantically a `@tracked` setter does not make sense, due to the fact that a setter is always talking about setting a new value which will typically result in the getter being recomputed. To guide developers down the correct path the following assertion will be thrown for installing a tracked property on a setter:

```
You attempted to use "@tracked" on the "keyName" setter in the "Employee" component. "@tracked" setters are not required, please remove the decorator.
```

### Computed Properties & Auto-tracking

Under the hood tracked properties use the Glimmer VM's [reference](https://github.com/glimmerjs/glimmer-vm/blob/master/guides/04-references.md) and [validator](https://github.com/glimmerjs/glimmer-vm/blob/master/guides/05-validators.md) systems to perform change detection. These subsystems allow us to model similar semantics to Ember's computed properties. For instance let's look at a conical example of computed properties:


```js
import Component from '@ember/component';
import { computed } from '@ember/object';

export default Component.extend({
  firstName: 'Chad',
  lastName: 'Hietala',
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`
  })
});
```

With tracked properties we are able to express the same example as such:

```js
import Component from '@ember/component';
import { tracked } from '@ember/object';

export default class extends Component {
  @tracked firstName = 'Chad';
  @tracked lastName = 'Hietala';
  @tracked get fullName() {
    return `${this.firstName} ${this.lastName}`
  }
}
```

There are 2 key differentiators with the tracked implementation:

1. It does **not** require the enumeration of the dependent keys
2. It uses an ES5 getter

We are able to elide the dependent key enumeration here because we are able to detect that when we access `fullName` we also access `firstName` and `lastName`. Therefore we can construct a "cache key" for `fullName` that is the combination of `firstName` and `lastName`'s "cache keys". This mens if either `firstName` or `lastName` update, then `fullName` needs to be recomputed the next time it is accessed. These are the exact semantics of Ember's `computed`.

#### Observability

TODO!

#### Arrays

Historically, Ember has used the [`[]` and `@each` mircosyntaxes](https://guides.emberjs.com/release/object-model/computed-properties-and-aggregate-data/) that allowed you create computed property based on the contents of arrays. For example:

```js
import EmberObject, { computed } from '@ember/object';
import Component from '@ember/component';

export default Component.extend({
  todos: null,

  init() {
    this.set('todos', [
      EmberObject.create({ title: 'Buy food', isDone: true }),
      EmberObject.create({ title: 'Eat food', isDone: false }),
      EmberObject.create({ title: 'Catalog Tomster collection', isDone: true }),
    ]);
  },

  incomplete: computed('todos.@each.isDone', function() {
    let todos = this.todos;
    return todos.filterBy('isDone', false);
  }),

  titles: computed('todos.[]', function() {
    return this.todos.mapBy('title');
  })
});
```

While this did allow you to create computed properties that were aware of arrays, it effectievly meant that we had an explosion of extra book keeping that we needed to perfom. Furthermore, this microsyntax is something that is something that developers must learn and at times can get wrong leading to further runtime issues.

With tracked properties we have explicitly chosen to adopt the immutable pattern for dealing with arrays. In practice what this means is:

1. Save the array as an "atom" in a tracked property on the object.
2. To change state, replace the root "atom" with a new copy of the array.

This approach helps you make changes predictable: you know that an array can only be updated when that root tracked property changes. JavaScript destructuring syntax can make this quite elegant:

```js
import Component from '@ember/component';
import { tracked } from '@ember/object';

export default class extends Component {
  @tracked todos = [
    { title: 'Buy food', isDone: true },
    { title: 'Eat food', isDone: false },
    { title: 'Catalog Tomster collection', isDone: true },
  ];

  @tracked get incomplete() {
    return this.todos.filter(todo => !todo.isDone);
  }

  @tracked get titles() {
    return this.todos.map(todo => todo.title);
  }

  addTodo(item) {
    this.cartItems = [...this.cartItems, item];
  }
}
```

In the example above, calling `addItem` replaces the entire array with the new item added as the last item in the array. Setting `cartItems` to the new array will cause both `incomplete` and `titles` to be recomputed the next time they are accessed.

## Interoperability

Since Ember 2.10 Ember has been utilizing references and validators to perform change tracking for values that were interacting with the templating layer. This has allowed us to ensure that it is possible to model the vast majority of Ember's change tracking semantics with the new system. What this means in practice is that all of Ember's existing APIs can be interleaved with tracked properties. For instance:

```hbs
{{! app/templates/application.hbs }}

<MyParent @firstName={{this.model.firstName}} @lastName={{this.model.lastName}} as |fullName|>
  <MyChild @fullName={{fullName}}>
</MyParent>
```

Where `MyParent` is using computed properties like:

```js
// app/components/my-parent.js

import { computed } from '@ember/object';
import Component from '@ember/component';

export default Component.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  });
})
```

```hbs
{{! app/templates/components/my-parent.hbs}}
{{yield this.fullName}}
```

And where `MyChild` is using tracked properties like:

```js
// app/components/my-parent.js

import { tracked } from '@ember/object';
import Component from '@ember/component';

export default class extends Component {
  @tracked get greeting() {
    return `Hello ${this.fullName}!`;
  }
}
```

```hbs
{{! app/templates/components/my-parent.hbs}}
{{this.greeting}}
```

## How we teach this

Tracked properties are intended to simplify the programming model by removing concepts. For instance people familiar with JavaScript do not need to learn a propritary setter method for updating data. They also don't need to learn about dependent keys and remembering to properly enumerate them. It also allows us to redirect people things like Mozilla's documentation if people are not familiar with things like accessor methods.

## Drawbacks

With tracked properties we are giving up some fine grained control over observing when data has changed for array mutations. As noted in the design section, arrays would need to use a immutable pattern for having their changes reflected through out the system.

This solution also pushes the control of what is and is not observable by the system out to the developer. This means that developers needs to manually mark the properties that can change. While this is more typing for the developer it allows us to clearly seperate out static and dynamic data. It also means that Ember only has to do the booking keeping for the explicity annotated properties.

## Alternatives

We could continue to use `computed` and just make a decorator form of it. It is possible to make `computed` infer it's dependencies just like `tracked`. The downside here is that we would then have to deprecate the array microsyntaxes from `computed` as those cases are not observable through the auto tracking functionality. This would likely create unnecessary deprecation noise and it's likely a better option to provide `tracked` that can be opted into.

## Unresolved questions

TBD?
