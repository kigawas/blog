+++
title = "Why state containers are necessary"
date = "2020-11-18"
cover = ""
tags = ["JavaScript", "Redux", "State container"]
description = "Understand and implement state containers like redux from scratch"
showFullContent = false
+++

This article is partially based on [a prior art](https://www.zhihu.com/question/63726609/answer/934233429).

## Overview

In modern JavaScript development, using some state container is a prevalent practice: `redux` for `react` and `vuex` for `vue`. But why it is necessary to integrate those "awesome state containers"? There might be too many reasons and explanations can be found all over the internet. In contrast, before stepping into the next level, we'll walk through the simplest scenario: states in module `A` have changed and module `B` needs to know and react.

### Archaic style

```js
class B {
  count = 0;
  updateCount(count) {
    this.count = count;
  }
}

class A {
  count = 0;
  b = new B();
  increase() {
    this.count += 1;
    this.b.updateCount(this.count);
  }
}
```

This solution is easy to understand, however it has a mortal defect: `A` needs to be aware of `B`'s interface. As complexity grows, the whole program would be a piece of spaghetti thus no one can or wants to maintain.

### Event bus

In order to resolve it, a design pattern called "event bus" appeared:

```js
// event bus (not complete yet)
export const bus = (() => {
  const listeners = {};
  return {
    dispatch(eventName, ...params) {
      const listener = listeners[eventName];
      if (listener !== undefined) {
        listener(...params);
      }
    },
    on(eventName, listener) {
      listeners[eventName] = listener;
    },
  };
})();

class A {
  count = 0;
  increase(n) {
    this.count += n;
    bus.dispatch("INCREASE_COUNT", n);
  }
}

class B {
  count = 0;
  constructor() {
    bus.on("INCREASE_COUNT", (n) => (this.count += n));
  }
}
```

Now `A` and `B` are decoupled. One aesthetic problem still remains: there is a redundant `count` update in `A`. Actually it can be moved into the closure, so let `A` subscribe it as well.

```js
class A {
  count = 0;
  constructor() {
    bus.on("INCREASE_COUNT", (n) => (this.count += n));
  }
  increase(n) {
    bus.dispatch("INCREASE_COUNT", n);
  }
}
```

Looks better, but there occurs another problem: `this.count += n` repeats. What we really wanted to do is to get the newest replica of `count`, unfortunately we wrote the logic to increase count (`this.count += n`) in every module.

### Event bus with state

To eliminate `count`, it's better to use a common state and update the state in the event bus:

```js
// event bus (with state)
const initialState = {
  count: 0,
};

export const bus = (() => {
  const state = initialState;
  const listeners = new Set();
  function updateState(eventName, ...params) {
    switch (eventName) {
      case "INCREASE_COUNT": {
        state.count += params[0];
        break;
      }
      default:
    }
  }
  return {
    getState() {
      return state;
    },
    dispatch(eventName, ...params) {
      updateState(eventName, ...params);
      for (const listener of listeners) {
        listener();
      }
    },
    subscribe(listener) {
      listeners.add(listener);
    },
  };
})();
```

This evolved version becomes pretty similar to redux. Finally `this.count` can be eliminated in module `A` and `B`:

```js
import { bus } from "./event-bus";

class A {
  constructor() {
    bus.subscribe(() => {
      const { count } = bus.getState();
      // do something with count...
    });
  }
  increase(n) {
    bus.dispatch("INCREASE_COUNT", n);
  }
}

class B {
  constructor() {
    bus.subscribe(() => {
      const { count } = bus.getState();
      // do something with count...
    });
  }
}
```

### Redux-like solution

In event bus version, everything is functional except `A` and `B`, and probably `A` and `B` are also unnecessary. If we borrow the idea from redux and rewrite:

```js
const createStore = (reducer) => {
  let state = reducer();
  const listeners = new Set();

  return {
    getState() {
      return state;
    },
    dispatch(action) {
      state = reducer(state, action);
      for (const listener of listeners) {
        listener();
      }
    },
    subscribe(listener) {
      listeners.add(listener);
    },
  };
};

const countStore = createStore((state = { count: 0 }, action = { type: "" }) => {
  switch (action.type) {
    case "INCREASE_COUNT":
      return { count: state.count + action.count };
    default:
      return state;
  }
});

countStore.subscribe(() => {
  // handler A
  const { count } = countStore.getState();
  console.log("A count:", count);
});

countStore.subscribe(() => {
  // handler B
  const { count } = countStore.getState();
  console.log("B count:", count);
});

const increase = (n) => {
  countStore.dispatch({ type: "INCREASE_COUNT", count: n });
};
```

Now we built our yet another state container library from scratch! If more powerful capabilities (like undo/redo, state persistence) are needed:

```js
const createStore = (reducer) => {
  const states = [reducer()];
  let sp = 0; // stack pointer
  const listeners = new Set();

  function notify() {
    for (const listener of listeners) {
      listener();
    }
  }
  function max(a, b) {
    return a > b ? a : b;
  }
  function min(a, b) {
    return a < b ? a : b;
  }

  return {
    getState() {
      return states[sp];
    },
    redo() {
      sp = min(sp + 1, states.length - 1);
      notify();
    },
    undo() {
      sp = max(sp - 1, 0);
      notify();
    },
    dispatch(action) {
      const newState = reducer(this.getState(), action);
      sp += 1;
      if (sp === states.length) {
        states.push(newState);
      } else {
        states[sp] = newState;
      }
      notify();
    },
    subscribe(listener) {
      listeners.add(listener);
    },
  };
};
```

What will happen if we `dispatch` in `reducer`? An exercise for you.

### Conclusion

Programming is about the art to control complexity. State containers not just separate the state and the logic but force you to think in an event-based and functional way, which is a ubiquitous pattern among many modern programming languages and frameworks.

The code above can be found at [repl.it](https://repl.it/talk/share/Why-state-container-blog-code/80447).
