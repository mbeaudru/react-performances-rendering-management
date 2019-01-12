# React performances: rendering management

## Goal

When you are choosing React as your primary library for your application, you probably will want to know how to write efficient components and hooks to provide a good experience to your users.

There are many ways to improve your apps performances (images compressions, memoization, code splitting to name a few) but here we will try to focus on how to write your _components_ so that you can take advantage of React and not the opposite.

You might have heard about `React.PureComponent` or `React.memo` which are indeed the tools you will want to use to improve some parts of your React app.

But wrongly used, you might end up having a hard to maintain / buggy codebase. We will see how and when to use those tools in a minute!

## When does a component re-renders ?

**By default**, a component will re-render if:

- One of the **props** the component receives have changed
- A component **state** variable has been updated
- It listens to a **context** variable that changes
- The update is forced by the developer using _this.forceUpdate_ (class syntax)
- Each time the **parent** that calls the component is being rendered

We will see that the first four points are quite straightforward, but hang on because the last one is the reason I'm writing this article!

### Practical example: <Button />

Let's figure out why Button re-renders when one of the above event happens:

> **Note :** I'm omitting the _forceUpdate_ and _context_ cases in this example to simplify the reasoning, but if you want to know more about them just read [the forceContext API docs](https://reactjs.org/docs/react-component.html#forceupdate) and [React Context docs](https://reactjs.org/docs/context.html).

_Button.js_

```javascript
import React from "react";

class Button extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      active: false
    };
  }

  toggleActiveState = () => {
    this.setState(prevState => ({ active: !prevState.active }));
  };

  render() {
    const { color } = this.props;
    const { active } = this.state;
    return (
      <button
        style={{ backgroundColor: color }}
        onClick={this.toggleActiveState}
      >
        {active ? "I am so actived right now!" : "Not very active today..."}
      </button>
    );
  }
}

export default Button;
```

- _props_ changes

Our _Button_ takes a color _prop_ which is used to determine the color of the button on screen. Whenever this property changes, we want the component to be re-rendered so that React reflects that change by updating the DOM for us.

- _state_ changes

Our Button has an inner state _active_ which is a boolean used to determine whether we should print _I am so actived right now!_ or _Not very active today..._ on screen. Again, whenever this property evolves React must update the DOM to reflect the changes, that's why our Button is re-rendered.

### Parent re-renderings makes all children re-render

At this point, you might be wondering:

> **You, wondering :** _If the props, the inner state and the context of the component has not changed, the component doesn't need to be re-rendered right ?_

Yup, totally ! **And yet**, even if your props and state haven't changed, React will make your components re-render each time the **parent** that calls your components is being rendered.

#### ButtonGroup example

To illustrate what a parent re-render means, let's use a <ButtonGroup /> that print our previous <Button /> multiple times. This component will have a state variable **counter** that will be displayed on screen.

```javascript
import React from "react";
import Button from "./Button";

class ButtonGroup extends React.Component {
  constructor(props) {
    super(props);

    this.state = {
      counter: 0
    };
  }

  incrementCounter = () => {
    this.setState(prevState => ({ counter: prevState.counter + 1 }));
  };

  render() {
    const { counter } = this.state;

    return (
      <div>
        <h1 onClick={this.incrementCounter}>Counter value: {counter}</h1>

        <Button color="red" />

        <Button color="blue" />

        <Button color="yellow" />
      </div>
    );
  }
}

export default ButtonGroup;
```

When you click on the `<h1> title`, the state variable _counter_ of <ButtonGroup /> will evolve and trigger a re-render of the component <ButtonGroup />. But not only ! All our `<Button />` components will be re-rendered as well, even if their props haven't evolved and their inner state remains the same.

Sounds absurd and inefficient ? Well, let's dig in to understand why they made that choice and how you should deal with that in your code!

## Props and state diffing

To say that a prop or state variable has **changed** is more complex that it seems depending on the kind of variable we are dealing with.

### Variables

#### Primitive variables

List of _primitive_ variables (ref: [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Data_types)):

- Boolean
- null
- undefined
- Number
- String
- Symbol (new in ECMAScript 6)

If we are dealing with a primitive variable, it's quite easy to check if there's been a change because you can compare their **values** directly:

```javascript
const foo = 1;
const bar = 1;

console.log(foo === bar); // true
```

#### "Object" variables

In JavaScript, object variables are for instance:

- Functions () => {}
- Array []
- Literal objects {}

> [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Objects): In computer science, an object is a value in memory which is possibly **referenced** by an identifier.

As stressed out, the key word in the sentence in _referenced_. Indeed, when you compare two "object" variables you are not comparing their _values_ but their in-memory _references_.

```javascript
const foo = {}; // creates an object with a reference X in memory
const bar = {}; // creates another object with another reference Y in memory

console.log(foo === bar); // false ! Despite having the same "values", the "===" operator only compares the in-memory references which are different (X and Y respectively)
```

### React perspective

That being said, let's get back into our initial issue: why does React decides to re-render all the children components when the parent is being re-rendered ?

**TL;DR :** There is no _generally_ applicable strategy for checking props / state changes, using *React.PureComponent* or *React.Memo* everywhere actually might harm performances and lead to bugs used without caution, while deep comparing objects is very costly and generally not a suitable option.

#### Object comparisons

When it comes to rendering optimizations, React gives you two solutions:

- **shouldComponentUpdate / React.Memo custom handler**

Using [_shouldComponentUpdate_](https://reactjs.org/docs/react-component.html#shouldcomponentupdate) lifecycle we can take the control and override the default React "re-rendering" behavior.

Every time React is tempted to re-render (because of one of the events above) it will run this method. The return value of _shouldComponentUpdate_ is a boolean, and React will re-render our component only if the return value is _true_.

- **Use _React.PureComponent_ (or _React.Memo_ for functional components)**

Implements a _shouldComponentUpdate_ for you using a [_shallow comparison algorithm_](https://reactjs.org/docs/react-api.html#reactpurecomponent).

##### Speed vs precision trade-off

As we've just seen, when we compare objects we in fact compare their references and not their values. Let's see what happens when you shallow compare props using a _PureComponent_

_ParentComponent.js_

```javascript
class ParentComponent extends React.Component {
  componentDidMount() {
    setTimeout(() => this.forceUpdate(), 500);
  }

  render() {
    const foo = {
      bar: "baz"
    };

    return <ChildComponent foo={foo} />;
  }
}
```

_ChildComponent.js_

```javascript
class ChildComponent extends React.Component {
  shouldComponentUpdate(nextProps) {
    // This is how PureComponent works, but it loops on your props in addition
    console.log(
      nextProps.foo, // { bar: "baz" }
      this.props.foo, // { bar: "baz" }
      nextProps.foo === this.props.foo // false ! Despite having the same values, the references are different
    );

    // Will return "true" despite we know the result in the DOM will be the same...
    return nextProps.foo !== this.props.foo;
  }

  render() {
    return <div>{this.props.foo.bar}</div>;
  }
}
```

In trying to improve the performances of our app in saving a render, we actually made it worse ! Indeed, we just made a useless _foo_ prop diffing that still triggered a re-render.

Besides, if the _ParentComponent_ was doing weird things with the object it is passing to ChildComponent (like make mutations to the object), a shallow comparison could even make the component [**not** to render while it should have](https://react-cn.github.io/react/docs/advanced-performance.html#shouldcomponentupdate-in-action), breaking your UI!

In conclusion, _shallow comparison algorithm_ is a nice tool to use where you are really needing it but it should not be generalized.

> **Note :** Unless you use a library like _ImmutableJS_ to manage all your Objects but that's another story !

So, can't we make a more precise check to solve this problem ? Well... yeah:

_ChildComponent.js_

```javascript
class ChildComponent extends React.Component {
  shouldComponentUpdate(nextProps) {
    console.log(
      nextProps.foo.bar, // "baz"
      this.props.foo.bar, // "baz"
      nextProps.foo.bar === this.props.foo.bar // true
    );

    // It returns false, that worked
    return nextProps.foo.bar !== this.props.foo.bar;
  }

  render() {
    return <div>{this.props.foo.bar}</div>;
  }
}
```

But this won't scale, because you need to dig in for each leaf of your objects until you reach a primary type you can value-compare.

Indeed, your objects will probably have more keys / values than this and they might be deeper too ! In order to work, every time your prop shape changes you would need to go back to this shouldComponentUpdate function and make sure you didn't forgot any keys. That's a huge source of bugs and maintenance issues.

There are [some libraries](https://github.com/FormidableLabs/react-fast-compare#do-i-need-shouldcomponentupdate) and techniques to deep compare two objects, but it comes at a cost that will totally defeat the initial purpose unless you use them very cautiously.

## Conclusions

Now that we've seen that performance optimizations couldn't be safely generalized and used with caution, here is my suggestion on how to manage react renderings in an application:

- First, write the most clear and maintainable code you can.
- If you have a doubt on whether the re-renders are costly or not, **measure it** using the [devTools](https://reactjs.org/docs/optimizing-performance.html#profiling-components-with-the-chrome-performance-tab) before trying to optimize your code.
- If you find out that some components are rendering too often and are costly, use `React.PureComponent` / `React.memo` or implement your own `shouldComponentUpdate` as a last resort.
