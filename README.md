## useWorkerizedReducer

`useWorkerizedReducer` is like `useReducer`, but the reducer runs in a worker. This makes it possible to place long-running computations in the reducer without affecting the responsiveness of the app.

- Works with both [React] and [Preact].
- Weighs in at ~5KiB brotli’d.
- Powered by [ImmerJS].

### Example

```js
// worker.js
import { initWorkerizedReducer } from "use-workerized-reducer";

initWorkerizedReducer(
  "counter", // Name of the reducer
  (state, action) => {
    // Manipulate `state` directly. ImmerJS will take care of maintaining
    // referential equality.
    switch (action.type) {
      case "increment":
        state.counter += 1;
        break;
      case "decrement":
        state.counter -= 1;
        break;
      default:
        throw new Error();
    }
  }
);

// main.js
import { render, h, Fragment } from "preact";
import { useWorkerizedReducer } from "use-workerized-reducer/preact";

// Spin up the worker running the reducers
const worker = new Worker(new URL("./worker.js", import.meta.url), {
  type: "module",
});

function App() {
  // A worker can contain multiple reducers, each with a unique name.
  // `busy` is true if any action is still being processed.
  const [state, dispatch, busy] = useWorkerizedReducer(
    worker,
    "counter", // Reducer name
    { counter: 0 } // Initial state
  );

  return (
    <>
      Count: {state?.count ?? "?"}
      <button disabled={busy} onclick={() => dispatch({ type: "decrement" })}>
        -
      </button>
      <button disabled={busy} onclick={() => dispatch({ type: "increment" })}>
        +
      </button>
    </>
  );
}
```

### Browser Support

`useWorkerizedReducer` works in all browsers. Firefox requires a polyfill.

(Currently, `useWorkerizedReducer` relies on `WritableStream`, which is available everywhere except Firefox. If you want to support Firefox, I recommend the [web-streams-polyfill].)

### Details

`useWorkerizedReducer` takes care of bringing the functionality of `useReducer` to a worker. It bridges the gap between worker and main thread by duplicating the reducer’s state to the main thread. The reducer manipulates the state object in the worker, and through [ImmerJS] only patches will be `postMessage()`’d to keep the main thread’s copy of the state up-to-date.

Due to the communication with a worker, `useWorkerizedReducer` is inherently asynchronous. In fact, part of the motivation was to enable long-running reducers, which means considerable time can pass between a `dispatch()` call and the subsequent state change. `useWorkerizedReducer` will fully finish processing an action before starting the next one, even if the reducer is async.

If a reducer is still running, the `busy` variable returned by `useWorkerizedReducer` will be set to `true`.

### API

#### Exported methods

##### `useWorkerizedReducer(worker, name, initialState): [State | null, DispatchFunc, isBusy];`

The returned state will be `null` until the `initialState` has been transferred to the worker and applied. `isBusy` will also be `true` until that has happened.

##### `initWorkerizedReducer(name, reducerFunc);`

In contrast to the reducer functions from the vanilla `useReducer` hook, it is important to manipulate the `state` object directly. ImmerJS is recording the operations performend on the object to generate a patch set. Creating copies of the object will not yield the desired effect.

#### React

```js
import {
  useWorkerizedReducer,
  initWorkerizedReducer,
} from "use-workerized-reducer/react";
```

#### Preact

```js
import {
  useWorkerizedReducer,
  initWorkerizedReducer,
} from "use-workerized-reducer/preact";
```

#### Generic

```js
import {
  useWorkerizedReducer,
  initWorkerizedReducer,
} from "use-workerized-reducer";
```

---

Apache-2.0

[immerjs]: https://immerjs.github.io/immer/
[web-streams-polyfill]: https://www.npmjs.com/package/web-streams-polyfill
[react]: https://reactjs.org/
[preact]: https://preactjs.com/
