# JavaScript Notes

> **Source / Reference:** [Boot.dev — Learn JavaScript](https://www.boot.dev/courses/learn-javascript)
> These are my personal notes from the Boot.dev JavaScript course. All credit for the course material goes to Boot.dev.

Three core concepts of JavaScript:

- Types
- Scope
- Object-oriented programming

## Variables and Types

### Undefined vs. Undeclared

The primitive `undefined` value represents the absence of a value... but it's not the same thing as an "undeclared" variable!

```js
let favoriteSandersonCharacter; // undefined
console.log(typeof favoriteSandersonCharacter); // "undefined"
```

However, if we never create the name, it's "undeclared", and undeclared variables actually throw an error (so don't do this):

```js
console.log(favoriteRothfussCharacter);
// ReferenceError: favoriteRothfussCharacter is not defined
```

The worst part is that the undeclared variable error actually says "not defined".

> [!NOTE]
> As an aside, you can use `const` to declare a variable that is assigned to `undefined`, but you wouldn't be able to set its value later, so why would you want to?

```js
const favoriteSandersonCharacter; // SyntaxError: missing = in const declaration
const favoriteSandersonCharacter = undefined; // undefined
```

### Null vs. Undefined

One of JavaScript's most cursed features is that it has two values for "nothing":

- `undefined`: It doesn't exist at all. In grug-speak, `undefined` is "very nothing".
- `null`: It (kind of) exists, but it's empty. In grug-speak, `null` is "kinda nothing".

There are [some practical differences between the two](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/null#difference_between_null_and_undefined), the primary one being that `undefined` is the _default_ value of a variable when it hasn't been given a value yet.

```js
let myName;
console.log(myName); // undefined
```

To get a `null` value, you have to explicitly assign it:

```js
let myName = null;
console.log(myName); // null
```

In most cases, `null` and `undefined` work the same way, but you'll want to be consistent in how you use them, and know that there are subtle behavioral differences.

> [!TIP]
> I use `undefined` almost everywhere I would use `None` in Python or `nil` in Go. JavaScript is fairly unique in having two options. I only use `null` in cases where the behavioral difference matters, or I'm relying on external code that forces me to use `null`.

### Dynamic and Weak

Like Python, Ruby, and PHP, JavaScript is a dynamically-typed (not statically-typed) language. Its variable types are only known at _runtime_ (yes, yes, we'll talk about TypeScript in another course). **Some of us _like_ being able to see the types of our variables in our editors, okay**?

Now, _unlike Python_, it's also [weakly-typed](https://en.wikipedia.org/wiki/Strong_and_weak_typing), meaning it will automatically convert types when you do things like add a number to a string. This can lead to some unexpected behavior if you're not careful...

```js
let answerToLife = 42;
let answerToTheUniverse = "42";

// obviously JavaScript thinks that adding strings
// and numbers is totally sane and normal behavior
const answerToEverything = answerToLife + answerToTheUniverse;

console.log(answerToEverything);
// "4242"
```

## Functions

### Immediate Invocation

You can immediately invoke a function after defining it using the not-at-all-hard-to-pronounce acronym ["IIFE" (Immediately Invoked Function Expression)](https://developer.mozilla.org/en-US/docs/Glossary/IIFE).

```js
(function () {
  console.log("JavaScript: at least it's not Java");
})();
// JavaScript: at least it's not Java
```

They can also return values and take arguments:

```js
const result = (function (a, b) {
  return a + b;
})(1, 2);

console.log(result);
// 3
```

The function is defined and then immediately called. It looks nasty, but it's occasionally useful for a couple of reasons:

- **Scope**: It has its own scope.
- **Expression**: Can be convenient for computing a value as a single expression (like above).
- **Async**: Can be used to quickly run code in an `async` function.

## Objects

JavaScript [objects](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object) are an almost weirdly versatile collection type. Object literals (POJOs, or "plain old JavaScript objects") are often used to store data in key-value pairs.

```js
const apple = {
  name: "Apple",
  radius: 2,
  color: "red",
};
```

You can access properties stored on an object using the `.` operator:

```js
console.log(apple.name); // prints "Apple"
console.log(apple.radius); // prints "2"
console.log(apple.color); // prints "red"
```

JavaScript objects are interesting because you'll use them the way you'd probably use a map or a dictionary in other languages (simple key-value pairs), but they can also be used for more complex things like classes and prototypes... which we'll get into later.

### Optional Chaining

Nested data can _quickly_ become hard to work with. In most production systems you'll deal with 3-4 levels of object nesting on a regular basis.

When using the normal `.` operator, if the object on the left side of the `.` is `null` or `undefined`, you'll get a `TypeError` at runtime! Thankfully, JavaScript has recently added a new operator to make dealing with this headache easier, the [optional chaining operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining): `?.`

> [!NOTE]
> The **optional chaining (`?.`)** operator accesses an object's property or calls a function. If the object accessed or function called using this operator is [`undefined`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/undefined) or [`null`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/null), the expression short circuits and evaluates to [`undefined`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/undefined) instead of throwing an error.

```js
const tournament = {
  prize: {
    units: "dollars",
    value: 100,
  },
};

const h = tournament.referee.height;
// TypeError: Cannot read properties of undefined (reading 'height')
// tournament.referee resolves to undefined, so reading height from it
// results in a TypeError.
```

So, if you're _not sure_ whether the `referee` property exists (maybe it was sent to us over the network), we can use the optional chaining operator to avoid the error:

```js
const tournament = {
  prize: {
    units: "dollars",
    value: 100,
  },
};

const h = tournament.referee?.height;
// h is simply undefined, no error is thrown
```

**Example 2**

```js
const adventurer = {
  name: "Alice",
  cat: {
    name: "Dinah",
  },
};

const dogName = adventurer.dog?.name;
console.log(dogName);
// Expected output: undefined

console.log(adventurer.someNonExistentMethod?.());
// Expected output: undefined
```

### When to Chain

You should only use `?.` chains when you _expect_ an object may not exist. For example, if according to our business logic, a `user` _must_ have an `address` object, but the `address` object may not have a `street` property, we wouldn't use the optional chaining operator because we expect `user.address` to never be `undefined`.

```js
const street = user.address.street;
```

But if not all users have an address, we might use the optional chaining operator:

```js
const street = user.address?.street;
```

Or, if we aren't even sure if the `user` object exists:

```js
const street = user?.address?.street;
```

We don't want to overuse it because if we _expect_ that all users have objects, and we come across one that doesn't, we probably _want_ an error thrown so we can see it and go fix the problem. _Good errors make debugging easier._

### Object Methods

JavaScript objects can have `methods`, just like classes in Python or structs in Go. Objects are interesting in JavaScript because they play the role of dictionaries _and_ classes in other languages (yes, JS also has classes, but we'll get to them later).

Methods are functions that are defined on an object. They can access and change the properties of the object in question. In the context of an object method, the [`this`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this) keyword refers to the object itself, like `self` in Python (we'll talk about the admittedly hairy `this` keyword in more detail later).

```js
const person = {
  firstName: "Lane",
  lastName: "Wagner",
  getFullName() {
    return this.firstName + " " + this.lastName;
  },
};

console.log(person.getFullName());
// Lane Wagner
```

> [!NOTE]
> Methods can change the properties of their objects as well:
>
> ```js
> const tree = {
>   height: 256,
>   color: "green",
>   cut() {
>     this.height /= 2;
>   },
> };
>
> tree.cut();
> console.log(tree.height);
> // prints 128
>
> tree.cut();
> console.log(tree.height);
> // prints 64
> ```

### This

**Video: This and Arrow Functions**

![This and arrow functions](../media/this-and-arrow-functions-js.webp)

[Watch the full-quality video (MP4)](../media/this-and-arrow-functions-js.mp4)

The `this` keyword refers to the context where a piece of code is executed... let's cover some of those cases:

#### Global Context

`this` refers to the [`window`](https://developer.mozilla.org/en-US/docs/Web/API/Window) object in browsers or `module.exports` in Node.js (not `global`, as you might expect).

```js
// in a browser
console.log(this);
// Window { ... }
```

```js
// in Node.js
console.log(this);
// {}
```

#### Strict Mode

In [strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode) (which we'll cover later, don't worry about it too much for now) `this` is `undefined` in the global scope in both the browser and Node.js.

```js
"use strict";
console.log(typeof this);
// "undefined"
```

#### Method Context

Inside a standard [method](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Method_definitions) or a [constructor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new), `this` refers to the object the method is called on.

```js
const myObject = {
  message: "Hello, World!",
  myMethod() {
    console.log(this);
    console.log(this.message);
  },
};
myObject.myMethod();
// { message: 'Hello, World!', myMethod: [Function: myMethod] }
// Hello, World!
```

### Fat Arrows and This

One reason to choose an arrow function over a regular `function` or method is to preserve the `this` context. It's particularly useful when working with objects. To be fair, in a simple object like this, the non-arrow method makes perfect sense:

```js
const author = {
  firstName: "Lane",
  lastName: "Wagner",
  getName() {
    return `${this.firstName} ${this.lastName}`;
  },
};
console.log(author.getName());
// Prints: Lane Wagner
```

With a fat-arrow function, the `this` keyword **refers to the same context as its parent**. In essence, **fat arrow functions "preserve" the `this` context**. That's why `this` isn't `author` in this example. At the top level of a module, `this` is `undefined`, so the call throws:

```js
const author = {
  firstName: "Lane",
  lastName: "Wagner",
  getName: () => {
    return `${this.firstName} ${this.lastName}`;
  },
};
console.log(author.getName());
// TypeError: Cannot read properties of undefined (reading 'firstName')
```

Developers working in some front-end (yuck) JavaScript frameworks (like React or Vue) _tend to use fat arrow functions often_. The `this` context can contain a _ton_ of component-wide state, and it needs to be preserved throughout nested function calls, so fat arrows make the code easier to read and write.

### Spread Syntax

JavaScript has a really nifty [spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax#spread_in_object_literals) (`...`) for moving groups of object properties around. It's a great way to copy objects and merge object properties. The spread syntax [shallow-copies](https://developer.mozilla.org/en-US/docs/Glossary/Shallow_copy) the properties of the objects you're spreading. If properties have the same name, the last (right-most) object's property will overwrite the previous ones.

### Destructuring

It's admittedly annoying to have to get the return values from an object by using the `.` operator. The [destructuring assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment#object_destructuring) lets us unpack object properties into variables with the same names.

So, instead of this:

```js
const apple = {
  radius: 2,
  color: "red",
};

const radius = apple.radius;
const color = apple.color;
```

We can do this:

```js
const apple = {
  radius: 2,
  color: "red",
};

const { radius, color } = apple;
```

Destructuring also works in function parameters, which means that if you write a function that takes an object as an argument, you can unpack the object's properties in function definition.

So, instead of this:

```js
function eatApple(apple) {
  console.log(`ate a ${apple.color} apple with a radius of ${apple.radius}`);
}
```

We can do this:

```js
function eatApple({ radius, color }) {
  console.log(`ate a ${color} apple with a radius of ${radius}`);
}
```

### Not Bound

Methods in JavaScript are _not_ bound to their object by default (as they are in languages like Python and Go). So if you use a method as a "callback" function, you may run into issues with the `this` keyword:

```js
const user = {
  name: "Lane",
  sayHi() {
    console.log(`Hi, my name is ${this.name}`);
  },
};

user.sayHi();
// Hi, my name is Lane

const sayHi = user.sayHi;
sayHi();
// TypeError: Cannot read properties of undefined (reading 'name')
```

This happens a lot when passing a method as a [callback function](https://developer.mozilla.org/en-US/docs/Glossary/Callback_function) to another function:

```js
const user = {
  firstName: "Lane",
  lastName: "Wagner",
  getFullName() {
    return `${this.firstName} ${this.lastName}`;
  },
};

function getGreeting(introduction, nameCallback) {
  return `${introduction}, ${nameCallback()}`;
}

console.log(user.getFullName());
// Lane Wagner
console.log(getGreeting("Hello", user.getFullName));
// TypeError: Cannot read properties of undefined (reading 'firstName')
```

If you want to use a method as a callback function, you'll need to [`bind`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) it to the object using the `bind` method:

```js
const boundGetFullName = user.getFullName.bind(user);
console.log(getGreeting("Hello", boundGetFullName));
```
