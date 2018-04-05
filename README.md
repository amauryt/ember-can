# Ember-can

<p align="center">
  <a href="http://badge.fury.io/js/ember-can" title="Package version">
    <img src="https://badge.fury.io/js/ember-can.svg"/>
  </a>

  <a href="https://travis-ci.org/minutebase/ember-can" title="Ember Observer">
    <img src="http://emberobserver.com/badges/ember-can.svg" alt="Ember Observer"/>
  </a>

  <a href="https://travis-ci.org/minutebase/ember-can" title="Travis CI status">
    <img src="https://travis-ci.org/minutebase/ember-can.svg?branch=master" alt="Travis CI Status"/>
  </a>

  <a href="https://david-dm.org/minutebase/ember-can" title="dependencies status">
    <img src="https://david-dm.org/minutebase/ember-can/status.svg"/>
  </a>
</p>

___

Simple authorisation addon for Ember.

## Installation

Install this addon via ember-cli:

```
ember install ember-can
```

**Compatible with Ember.js `>=2.12.0`**

## Quick Example

You want to conditionally allow creating a new blog post:

```hbs
{{#if (can "write post")}}
  Type post content here...
{{else}}
  You can't write a new post!
{{/if}}
```

We define an ability for the `Post` model in `/app/abilities/post.js`:

```js
// app/abilities/post.js

import { computed } from '@ember/object';
import { Ability } from 'ember-can';

export default Ability.extend({
  canWrite: computed('user.isAdmin', function() {
    return this.get('user.isAdmin');
  })
});
```

We can also re-use the same ability to check if a user has access to a route:

```js
// app/routes/posts/new.js

import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';

export default Route.extend({
  can: service(),

  beforeModel() {
    let result = this._super(...arguments);

    if (!this.get('can').can('write post')) {
      return this.transitionTo('index');
    }

    return result;
  }
});
```

And we can also check the permission before firing action:

```js
import Component from '@ember/component';

export default Component.extend({
  can: service(),

  actions: {
    createPost() {
      let canWrite = this.get('can').can('write post' this.get('post'), { user: this.get('author') });

      if (canWrite) {
        // create post!
      }
    }
  }
});
```

## Helpers

### `can`

The `can` helper is meant to be used with `{{if}}` and `{{unless}}` to protect a block (but can be used anywhere in the template).

```hbs
{{can "doSth in myModel" model extraProperties}}
```
- `"doSth in myModel" ` - The first parameter is a string which is used to find the ability class call the appropriate property (see [Looking up abilities](#looking-up-abilities)).

- `model` - The second parameter is an optional model object which will be given to the ability to check permissions.

- `extraProperties` - The third parameter are extra properties which will be assigned to the ability

**As activities are standard Ember objects and computed properties if anything changes then the view will
automatically update accordingly.**

#### Example
```hbs
{{#if (can "edit post" post)}}
  ...
{{else}}
  ...
{{/if}}
```

As it's a sub-expression, you can use it anywhere a helper can be used.
For example to give a div a class based on an ability you can use an inline if:

```hbs
<div class="{{if (can 'edit post' post) 'is-editable'}}">

</div>
```

### `cannot`

Cannot helper is a negation of `can` helper with the same API.

```hbs
{{cannot "doSth in myModel" model extraProperties}}
```


## Abilities

An ability class protects an individual model which is available in the ability as `model`.

The ability checks themselves are simply standard Ember objects with computed properties:

```js
// app/abilities/post.js

import { computed } from '@ember/object';
import { Ability } from 'ember-can';

export default Ability.extend({
  // only admins can write a post
  canWrite: computed('user.isAdmin', function() {
    return this.get('user.isAdmin');
  }),

  // only the person who wrote a post can edit it
  canEdit: computed('user.id', 'model.author', function() {
    return this.get('user.id') === this.get('model.author');
  })
});

// Usage:
// {{if (can "write post" post) "true" "false"}}
// {{if (can "edit post" post user=author) "true" "false"}}
```

## Additional attributes

If you need more than a single resource in an ability, you can pass them additional attributes.

You can do this in the helpers, for example this will set the `model` to `project` as usual,
but also `member` as a bound property.

```hbs
{{#if (can "remove member from project" project member=member)}}
  ...
{{/if}}
```

Similarly using `can` service you can pass additional attributes after or instead of the resource:

```js
this.get('can').can('edit post', post, { author: bob });
this.get('can').cannot('write post', null, { project: project });
```

These will set `author` and `project` on the ability respectively so you can use them in the checks.

## Looking up abilities

In the example above we said `{{#if (can "write post")}}`, how do we find the ability class & know which property to use for that?

First we chop off the last word as the resource type which is looked up via the container.

The ability file can either be looked up in the top level `/app/abilities` directory, or via pod structure.

Then for the ability name we remove some basic stopwords (of, for in) at the end, prepend with "can" and camelCase it all.

For example:

| String                      | property           | resource                | pod                            |
|-----------------------------|--------------------|-------------------------|--------------------------------|
| write post                  | `canWrite`         | `/abilities/post.js`    | `app/pods/post/ability.js`     |
| manage members in projects  | `canManageMembers` | `/abilities/projects.js`| `app/pods/projects/ability.js` |
| view profile for user       | `canViewProfile`   | `/abilities/user.js`    | `app/pods/user/ability.js`     |

Current stopwords which are ignored are:

* for
* from
* in
* of
* to
* on

## Custom Ability Lookup

The default lookup is a bit "clever"/"cute" for some people's tastes, so you can override this if you choose.

Simply extend the default `CanService` in `app/services/can.js` and override `parse`.

`parse` takes the ability string eg "manage members in projects" and should return an object with `propertyName` and `abilityName`.

For example, to use the format "person.canEdit" instead of the default "edit person" you could do the following:

```js
// app/services/can.js
import Service from 'ember-can/services/can';

export default CanService.extend({
  parse(str) {
    const [abilityName, propertyName] = str.split('.');
    return {
      propertyName,
      abilityName
    }
  }
});
```

## Injecting the user

How does the ability know who's logged in? This depends on how you implement it in your app!

If you're using an `Ember.Service` as your session, you can just inject it into the ability:

```js
// app/abilities/foo.js
import { Ability } from 'ember-can';
import { inject as service } from '@ember/service';

export default Ability.extend({
  session: service()
});
```

The ability classes will now have access to `session` which can then be used to check if the user is logged in etc...

## Components & computed properties

In a  component, you may want to expose abilities as computed properties
so that you can bind to them in your templates.

```js
import Component from '@ember/component';
import { computed } from '@ember/object';

export default Component.extend({
  can: service(), // inject can service

  post: null, // received in higher template

  ability: computed('post', function() {
    return this.get('can').abilityFor('post', this.get('post'));
  }),
});

// {{if ability.canWrite "true" "false"}}
```

## Upgrade guide

See [UPGRADING.md](https://github.com/minutebase/ember-can/blob/master/UPGRADING.md) for more details.

## Testing

Make sure that you've either `ember install`-ed this addon, or run the addon
blueprint via `ember g ember-can`. This is an important step that teaches the
test resolver how to resolve abilities from the file structure.

### Unit testing abilities

An ability unit test will be created each time you generate a new ability via
`ember g ability <name>`. The package currently supports generating QUnit and
Mocha style tests.

### Unit testing in your app

To unit test modules that use the `can` helper, you'll need to explicitly add `needs` for the
ability and helper file like this:
``` needs: ['helper:can', 'ability:foo'] ```

### Integration testing in your app

For integration testing components, you should not need to specify anything explicitly. The
helper and your abilities should be available to your components automatically.

## Development

### Installation

* `git clone https://github.com/minutebase/ember-can.git`
* `cd ember-can`
* `npm install`

### Linting

* `npm run lint:js`
* `npm run lint:js -- --fix`

### Running tests

* `ember test` – Runs the test suite on the current Ember version
* `ember test --server` – Runs the test suite in "watch mode"
* `npm test` – Runs `ember try:each` to test your addon against multiple Ember versions

### Running the dummy application

* `ember serve`
* Visit the dummy application at [http://localhost:4200](http://localhost:4200).

For more information on using ember-cli, visit [https://ember-cli.com/](https://ember-cli.com/).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/minutebase/ember-can. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the Contributor Covenant code of conduct.

## License

This version of the package is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
