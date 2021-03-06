---
title: "Your Guide to React.useCallback()"
description: "React.useCallback() memoizes callback functions. In this post I will explain when to and when not to use React.useCallback()."
published: "2020-05-04T08:40Z"
modified: "2020-11-07T21:40Z"
thumbnail: "./images/cover-6.png"
slug: dont-overuse-react-usecallback
tags: ["react", "memoization"]
recommended: ["use-react-memo-wisely", "react-usestate-hook-guide"]
type: post
---

A reader of my blog reached me on Facebook with an interesting question. He said his teammates, no matter the situation, were wrapping every callback function inside `useCallback()`:

```jsx{4-6}
import React, { useCallback } from 'react';

function MyComponent() {
  const handleClick = useCallback(() => {
    // handle the click event
  }, []);

  return <MyChild onClick={handleClick} />;
}
```

*"Every callback function should be memoized to prevent useless re-rendering of child components that use the callback function"* is the reasoning of his teammates.   

This reasoning is far from the truth. Moreover, such usage of `useCallback()` makes the component slower.  

In this post, I'm going to explain how to use correctly `useCallback()`.

## 1. Understanding functions equality check

Before diving into `useCallback()` usage, let's distinguish the problem `useCallback()` solves &mdash; the functions equality check.    

Let's write a function named `factory()` that returns functions: 

```javascript{11-12}
function factory() {
  return (a, b) => a + b;
}

const sum1 = factory();
const sum2 = factory();

sum1(1, 2); // => 3
sum2(1, 2); // => 3

sum1 === sum2; // => false
sum1 === sum1; // => true
```

`sum1` and `sum2` are functions that sum two numbers. They've been created by the `factory()` function.  

Functions in JavaScript are first-class citizens, meaning that a function is a regular object. The function object can be returned by other functions (like `factory()` does), be compared, etc.: anything you can do with an object.   

The functions `sum1` and `sum2` share the same code source but they are different function objects. Comparing them `sum1 === sum2` evaluates to `false`.  

That's just how JavaScript objects works. An object (including a function object) [equals](/the-legend-of-javascript-equality-operator/#the-identity-operator) only to itself.  

## 2. The purpose of useCallback()

Different function objects sharing the same code are often created inside React components:

```jsx{5-7}
import React from 'react';

function MyComponent() {
  // handleClick is re-created on each render
  const handleClick = () => {
    console.log('Clicked!');
  };

  // ...
}
```

`handleClick` is a different function object on every rendering of `MyComponent`.  

Because inline functions are cheap, the re-creation of functions on each rendering is not a problem. *A few inline functions per component are acceptable.*  

But in some cases you need to maintain one function instance between renderings:

1. A functional component wrapped inside `React.memo()` accepts a  function object prop
2. When the function object is a dependency to other hooks, e.g. `useEffect(..., [callback])`  

That's when `useCallback(callbackFun, deps)` is helpful: given the same dependency values `deps`, the hook returns (aka memoizes) the function instance between renderings:

```jsx{5-7}
import React, { useCallback } from 'react';

function MyComponent() {
  // handleClick is the same function object
  const handleClick = useCallback(() => {
    console.log('Clicked!');
  }, []);

  // ...
}
```

`handleClick` variable has always the same callback function object between renderings of `MyComponent`. 

## 3. A good use case

Imagine you have a component that renders a big list of items:

```jsx
import React from 'react';
import useSearch from './fetch-items';

function MyBigList({ term, onItemClick }) {
  const items = useSearch(term);

  const map = item => <div onClick={onItemClick}>{item}</div>;

  return <div>{items.map(map)}</div>;
}

export default React.memo(MyBigList);
```

The list could be big, maybe hundreds of items. To prevent useless list re-renderings, you wrap it into `React.memo()`.  

The parent component of `MyBigList` provides a handler function to know when an item is clicked:  

```jsx
import React, { useCallback } from 'react';

export default function MyParent({ term }) {
  const onItemClick = useCallback(event => {
    console.log('You clicked ', event.currentTarget);
  }, [term]);

  return (
    <MyBigList
      term={term}
      onItemClick={onItemClick}
    />
  );
}
```

`onItemClick` callback is memoized by `useCallback()`. As long as `term` is the same, `useCallback()` returns the same function object.  

When `MyParent` component re-renders, `onItemClick` function object remains the same and doesn't break the memoization of `MyBigList`.  

That was a good use case of `useCallback()`.  

## 4. A bad use case

Let's look at another example:

```jsx{4-6}
import React, { useCallback } from 'react';

function MyComponent() {
  const handleClick = useCallback(() => {
    // handle the click event
  }, []);

  return <MyChild onClick={handleClick} />;
}

function MyChild ({ onClick }) {
  return <button onClick={onClick}>I am a child</button>;
}
```

Does it make sense to apply `useCallback()`? Most likely not.  

`useCallback()` hook is called every time `MyComponent` renders. Even `useCallback()` returning the same function object, still, the inline function is re-created on every re-rendering (`useCallback()` just skips it).   

This doesn't bring any benefits because *the optimization costs more than not having the optimization*.  

Don't forget about the increased code complexity. You have to keep the `deps` of `useCallback(..., deps)` in sync with what you're using inside the memoized callback.  

Simply *accept* that on each re-rendering new functions are created:

```jsx{4-6}
import React, { useCallback } from 'react';

function MyComponent() {
  const handleClick = () => {
    // handle the click event
  };

  return <MyChild onClick={handleClick} />;
}

function MyChild ({ onClick }) {
  return <button onClick={onClick}>I am a child</button>;
}
```

## 5. Summary

When thinking about performance tweaks, recall the [statement](https://wiki.c2.com/?ProfileBeforeOptimizing):

> Profile before optimizing

Any optimization adds complexity. Any optimization added too early is a risk because the optimized code may change many times.  

These considerations apply to `useCallback()` hook too. Its appropriate use case is to memoize the callback functions that are supplied to memoized *heavy* child components. 

Either way:

* profile
* quantify the increased performance (e.g. `150ms` vs `50ms` render speed increase)

Then ask yourself: does the increased performance, compared to increased complexity, worth using `useCallback()`?

To enable the memoization of the entire component output I recommend checking my post [Use React.memo() wisely](/use-react-memo-wisely/).  

*Do you know use cases that worth using `useCallback()`? Please share your experience in a comment below.*