# Using with Vuex

In this guide, we'll see how to test Vuex in components with `vue-test-utils`.

## Mocking Actions

Let’s look at some code.

This is the component we want to test. It calls Vuex actions.

``` html
<template>
  <div class="text-align-center">
    <input type="text" @input="actionInputIfTrue" />
    <button @click="actionClick()">Click</button>
  </div>
</template>

<script>
import { mapActions } from 'vuex'

export default{
  methods: {
    ...mapActions([
      'actionClick'
    ]),
    actionInputIfTrue: function actionInputIfTrue (event) {
      const inputValue = event.target.value
      if (inputValue === 'input') {
        this.$store.dispatch('actionInput', { inputValue })
      }
    }
  }
}
</script>
```

For the purposes of this test, we don’t care what the actions do, or what the store looks like. We just need to know that these actions are being fired when they should, and that they are fired with the expected value.

To test this, we need to pass a mock store to Vue when we shallow our component.

Instead of passing the store to the base Vue constructor, we can pass it to a - [localVue](../api/options.md#localvue). A localVue is a scoped Vue constructor that we can make changes to without affecting the global Vue constructor.

Let’s see what this looks like:

``` js
import { shallow, createLocalVue } from 'vue-test-utils'
import Vuex from 'vuex'
import Actions from '../../../src/components/Actions'

const localVue = createLocalVue()

localVue.use(Vuex)

describe('Actions.vue', () => {
  let actions
  let store

  beforeEach(() => {
    actions = {
      actionClick: jest.fn(),
      actionInput: jest.fn()
    }
    store = new Vuex.Store({
      state: {},
      actions
    })
  })

  it('calls store action "actionInput" when input value is "input" and an "input" event is fired', () => {
    const wrapper = shallow(Actions, { store, localVue })
    const input = wrapper.find('input')
    input.element.value = 'input'
    input.trigger('input')
    expect(actions.actionInput).toHaveBeenCalled()
  })

  it('does not call store action "actionInput" when input value is not "input" and an "input" event is fired', () => {
    const wrapper = shallow(Actions, { store, localVue })
    const input = wrapper.find('input')
    input.element.value = 'not input'
    input.trigger('input')
    expect(actions.actionInput).not.toHaveBeenCalled()
  })

  it('calls store action "actionClick" when button is clicked', () => {
    const wrapper = shallow(Actions, { store, localVue })
    wrapper.find('button').trigger('click')
    expect(actions.actionClick).toHaveBeenCalled()
  })
})
```

What’s happening here? First we tell Vue to use Vuex with the `localVue.use` method. This is just a wrapper around `Vue.use`.

We then make a mock store by calling new `Vuex.store` with our mock values. We only pass it the actions, since that’s all we care about.

The actions are [jest mock functions](https://facebook.github.io/jest/docs/en/mock-functions.html). These mock functions give us methods to assert whether the actions were called or not.

We can then assert in our tests that the action stub was called when expected.

Now the way we define the store might look a bit foreign to you.

We’re using `beforeEach` to ensure we have a clean store before each test. `beforeEach` is a mocha hook that’s called before each test. In our test, we are reassigning the store variables value. If we didn’t do this, the mock functions would need to be automatically reset. It also lets us change the state in our tests, without it affecting later tests.

The most important thing to note in this test is that **we create a mock Vuex store and then pass it to `vue-test-utils`**.

Great, so now we can mock actions, let’s look at mocking getters.

## Mocking Getters


``` html
<template>
  <div>
    <p v-if="inputValue">{{inputValue}}</p>
    <p v-if="clicks">{{clicks}}</p>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'

export default{
  computed: mapGetters([
    'clicks',
    'inputValue'
  ])
}
</script>
```

This is a fairly simple component. It renders the result of the getters `clicks` and `inputValue`. Again, we don’t really care about what those getters returns – just that the result of them is being rendered correctly.

Let’s see the test:

``` js
import { shallow, createLocalVue } from 'vue-test-utils'
import Vuex from 'vuex'
import Actions from '../../../src/components/Getters'

const localVue = createLocalVue()

localVue.use(Vuex)

describe('Getters.vue', () => {
  let getters
  let store

  beforeEach(() => {
    getters = {
      clicks: () => 2,
      inputValue: () => 'input'
    }

    store = new Vuex.Store({
      getters
    })
  })

  it('Renders "state.inputValue" in first p tag', () => {
    const wrapper = shallow(Actions, { store, localVue })
    const p = wrapper.find('p')
    expect(p.text()).toBe(getters.inputValue())
  })

  it('Renders "state.clicks" in second p tag', () => {
    const wrapper = shallow(Actions, { store, localVue })
    const p = wrapper.findAll('p').at(1)
    expect(p.text()).toBe(getters.clicks().toString())
  })
})
```

This test is similar to our actions test. We create a mock store before each test, pass it as an option when we call `shallow`, and assert that the value returned by our mock getters is being rendered.

This is great, but what if we want to check our getters are returning the correct part of our state?

## Mocking with Modules

[Modules](https://vuex.vuejs.org/en/modules.html) are useful for separating out our store into manageable chunks. They also export getters. We can use these in our tests.

Let’s look at our component:

``` html
<template>
  <div>
    <button @click="moduleActionClick()">Click</button>
    <p>{{moduleClicks}}</p>
  </div>
</template>

<script>
import { mapActions, mapGetters } from 'vuex'

export default{
  methods: {
    ...mapActions([
      'moduleActionClick'
    ])
  },

  computed: mapGetters([
    'moduleClicks'
  ])
}
</script>
```

Simple component that includes one action and one getter.

And the test:

``` js
import { shallow, createLocalVue } from 'vue-test-utils'
import Vuex from 'vuex'
import Modules from '../../../src/components/Modules'
import module from '../../../src/store/module'

const localVue = createLocalVue()

localVue.use(Vuex)

describe('Modules.vue', () => {
  let actions
  let state
  let store

  beforeEach(() => {
    state = {
      module: {
        clicks: 2
      }
    }

    actions = {
      moduleActionClick: jest.fn()
    }

    store = new Vuex.Store({
      state,
      actions,
      getters: module.getters
    })
  })

  it('calls store action "moduleActionClick" when button is clicked', () => {
    const wrapper = shallow(Modules, { store, localVue })
    const button = wrapper.find('button')
    button.trigger('click')
    expect(actions.moduleActionClick).toHaveBeenCalled()
  })

  it('Renders "state.inputValue" in first p tag', () => {
    const wrapper = shallow(Modules, { store, localVue })
    const p = wrapper.find('p')
    expect(p.text()).toBe(state.module.clicks.toString())
  })
})
```

### Resources

- [Example project for this guide](https://github.com/eddyerburgh/vue-test-utils-vuex-example)
- [`localVue`](../api/options.md#localvue)
- [`createLocalVue`](../api/createLocalVue.md)
