---
title: Intro to React Hooks
category: CISC275
layout: post
permalink: /software/react-hooks
---
# {{page.title}}
### Key Takeaways
I followed the official [react.js](https://reactjs.org/docs/hooks-overview.html) tutorial on hooks in order to learn more about how they work. I still have much to learn and will continue to update this post as I learn more! 

#### What are they? 
Hooks are functions that let you “hook into” React state and lifecycle features from function components.

#### useState
The following example increments a counter when you click on it. 
```js
import React, { useState } from 'react';

function Example() {
  // Declare a new state variable, which we'll call "count"
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
Here ``useState()`` is a hook. It returns current state value and a function that lets you modify state. Takes as argument initial value of the state value. Initial state only used during first render. 

```js
function ExampleWithManyStates() {
  const [fruit, setFruit] = useState('banana');
  const [todos, setTodos] = useState([{ text: 'Learn Hooks' }]);
  //using multiple times in a single component. 
}
```

#### useEffect
Side effects are when you, for example, perform data fetching, subscriptions etc. ``useEffect()`` is a hook that allows you to perform side effects from a function component

Note, since effects are declared inside the componenent, they have access to props and state. By default, React runs the effects after every render
eg. 
```js
import React, { useState, useEffect } from 'react';

function Example() {
  const [count, setCount] = useState(0);

  // Similar to componentDidMount and componentDidUpdate:
  useEffect(() => {
    // Update the document title using the browser API
    document.title = `You clicked ${count} times`;
  });
//this is a side effect and is done after React updates the DOM. 
  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```
Effects can be used to do something when a componenet mounts, or unmounts. 
