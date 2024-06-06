# Zustand
A small, fast, and scalable bearbones state management solution.

## First create a store
Your store is a hook!

> The `set` function to merges state

```js
import { create } from 'zustand'

const useStore = create((set) => ({
  bears: 0,
  increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
  removeAllBears: () => set({ bears: 0 }),
  updateBears: (newBears) => set({ bears: newBears }),
}))
```

You can use the hook anywhere

```js
function BearCounter() {
  const bears = useStore((state) => state.bears)
  return <h1>{bears} around here...</h1>
}

function Controls() {
  const increasePopulation = useStore((state) => state.increasePopulation)
  return <button onClick={increasePopulation}>one up</button>
}
```

## Updating State

Updating state with Zustand is simple! Call the provided `set` function with the new state, and it will be shallowly merged with the existing state in the store.

**Normal approach**
```js
normalInc: () =>
    set((state) => ({
      deep: {
        ...state.deep,
        nested: {
          ...state.deep.nested,
          obj: {
            ...state.deep.nested.obj,
            count: state.deep.nested.obj.count + 1
          }
        }
      }
    })),
```

**With Immer**
Many people use Immer to update nested values. Immer can be used anytime you need to update nested state such as in React, Redux and of course, Zustand!

```js
immerInc: () =>
    set(produce((state: State) => { ++state.deep.nested.obj.count })),
```

## Immutatble state and merging
Here's a typical example:

```js
import { create } from 'zustand'

const useCountStore = create((set) => ({
  count: 0,
  inc: () => set((state) => ({ count: state.count + 1 })),
}))
```

The set function is to update state in the store. Because the state is immutable, it should have been like this:

```js
set((state) => ({ ...state, count: state.count + 1 }))
```

However, as this is a common pattern, set actually merges state, and we can skip the ...state part:

```js
set((state) => ({ count: state.count + 1 }))
```

**Nested objects**
The set function merges state at only one level. If you have a nested object, you need to merge them explicitly. You will use the spread operator pattern like so:

```js
import { create } from 'zustand'

const useCountStore = create((set) => ({
  nested: { count: 0 },
  inc: () =>
    set((state) => ({
      nested: { ...state.nested, count: state.nested.count + 1 },
    })),
}))
```

### Replace flag
To disable the merging behavior, you can specify a replace boolean value for set like so:

```js
set((state) => newState, true)
```

## Flux inspired practice
### Recommended patterns
**Single store**
Your applications global state should be located in a single Zustand store.

**Use `set` / `setState` to update the store**
Always use set (or setState) to perform updates to your store. set (and setState) ensures the described update is correctly merged and listeners are appropriately notified.

**Colocate store actions**
In Zustand, state can be updated without the use of dispatched actions and reducers found in other Flux libraries. These store actions can be added directly to the store as shown below.

```js
const useBoundStore = create((set) => ({
  storeSliceA: ...,
  storeSliceB: ...,
  storeSliceC: ...,
  updateX: () => set(...),
  updateY: () => set(...),
}))
```

**Redux-like patterns**
If you can't live without Redux-like reducers, you can define a dispatch function on the root level of the store:

```js
const types = { increase: 'INCREASE', decrease: 'DECREASE' }

const reducer = (state, { type, by = 1 }) => {
  switch (type) {
    case types.increase:
      return { grumpiness: state.grumpiness + by }
    case types.decrease:
      return { grumpiness: state.grumpiness - by }
  }
}

const useGrumpyStore = create((set) => ({
  grumpiness: 0,
  dispatch: (args) => set((state) => reducer(state, args)),
}))

const dispatch = useGrumpyStore((state) => state.dispatch)
dispatch({ type: types.increase, by: 2 })
```

You could also use our redux-middleware. It wires up your main reducer, sets initial state, and adds a dispatch function to the state itself and the vanilla api.

```js
import { redux } from 'zustand/middleware'

const useReduxStore = create(redux(reducer, initialState))
```

## Auto Generating Selectors

We recommend using selectors when using either the properties or actions from the store. You can access values from the store like so:

```js
const bears = useBearStore((state) => state.bears)
```

### create the following function `createSelectors`

```js
import { StoreApi, UseBoundStore } from 'zustand'

type WithSelectors<S> = S extends { getState: () => infer T }
  ? S & { use: { [K in keyof T]: () => T[K] } }
  : never

const createSelectors = <S extends UseBoundStore<StoreApi<object>>>(
  _store: S,
) => {
  let store = _store as WithSelectors<typeof _store>
  store.use = {}
  for (let k of Object.keys(store.getState())) {
    ;(store.use as any)[k] = () => store((s) => s[k as keyof typeof s])
  }

  return store
}
```

If you have a store like this:

```js
interface BearState {
  bears: number
  increase: (by: number) => void
  increment: () => void
}

const useBearStoreBase = create<BearState>()((set) => ({
  bears: 0,
  increase: (by) => set((state) => ({ bears: state.bears + by })),
  increment: () => set((state) => ({ bears: state.bears + 1 })),
}))
```

Apply that function to your store:

```js
const useBearStore = createSelectors(useBearStoreBase)
```

Now the selectors are auto generated and you can access them directly.

```js
// get the property
const bears = useBearStore.use.bears()

// get the action
const increment = useBearStore.use.increment()
```

## Practice with no store actions

The recommended usage is to colocate actions and states within the store (let your actions be located together with your state).

```js
export const useBoundStore = create((set) => ({
  count: 0,
  text: 'hello',
  inc: () => set((state) => ({ count: state.count + 1 })),
  setText: (text) => set({ text }),
}))
```

An alternative approach is to define actions at module level, external to the store.

```js
export const useBoundStore = create(() => ({
  count: 0,
  text: 'hello',
}))

export const inc = () =>
  useBoundStore.setState((state) => ({ count: state.count + 1 }))

export const setText = (text) => useBoundStore.setState({ text })
```

This has a few advantages:
- It doesn't require a hook to call an action
- It facilitates code splitting.

## TypeScript Guide

https://docs.pmnd.rs/zustand/guides/typescript

## Testing

https://docs.pmnd.rs/zustand/guides/testing

## Calling actions outside a React event handler in pre React 18

> React zombie-child effect

https://docs.pmnd.rs/zustand/guides/event-handler-in-pre-react-18

## Map and Set Usage

You need to wrap Maps and Sets inside an object. When you want its update to be reflected (e.g. in React), you do it by calling setState on it:

```js
import { create } from 'zustand'

const useFooBar = create(() => ({ foo: new Map(), bar: new Set() }))

function doSomething() {
  // doing something...

  // If you want to update some React component that uses `useFooBar`, you have to call setState
  // to let React know that an update happened.
  // Following React's best practices, you should create a new Map/Set when updating them:
  useFooBar.setState((prev) => ({
    foo: new Map(prev.foo).set('newKey', 'newValue'),
    bar: new Set(prev.bar).add('newKey'),
  }))
}
```

## Connect to state with URL

https://docs.pmnd.rs/zustand/guides/connect-to-state-with-url-hash

## How to reset state

The following pattern can be used to reset the state to its initial value

```js
import { create } from 'zustand'

// define types for state values and actions separately
type State = {
  salmon: number
  tuna: number
}

type Actions = {
  addSalmon: (qty: number) => void
  addTuna: (qty: number) => void
  reset: () => void
}

// define the initial state
const initialState: State = {
  salmon: 0,
  tuna: 0,
}

// create store
const useSlice = create<State & Actions>()((set, get) => ({
  ...initialState,
  addSalmon: (qty: number) => {
    set({ salmon: get().salmon + qty })
  },
  addTuna: (qty: number) => {
    set({ tuna: get().tuna + qty })
  },
  reset: () => {
    set(initialState)
  },
}))
```

**Resetting multiple stores at once**

https://docs.pmnd.rs/zustand/guides/how-to-reset-state

## Initialize state with props

https://docs.pmnd.rs/zustand/guides/initialize-state-with-props

## Slices Pattern

### Slicing the store into smaller stores

Your store can become bigger and bigger and tougher to maintain as you add more features.

You can divide your main store into smaller individual stores to achieve modularity. This is simple to accomplish in Zustand!

The first individual store:

```js
export const createFishSlice = (set) => ({
  fishes: 0,
  addFish: () => set((state) => ({ fishes: state.fishes + 1 })),
})
```

Another individual store:

```js
export const createBearSlice = (set) => ({
  bears: 0,
  addBear: () => set((state) => ({ bears: state.bears + 1 })),
  eatFish: () => set((state) => ({ fishes: state.fishes - 1 })),
})
```

You can now combine both the stores into one bounded store:

```js
import { create } from 'zustand'
import { createBearSlice } from './bearSlice'
import { createFishSlice } from './fishSlice'

export const useBoundStore = create((...a) => ({
  ...createBearSlice(...a),
  ...createFishSlice(...a),
}))
```

and more...
https://docs.pmnd.rs/zustand/guides/slices-pattern

## Prevent rerenders with useShallow

When you need to subscribe to a computed state from a store, the recommended way is to use a selector.

The computed selector will cause a rererender if the output has changed according to Object.is.

In this case you might want to use useShallow to avoid a rerender if the computed value is always shallow equal the previous one.

and more...
https://docs.pmnd.rs/zustand/guides/prevent-rerenders-with-use-shallow

## SSR and Hydration
### SSR
Server-side Rendering (SSR) is a technique that helps us render our components into HTML strings on the server, send them directly to the browser, and finally "hydrate" the static markup into a fully interactive app on the client.

### Hyration
Hydration turns the initial HTML snapshot from the server into a fully interactive app that runs in the browser.

and more...
https://docs.pmnd.rs/zustand/guides/ssr-and-hydration

## Setup with Next.js

https://docs.pmnd.rs/zustand/guides/nextjs

# Integrations

## Immer middleware

https://docs.pmnd.rs/zustand/integrations/immer-middleware

## Third-party Libraries

https://docs.pmnd.rs/zustand/integrations/third-party-libraries

## Persisting store data

https://docs.pmnd.rs/zustand/integrations/persisting-store-data
