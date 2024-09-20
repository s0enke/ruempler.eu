---
title: "Understanding the React Function as Children Pattern"
date: 2024-09-20 12:00:00
---

Let’s have a look at the **React Function as Children Pattern**.

## The simple component

Usually, JSX/TSX code looks like this:

```tsx
<SomeComponent>
  <SomeChild someProp={something}/>
</SomeComponent>
```

<!--more-->

## The inline JS/TS code

One complication is rending JSX/TSX inline conditionally:

```tsx
<SomeComponent>
{condition && <SomeChild someProp={something}/>}
</SomeComponent>
```

The code in curly braces is plain JS/TS, so it’s JS/TS wrapped in JSX/TSX, which is again wrapped in JS/TS. 🤯

## The anonymous function with magically appearing props

So far, so good. But some time ago I stumbled upon this syntax in Formik, a form library for React, which I found even more confusing at first glance:

```tsx
<Formik>
    {({ values, handleChange }) => (
        <form>
            <input type="text" value={values.someValue} onChange={handleChange} />
        </form>
    )}
</Formik>
```
What's that, a function call inside JSX/TSX? Where do `values` and `handleChange` come from so magically? 🤯🤯🤯

That's the so-called "function as children pattern": The `Formik` component takes a function as a child, which is called with some arguments. The function returns JSX/TSX.

## Unwrapping the magic

Let's unwrap this with a more simple example:

```tsx
type Props = {
  children: (someArg: string) => ReactNode;
}

const MyComponent = ({ children }: Props) => {
  return children('someValue');
}
```

The `MyComponent` component takes a function as a child, which is called with no arguments. The function returns JSX/TSX. So why would ony use a function instead of a plain JSX/TSX element? The answer is: The function can be called with arguments, which can be passed from the parent component. This is a powerful pattern to pass data and state from the parent to the child component:

```tsx
<MyComponent>
  {(someArg) => <div>{someArg}</div>}
</MyComponent>
```

This is shorthand syntax for:

```tsx
<MyComponent>
  {({ someArg }) => {
    return <div>{someArg}</div>
  }}
</MyComponent>
```

Or to make it even more clear, here the syntax with `function` style:

```tsx
<MyComponent>
  {function({ someArg }) {
    return <div>{someArg}</div>
  }}
</MyComponent>
```
Now it's even readable for the old school JS developers like me. 🤓

## Passing either plain JSX/TSX or a function

It's also possible to pass either a function or a JSX/TSX element to the `children` prop:

```tsx
type Props = {
  children: ((someArg: string) => ReactNode) | ReactNode;
}

const MyComponent = ({ children }: Props) => {
  if (typeof children === 'function') {
    return children('someValue');
  }
  return children;
}

<MyComponent>
  <div>Some static content</div>
</MyComponent>

<MyComponent>
  {(someArg) => <div>{someArg}</div>}
</MyComponent>
```

That's also [what Formik actually does](https://github.com/jaredpalmer/formik/blob/2618cc4e6af0b2b1fcbff93936b5d3c68809b791/packages/formik/src/types.tsx#L194-L196).