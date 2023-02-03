# React Best Practices

The below details a guide of best practices on how to use and write React.

### `React.memo`

#### General Notes

* `React.memo` is a higher order component. When a component is wrapped with `React.memo`, React
  renders the component and memoizes the result. Before the next render, if the new props are
  the same, React will use the memoized result to skip the next rendering.
* `React.memo` only checks for prop changes. If your component uses `useState`, `useReducer`, or
  `useContext`, then the component will still rerender when state or context has been changed.
* React.memo will, by default, only provide a shallow comparison of complex objects in the props
  object. If you want more control over this comparison, you can provide your own custom function.
    * returning `true` tells `React.memo` to skip the next rendering whereas returning `false`
      tells React to rerender the component.
* If you are trying to memoize a component that accepts a function as a prop, you may need to
  use `useCallback` to make sure that React.memo recognizes this function as the same callback
  instance.

#### When should you use `React.memo`?

1. It is a pure functional component. Given the same props, the component always renders the same.
2. The component renders often.
3. The component re-renders with the same props.
4. The component is medium to Big size. It contains a decent amount of UI elements to reason a
   props equality check.

Examples:
* The component is being forced to re-render by a parent component.
* The application is regularly pulling apis in the background and updating views, such as an
  infinite loading table.

#### When to avoid using `React.memo`?

* The component isn't big or complex, or usually renders with different props.

`React.memo` is actually expensive, and can actually outweigh its performance gains.

If a component usually renders with different props, then the memoization of `React.memo` doesn't
provide any benifits, but instead can actually hinder performance because `React.memo` comes
with its own performance cost. This is because React does 2 jobs on every render:
1. It invokes the comparison function to determine whether the previous and next props are equal.
2. Because the props comparison almost always returns false, React performs the comparison
   calculation of the previous and current render.

Thus, wrapping such a component with `React.memo` will provide no performance gains and will
ultimately hinder performance because you will end up spend extra time running the comparison
function that comes along with `React.memo`.

#### `React.memo` and callback functions

Remember that function objects are only equal to itself:

```js
function sumFactory() {
  return (a, b) => a + b;
}
const sum1 = sumFactory();
const sum2 = sumFactory();
console.log(sum1 === sum2); // => false
console.log(sum1 === sum1); // => true
console.log(sum2 === sum2); // => true
```

Thus, when function callbacks are passed as props to memoized components, this breaks memoization
because every time a parent component defines a callback function for its child component, it
creates a new function instance:

```js
import React from 'react';

const Button = ({ onClick }) => <button onClick={onClick}>Clear cookies</button>;
  
const MemoizedButton = React.memo(Button);
  
const ParentComponent = ({ cookies, username }) => {
  // 👎 The parent component could provide different instances of the callback function on every
  // render, even with the same username value, thus breaking the memoization of `MemoizedButton`.
  return (
    <div>
      <h2>User: {username}</h2>
      <MemoizedButton onClick={() => cookies.clear('session')} />
    </div>
  );
};
```

In order to fix this, the callback function that is passed to the memoized component must
receive the same callback function instance. We achieve this by applying `useCallback` to preserve
the callback instance between renderings:

```js
import React, { useCallback } from 'react';

const Button = ({ onClick }) => <button onClick={onClick}>Clear cookies</button>;

const MemoizedButton = React.memo(Button);

const ParentComponent = ({ cookies, username }) => {
  // 👍 `useCallback` preserves the callback function instance between renderings and always 
  // returns the same function instance as long as cookies is the same, thus maintaining the 
  // memoization of `MemoizedButton`. 
  const handleClick = useCallback(
    () => cookies.clear('session'),
    [cookies]
  );

  return (
    <div>
      <h2>User: {username}</h2>
      <MemoizedButton onClick={() => cookies.clear('session')} />
    </div>
  );
};
```

#### Conclusion

* The more often the component renders with the same props, the heavier and the more
  computationally expensive the output is, and thus the more chances are that the component needs
  to be wrapped in `React.memo()`.
* Adding `React.memo` provides a performance boost by reusing the memoized content and thus skips
  rendering the component and not performing a virtual DOM difference check.
* `React.memo` can hinder performance if it is used on components that don't need it.
* Make sure to use `useCallback` for callback functions that are passed as props to memoized
  components.

#### Resources

* https://reactjs.org/docs/react-api.html#reactmemo
* https://dmitripavlutin.com/use-react-memo-wisely/

### `React.useCallback`

#### General Notes

* Functions in JavaScript are first-class citizens, meaning that a function is just a regular object that can be
  returned by other functions, be compared, etc. Thus, in JavaScript, since an object can only be equal to itself, the
  following is also true
```js
function sumFactory() {
  return (a, b) => a + b;
}
const sum1 = sumFactory();
const sum2 = sumFactory();
console.log(sum1 === sum2); // => false
console.log(sum1 === sum1); // => true
console.log(sum2 === sum2); // => true
```

* Normally, in react, every time a component re-renders, a function is recreated:
```js
const SampleComonent: React.FC = () => {
  // handleClick is re-created on every render
  const handleClick = () => {...};

  // ...
};
```

* However, sometimes you may want to maintain single function on every component rerender, instead of having a new
  function be created. This is what `useCallback` solves for.
* `useCallback(callbackFunction, deps)` is used that, given the same dependency values, the hook returns the same
  function instance on every render (the function instance is memoized).

#### When to use `useCallback`

`useCallback` should be used when it is important that you maintain the same function on every render:
* A functional component that is wrapped with `React.memo` uses a function as an equality check (see more notes
  regarding `React.memo` above).
* When the function object is a dependency to other hooks, e.g. `useEffect(..., [callback])`
* When the function has some internal state.

#### When to avoid using `useCallback`

Using useCallback increases code complexity and has its performance drawbacks, as it still has to run on every component
re-render. The optimization costs more than not having the optimization. Unless `useCallback` is needed for one of the
above purposes, you should not use it. You can simply accept that rendering creates new function objects.

#### Resources
* https://dmitripavlutin.com/dont-overuse-react-usecallback/
* https://kinsta.com/blog/react-usecallback/

### `React.useMemo`

#### General Notes

* `useMemo` accepts two arguments, a compute function, and a dependency array.
* During initial rendering, the compute function is invoked, memoizes the calculation result, and returns it to the
  component.
* `useMemo` does not invoke the compute function again until the dependencies change.
* Like other performance optimization techniques, `useMemo` can improve the performance of the component, but if used
  inappropriately, it could harm the performance of the component.
