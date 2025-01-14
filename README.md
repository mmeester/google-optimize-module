# nuxt-google-optimize

[![npm (scoped with tag)](https://img.shields.io/npm/v/nuxt-google-optimize/latest.svg?style=flat-square)](https://npmjs.com/package/nuxt-google-optimize)
[![npm](https://img.shields.io/npm/dt/nuxt-google-optimize.svg?style=flat-square)](https://npmjs.com/package/nuxt-google-optimize)
[![CircleCI](https://img.shields.io/circleci/project/github/alibaba-aero/nuxt-google-optimize.svg?style=flat-square)](https://circleci.com/gh/alibaba-aero/nuxt-google-optimize)
[![Codecov](https://img.shields.io/codecov/c/github/alibaba-aero/nuxt-google-optimize.svg?style=flat-square)](https://codecov.io/gh/alibaba-aero/nuxt-google-optimize)
[![Dependencies](https://david-dm.org/alibaba-aero/nuxt-google-optimize/status.svg?style=flat-square)](https://david-dm.org/alibaba-aero/nuxt-google-optimize)
[![js-standard-style](https://img.shields.io/badge/code_style-standard-brightgreen.svg?style=flat-square)](http://standardjs.com)

> SSR friendly Google Optimize module for Nuxt.js

[📖 **Release Notes**](./CHANGELOG.md)

## Features

- Support multiple experiments (AB or MVT[Multi-Variant])
- Auto assign experiment/variant to users
- SSR support using cookies
- CSS and state injection
- Automatically revoke expired experiments from testers
- Ability to assign experiments based on context conditions (Route, State, etc)

## Setup

- Add `nuxt-google-optimize` dependency using yarn or npm to your project
```sh
yarn add nuxt-google-optimize-next
```
OR
```sh
npm install nuxt-google-optimize-next --save
```

- Add `nuxt-google-optimize-next` to `modules` section of `nuxt.config.js`

```js
{
  modules: [
    'nuxt-google-optimize-next',
  ],

  // Optional options
  googleOptimize: {
    /* module options */
  }
}
```

## Options

### `experimentsDir`

- Type: `String`
- Default: `'~/experiments'`

Provide path where experiments are located.

### `maxAge`

- Type: `Number`
- Default: `60 * 60 * 24 * 7`

Provides default max age for user to test.

### `pushPlugin`

- Type: `Boolean`
- Default: `true`

### `eventHandler`

- Type: `Function`
- Default:
```js
(experiment, context) => {
  const exp =
    experiment.experimentID + '.' + experiment.$variantIndexes.join('-')
  const { ga } = window
  if (!ga) return
  ga('set', 'exp', exp)
}
```

Provide custom event handler to send experiment details.

Usage example:
```js
googleOptimize: {
  eventHandler: (experiment, context) => {
    const exp = experiment.experimentID + '.' + experiment.$variantIndexes.join('-')
    const { ga } = window
    if (!ga) return
    ga('set', 'exp', exp)
  }
}
```

### `excludeBots`

- Type: `Boolean`
- Default: `true`

### `botExpression`

- Type: `RegExp`
- Default: `/(bot|spider|crawler)/i`


## Usage

Create `experiments` directory inside your project.

Create `experiments/index.js` to define all available experiments:

```js
import backgroundColor from './background-color'

export default [
  backgroundColor
]
```

### Creating an experiment

Each experiment should export an object to define itself.

`experiments/background-color/index.js`:

```js
export default {
  // A helper exp-{name}-{var} class will be added to the root element
  name: 'background-color',

  // Google optimize experiment id
  experimentID: '....',

  // [optional] specify number of sections for MVT experiments
  // sections: 1,

  // [optional] maxAge for a user to test this experiment
  // maxAge: 60 * 60 * 24, // 24 hours,

  // [optional] Enable/Set experiment on certain conditions
  // isEligible: ({ route }) => route.path !== '/foo'

  // Implemented variants and their weights
  variants: [
    { weight: 0 }, // <-- This is the default variant
    { weight: 2 },
    { weight: 1 }
  ],
}
```

### `$exp`

Global object `$exp` will be universally injected in the app context to determine the currently active experiment.

It has the following keys:

```json6
{
  // Index of currently active experiment
  "$experimentIndex": 0,

  // Index of currently active experiment variants
  "$variantIndexes": [
    1
  ],

  // Same as $variantIndexes but each item is the real variant object
  "$activeVariants": [
    {
      /* */
    }
  ],

  // Classes to be globally injected (see global style tests section)
  "$classes": [
    "exp-background-color-1" // exp-{experiment-name}-{variant-id}
  ],

  // All of the keys of currently active experiment are available
  "name": "background-color",
  "experimentID": "testid",
  "sections": 1,
  "maxAge": 60,
  "variants": [
    /* all variants */
  ]
}
```

**Using inside components:**

```html
<script>
export default {
  methods: {
    foo() {
      // You can use this.$exp here
    }
  }
}
</script>
```

**Using inside templates:**

```html
<div v-if="$exp.name === 'something'">
  <!-- You can optionally use $exp.$activeVariants and $exp.$variantIndexes here -- >
  ...
</div>
<div v-else>
  ...
</div>
```

### Global style tests

Inject global styles to page body.

`layouts/default.vue`:

```vue
<template>
  <nuxt/>
</template>

<script>
export default {
      head () {
        return {
            bodyAttrs: {
                class: this.$exp.$classes.join(' ')
            }
        }
    },
}
</script>
```

If you have custom CSS for each test, you can import it inside your experiment's `.js` file.

`experiments/background-color/index.js`:

```js
import './styles.scss'
```

**With Sass:**

```scss
.exp-background-color {
  // ---------------- Variant 1 ----------------
  &-1 {
    background-color: red;
  }
  // ---------------- Variant 2 ----------------
  &-2 {
    background-color: blue;
  }
}
```

**With CSS:**

```css
/* Variant 1 */
.exp-background-color-1 {
  background-color: red;
}

/* Variant 2 */
.exp-background-color-2 {
  background-color: blue;
}
```

## Usage with GTM

- Set `options.eventHandler`:
```js
// GTM module
{
  eventHandler: (experiment, { $gtm }) => {
    const exp = `${experiment.experimentID}.${experiment.$variantIndexes.join( '-' )}`
    $gtm.push({ exp })
  }
}

// Default datalayer
{
  eventHandler: (experiment, { $gtm }) => {
    const exp = `${experiment.experimentID}.${experiment.$variantIndexes.join( '-' )}`
    const { dataLayer } = window
    if (!dataLayer) return
    dataLayer.push({ exp })
  }
}
```
- Edit your "Page view (Google Analytics)" -tag in [Google Tag Manager](https://tagmanager.google.com/#/home)
  - Add new "Fields to Set":
    - Field Name: exp
    - Value: {{googleOptimizeExp}}
- Add new Data Layer Variable called "googleOptimizeExp" as defined above.
  - Variable type: Data Layer Variable
  - Data Layer Variable Name: exp


[Source for this setup lossleader's answer in StackOverflow](https://stackoverflow.com/a/53253769/871677)

Now this module pushes experiment id and variable number to Google Analytics via Google Tag Manager.
experiment.experimentID + '.' + experiment.$variantIndexes.join('-')

## Development

- Clone this repository
- Install dependencies using `yarn install` or `npm install`
- Start development server using `yarn run dev` or `npm run dev`
- Point your browser to `http://localhost:3000`
- You will see a different colour based on the variant set for you
- In order to test your luck, try clearing your cookies and see if the background colour changes or not

## License

[MIT License](./LICENSE) - Alibaba Travels Co

