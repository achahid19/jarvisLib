# Building a React-like Library from Scratch ( jarvisLib )

![42 School](https://img.shields.io/badge/42%20Project-react-black?style=for-the-badge&logo=42)
![1337 School](https://img.shields.io/badge/1337%20Project-React-black?style=for-the-badge&logo=1337)
![JavaScript](https://img.shields.io/badge/JavaScript-ES6-yellow?style=for-the-badge&logo=javascript)
![TypeScript](https://img.shields.io/badge/TypeScript-4.0-blue?style=for-the-badge&logo=typescript)
![React](https://img.shields.io/badge/React-17.0.2-blue?style=for-the-badge&logo=react)
![jsx](https://img.shields.io/badge/JSX-React-blue?style=for-the-badge&logo=react)
![fibers](https://img.shields.io/badge/Fibers-React-blue?style=for-the-badge&logo=react)
![Hooks](https://img.shields.io/badge/Hooks-React-blue?style=for-the-badge&logo=react)
![Reconciliation](https://img.shields.io/badge/Reconciliation-React-blue?style=for-the-badge&logo=react)
![Rendering](https://img.shields.io/badge/Rendering-React-blue?style=for-the-badge&logo=react)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)

## Overview

This project is a step-by-step guide to building a lightweight, custom version of React from the ground up. It focuses on implementing the core architecture of React 17, including JSX, fibers, reconciliation, function components, and hooks. Non-essential features and optimizations have been omitted to keep the focus on the fundamental concepts.

This library was created to gain a deep understanding of how modern web frameworks operate and to serve as a dependency-free rendering engine for the 1337 / 42 school project, `ft_transcendence`, where external frameworks are not permitted.

## Core Features

* **JSX Support:** Using a custom `createElement` function.
* **Render Function:** A mechanism to render elements to the DOM.
* **Fibers:** A modern architecture for concurrent and non-blocking rendering.
* **Reconciliation:** An efficient diffing algorithm to update the UI.
* **Function Components:** Support for modern, functional components.
* **Hooks:** Implementation of the `useState` hook for managing component state.

## 📋 Table of Contents

* [How React Works: A Step Back](#️how-react-works-a-step-back)
* [Implementation](#implementation)
    * [1. `createElement` Function](#1-createelement-function)
    * [2. The `render` Function and Concurrency](#2-the-render-function-and-concurrency)
    * [3. Fibers: The Building Blocks of Concurrency](#3-fibers-the-building-blocks-of-concurrency)
    * [4. Render and Commit Phases](#4-render-and-commit-phases)
    * [5. Reconciliation: Efficiently Updating the UI](#5-reconciliation-efficiently-updating-the-ui)
    * [6. Function Components and Hooks](#6-function-components-and-hooks)
* [Author](#author)
* [License](#license)

***

## How React Works: A Step Back

Before we build our library, let's understand the basics of how React, JSX, and the DOM interact.

Consider this simple line of code:

```javascript
const element = <h1 title="foo">Hello World</h1>;
const container = document.getElementById("root");
ReactDOM.render(element, container);
```

There are two key things happening here:

1.  **JSX Transformation:** The `<h1>` tag looks like HTML, but it's actually JSX. Browsers don't understand JSX, so a tool like Babel or the TypeScript compiler transforms it into a standard JavaScript function call.
2.  **Rendering:** The `ReactDOM.render` function takes the result of that function call and renders it into the DOM.

### From JSX to JavaScript Object

The JSX `<h1>Hello World</h1>` gets transformed into a call to a `createElement` function:

```javascript
// This is what the compiler turns JSX into
const element = Jarvis.createElement(
  "h1",
  { title: "foo" },
  "Hello World"
);
```

Our `createElement` function then returns a simple JavaScript object. This object is a description of the HTML element, not the actual DOM element itself. This in-memory representation is often called a "Virtual DOM".

```javascript
// The object returned by createElement
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: [
      {
        type: "TEXT_ELEMENT",
        props: { nodeValue: "Hello World", children: [] }
      }
    ]
  }
};
```

Notice that the string "Hello World" is wrapped in its own object with the type `TEXT_ELEMENT`. Our library does this to treat all children uniformly, which simplifies the rendering logic.

## Implementation

### 1. `createElement` Function

Our `createElement` function takes a type, props, and an array of children. It constructs the element object, ensuring all children (even strings) are treated as elements.

```javascript
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.flat().map(child =>
        typeof child === "object" ? child : createTextElement(child)
      ),
    },
  };
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  };
}
```

> **Note:** To use JSX in TypeScript, your files must have a `.tsx` extension, and your `tsconfig.json` needs the following compiler options:
>
> ```json
> {
>   "compilerOptions": {
>     "jsx": "react",
>     "jsxFactory": "Jarvis.createElement"
>   }
> }
> ```

### 2. The `render` Function and Concurrency

A naive `render` function would recursively create each DOM node and append it. However, this has a major flaw: **it blocks the main thread**. If the element tree is large, the UI will freeze, unable to respond to user input or run animations until the entire tree is rendered.

**The Problem: A Blocking Render**

```
Main Thread: [ Start Render | ... | ... | ... | ... | End Render ]
User Input:   (waits)------------------------------------(processed)
Animation:    (stuck)-------------------------------------(resumes)
```

**The Solution: Concurrent Mode**
To solve this, we break the rendering work into small units. After each unit, we yield control back to the browser, allowing it to handle higher-priority tasks. We use `requestIdleCallback` to run our work loop whenever the main thread is idle.

```javascript
let nextUnitOfWork = null;

function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }
  requestIdleCallback(workLoop);
}

requestIdleCallback(workLoop);
```

This new approach requires a way to organize these "units of work". This brings us to fibers.

### 3. Fibers: The Building Blocks of Concurrency

A **fiber** is a JavaScript object that represents a unit of work. It corresponds to a component and contains information about its type, props, and its relationship to other fibers. Every element in the tree will have a corresponding fiber.

**Anatomy of a Fiber**

```
{
  type: 'h1',
  props: { ... },
  dom: null, // Will hold the actual DOM node
  parent: Fiber, // Link to parent fiber
  child: Fiber, // Link to first child fiber
  sibling: Fiber, // Link to next sibling fiber
  alternate: Fiber, // Link to the fiber from the previous commit
  effectTag: 'PLACEMENT', // Describes the work to be done (UPDATE, DELETION)
}
```

**The Fiber Tree Structure**
Instead of a recursive tree, fibers form a linked list. Each fiber has links to its `parent`, its first `child`, and its next `sibling`. This structure makes it easy to pause and resume work.

**Diagram: Fiber Traversal**
The renderer completes work on one fiber, then moves to the next following a specific order.

```
      [div (parent)]
          | (1. child)
      [h1 (child)]-- (2. sibling) -->[h2 (sibling)]
          | (3. child)                  ^
      [p (child)]-- (4. sibling) -->[a (sibling)]
                                        | (5. no child, no sibling)
                                        | (6. go to parent's sibling: h2)
```

When `performUnitOfWork` runs on a fiber, it does three things:

1.  Creates a DOM node for the fiber (but doesn't append it yet).
2.  Creates new fibers for its children.
3.  Returns the next fiber to work on based on the traversal logic.

### 4. Render and Commit Phases

To avoid showing an incomplete UI, we separate our work into two phases. This is a core concept of the fiber architecture.

**Diagram: The Two Phases**

```
+---------------------------------+      +--------------------------------+
|        Phase 1: Render          |      |        Phase 2: Commit         |
| (Can be paused and resumed)     |      | (Cannot be interrupted)        |
+---------------------------------+      +--------------------------------+
| - Build the "work-in-progress"  |      | - Take the finished tree       |
|   fiber tree in memory.         |      | - Apply all changes to the DOM |
| - Determine what needs to be    | ---> |   (add, update, delete nodes)  |
|   added, updated, or deleted.   |      | - Fast and synchronous         |
| - No changes to the actual DOM. |      |                                |
+---------------------------------+      +--------------------------------+
```

The `commitRoot` function orchestrates the commit phase. It processes deletions first, then walks the finished fiber tree and applies all other effects.

```javascript
function commitRoot() {
  // 1. Commit deletions from the previous tree
  deletions.forEach(commitWork);
  // 2. Commit the new or updated fibers
  commitWork(wipRoot.child);
  // 3. Save this tree as the "current" tree for the next render
  currentRoot = wipRoot;
  wipRoot = null;
}
```

### 5. Reconciliation: Efficiently Updating the UI

So far, we've only added to the DOM. To update or delete nodes, we need **reconciliation**. This is the process of comparing the new element tree with the last one we committed to the DOM (the "current" tree).

In the `reconcileChildren` function, as we iterate through the children, we compare each new element with its corresponding "old fiber" (which we can access via the `alternate` property).

**The Diffing Logic**

* **If the types are the same** (e.g., both are `h1`), we can reuse the existing DOM node. We create a new fiber, copy the DOM node over, and mark the fiber with an `UPDATE` effect tag.
* **If the types are different** and there's a new element, we create a new fiber and mark it for `PLACEMENT`.
* **If the types are different** and there's an old fiber that is no longer present, we mark the old fiber for `DELETION`.

**Diagram: Reconciliation in Action**

```
  Current Tree (in DOM)      New Tree (from render)      Resulting Effects
+-----------------------+   +----------------------+
|      div (fiber)      |   |     div (element)    |  ->  div fiber (UPDATE)
|          |            |   |          |           |
|      p (fiber)        |   |     h1 (element)   |  ->  p fiber (DELETION)
+-----------------------+   +----------------------+      h1 fiber (PLACEMENT)
```

During the commit phase, we use these `effectTag`s to perform the correct DOM operation: `appendChild`, `removeChild`, or updating properties.

> **Note on Keys:** Real React uses `key` props to optimize this process. Keys help React identify if a child has been re-ordered, moved, or removed, preventing it from unnecessarily re-creating DOM nodes. Our version simplifies this by only comparing types at the same position.

### 6. Function Components and Hooks

Our library also supports function components. The logic is slightly different:

* A fiber for a function component doesn't have a DOM node.
* Its `type` is the function itself.
* To get the children, we simply run the function (`fiber.type(fiber.props)`).

**Hooks (`useState`)**
To add state to function components, we implement `useState`.

* When `useState` is called, we look for a corresponding hook from the previous render (on the `alternate` fiber). If one exists, we use its state; otherwise, we use the initial state.
* The `setState` function it returns pushes an "action" (the new value or a function) to a queue on the hook object.
* When the component renders again, we process this queue of actions to compute the new, up-to-date state.
* Crucially, calling `setState` triggers a new render by creating a new `wipRoot` and setting `nextUnitOfWork`, kicking off the entire render/commit cycle again.

**Diagram: `useState` Hook Logic**

```
+---------------------------------+      +--------------------------------+
|     First Render (`useState(0)`)  |      |     Second Render (`setState(1)`) |
+---------------------------------+      +--------------------------------+
| - No alternate fiber exists.    |      | - Alternate fiber exists.      |
| - Create a new hook object:     |      | - Find old hook at hookIndex.  |
|   `{ state: 0, queue: [] }`     |      | - Create new hook object:      |
| - Return `[0, setState]`.       | ---> |   `{ state: 0, queue: [1] }`   |
|                                 |      | - Process queue: `state` -> `1`. |
|                                 |      | - Return `[1, setState]`.      |
+---------------------------------+      +--------------------------------+
```

This completes the core logic of our React-like library, providing a solid foundation for building simple, interactive applications.

## Author
**© Anas Chahid ksabi **@KsDev**** - [achahid19](https://github.com/achahid19)

## License
This project is academic work and is not intended for commercial use or redistribution.
