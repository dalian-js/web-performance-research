# Elements, children as props, and re-renders

## The Component

A component with many slow children components

```js
const App = () => {
  return (
    <div className="scrollable-block">
      <VerySlowComponent />
      <BunchOfStuff />
      <OtherStuffAlsoComplicated />
    </div>
  );
};
```

## With the scrollable feature

Re-rendering all slow children components

```js
const MainScrollableArea = () => {
  const [position, setPosition] = useState(300);
  const onScroll = (e) => {
    const calculated = getPosition(e.target.scrollTop);
    setPosition(calculated);
  };

  return (
    <div className="scrollable-block" onScroll={onScroll}>
      <MovingBlock position={position} />
      <VerySlowComponent />
      <BunchOfStuff />
      <OtherStuffAlsoComplicated />
    </div>
  );
};
```

## With the Element as an optimization

Because the slow components are not part of the same component that re-renders, they are outside the re-rendered component, that way they will be intacted, without the unnecessary re-render.

```js
const App = () => {
  const slowComponents = (
    <>
      <VerySlowComponent />
      <BunchOfStuff />
      <OtherStuffAlsoComplicated />
    </>
  );

  return <ScrollableWithMovingBlock content={slowComponents} />;
};
```

```js
const ScrollableWithMovingBlock = ({ content }) => {
  const [position, setPosition] = useState(300);
  const onScroll = (e) => {
    const calculated = getPosition(e.target.scrollTop);
    setPosition(calculated);
  };

  return (
    <div className="scrollable-block" onScroll={onScroll}>
      <MovingBlock position={position} />
      {content}
    </div>
  );
};
```

## Element as children

Instead of having a content prop, use the special children prop to pass children components

```js
const App = () => {
  return (
    <ScrollableWithMovingBlock>
      <VerySlowComponent />
      <BunchOfStuff />
      <OtherStuffAlsoComplicated />
    </ScrollableWithMovingBlock>
  );
};
```

```js
const ScrollableWithMovingBlock = ({ children }) => {
  const [position, setPosition] = useState(300);
  const onScroll = (e) => {
    const calculated = getPosition(e.target.scrollTop);
    setPosition(calculated);
  };

  return (
    <div className="scrollable-block" onScroll={onScroll}>
      <MovingBlock position={position} />
      {children}
    </div>
  );
};
```

### Key takeaways

- A Component is just a function that accepts an argument (props) and returns Elements that should be rendered when this Component renders on the screen. `const A = () => <B />` is a Component.
- An Element is an object that describes what needs to be rendered on the screen, with the type either a string for DOM elements or a reference to a Component for components. `const b = <B />` is an Element.
- Re-render is just React calling the Component's function.
- A component re-renders when its element object changes, as determined by Object.is comparison of it before and after re-render.
- When elements are passed as props to a component, and this component triggers a re-render through a state update, elements that are passed as props won't re-render.
- "children" are just props and behave like any other prop when they are passed via JSX nesting syntax:

## Configuration concerns with elements as props

```js
const App = () => {
  const [isDialogOpen, setIsDialogOpen] = useState(false);

  // when is this one going to be rendered?
  const footer = <Footer />;

  return isDialogOpen ? <ModalDialog footer={footer} /> : null;
};
```

- `footer` will be created as an element and stored in memory.
- This Footer will actually be rendered only when it ends up in the return object of one of the components, not sooner.

## Memoization with useMemo, useCallback and React.memo

Comparing values in JS

```js
const a = 1;
const b = 1;

a === b; // true
```

Comparing objects in JS:

```js
const a = { id: 1 };
const b = { id: 1 };

a === b; // false
```

When comparing objects, we are not comparing values, we are comparing references.

Learnings:

- Try to find the root cause of the re-render before adding `useCallback` or `useMemo`
- The inline function passed as an argument to either useMemo or useCallback will be re-created on every re-render. useCallback memoizes that function itself, useMemo memoizes the result of its execution.
  - Using memoization won't stop the re-render but just only stop recreating functions
- Memoize a prop only if passing a function reference as a prop (stable reference). Only makes sense when:
  - This component is wrapped in React.memo.
  - This component uses those props as dependencies in any of the hooks.
  - This component passes those props down to other components, and they have either of the situations from above.
- Memoizing all props on a component wrapped in React.memo is harder than it seems. Avoid passing non-primitive values that are coming from other props or hooks to it.
  - Avoid passing custom hooks functions as props to other components: any state change in the custom hook will make it re-render, update the reference and re-render the children components
  - When memoizing props, remember that "children" is also a non-primitive prop that needs to be memoized.
- React compares objects/arrays/functions by their reference, not their value. That comparison happens in hooks' dependencies and in props of components wrapped in `React.memo.`
- If a component is wrapped in React.memo and its re-render is triggered by its parent, then React will not re-render this component if its props haven't changed. In any other case, re-render will proceed as usual.

## Diffing and Reconciliation

- The Virtual DOM is just a giant object with all the components that are supposed to render, all their props, and their children - which are also objects of the same shape. Just a tree

```js
const Input = () => {
  return (
    <>
      <label htmlFor="input-id">{label}</label>
      <input type="text" id="input-id" />
    </>
  );
};
```

The virtual DOM object:

```js
[
  {
    type: "label",
    // other stuff
  },
  {
    type: "input",
    // other stuff
  },
];
```

- If the "type" is a string, it generates the HTML element of that type.
- If the "type" is a function (i.e., our component), it calls it and iterates over the tree that this function returned.

When re-rendering components:

- If "type" is changed, the component will mount the new one.
- If "type" is the same, the component will be re-rendered.
