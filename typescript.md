# Notes on TypeScript

## Reasons to Use

### Editor Support

- Because **everything has a type** in TypeScript, autosuggestions from your editor will generally be more complete, consistent and useful.

### Type Errors

A "type error" is when you an incorrect assumption is made about the shape of a particular piece of data. So for example:

```JavaScript
const foo = { bar: "value" };
console.log(foo.baz); // "Prints 'undefined'"
```

or worse

```JavaScript
const foo = {
  bar: () => {
    console.log("Log something!");
  },
};

console.log(foo.baz()); // TypeError: foo.baz is not a function
```

If you use TypeScript, you will spot these errors

- In your editor
- At build time

If you don't, you will spot them

- In your tests _if they are good enough_ (where they are usually harder to debug) or worse
- In log messages when an error has occurred **in production**

### Write Less Code

In JavaScript, it is good practise to code defensively and assume that any function you write will be misused, and so you really should be handling all possible inputs (e.g. what happens if I pass in `undefined` a `null`, a negative number, or an object that is the wrong shape). In practise, we all know that this doesn't always happen, but in TypeScript, _you don't have to_.

You tell the compiler that a specific function can **only take a string**, and so long as you follow certain best practises, this is actually **guaranteed**, which means you no longer need to write logic to handle other cases, or tests to validate the error handling.

### Readability

#### Duck Typing vs Explicit Typing

JavaScript uses what is known as [duck typing](https://en.wikipedia.org/wiki/Duck_typing) ("If it walks like a duck, and quacks like a duck, it must be a duck"). This is the general concept that you establish the type of something by its properties.

So for example

```JavaScript
const sum = (thing) => {
   return thing.reduce((accum, item) => accum + item);
}
```

We know that `thing` must be an array because for this function to work, `Array.prototype.reduce(...)` must be present. But to make that insight, you need to know all the functions available on a given type, and you need to read the function code (which might be much bigger than this). In TypeScript, you do

```TypeScript
const sum = (thing: string[]) => {
   return thing.reduce((accum, current) => accum + current);
}
```

Now you know right away that you are dealing with an array of strings, which means you can assume that all the appropriate functions will be present.

### Dynamically added properties

One of the reasons JavaScript types can be hard to understand in complex programs is because you can add properties on the fly as data is passed around the program. So for example:

```JavaScript
const foo = {
  bar: "value",
  baz: "value"
}
// 1231242 lines of code

foo.bap = "more values";
```

At this point, `foo` has now got **three** properties, but you have no way of knowing that from the original declaration.

In TypeScript, this kind of ad-hoc type construction is _not possible_. **Everything has a type at the point of definition** (although sometimes that type is _any_) and that type _cannot be changed_ without an explicit cast (which should be avoided unless absolutely necessary).

In the above snippet, if you ran it through the TypeScript compiler, where `foo` is defined TypeScript would _infer_ its type to be `{ bar: string, baz:string }`. Since this type doesn't _have_ a property called `bap`, the last line would produce an error: `Property 'bap' does not exist on type '{ bar: string, baz: string }`

So if you wanted to do this, how would you handle it? You'd define an explicit type like this:

```TypeScript
interface FooType {
	bar: string;
	baz: string;
	bap?: string;
}

const foo: FooType = {
  bar: "value",
  baz: "value"
}
// 1231242 lines of code

foo.bap = "more values"; // Yay! No error!
```

A really good example of the problem described in this section can be found in the presentation layer. Presentation Layer "middleware" modules are passed in a `context` object, which the middleware is expected to then mutate in order to compose the page. Finding out what properties are available to mutate is guesswork - you have to read other middleware, or read the code that renders the page based on this context object.

So you would define this object as having an **explicit type**. This would now mean your editor will autosuggest absolutely everything that is a available, and you could press F12 within VS Code to jump to that type in order to read it.

The TL;DR of this section is that TypeScript ensures that each data object has a **single source of truth to help a developer understand its shape**.

### Use in industry

In my experience on the job market this last few weeks, most employers I was speaking to make at least _some_ use of TypeScript, and some use it end to end. In the JavaScript domain it
is used more and more and getting to the point where you are all comfortable enough using TypeScript that it is not a barrier will make you more employable.
If you leave the BBC, its unlikely you are going to be able to avoid it.

## Best Practises

### Don't ever use `any`, use `unknown` instead

In TypeScript, all types are assignable to `any` and anything typed as `any` is assignable to anything else. The problem with this is that it **destroys the guarantees provided by the TypeScript compiler**, which means the assumptions I discussed above no longer hold. Consider this:

```TypeScript
const foo: any = 3;

const sum = (thing: string[]) => {
   return thing.reduce((accum, current) => accum + current);
}

sum(foo);
```

This compiles perfectly well, but when I execute it, I get `TypeError: thing.reduce is not a function`. Now, if this were JavaScript and we were coding defensively, we'd probably check that the right kind of thing was passed in, right? But this is TypeScript and we said that we don't need to do those kind of checks any more - and indeed you don't... unless you allow anything typed as `any` to be passed around your program. This is the code above without `any`

```TypeScript
const foo = 3; // TypeScript now infers the type to be 'number'

const sum = (thing: string[]) => {
   return thing.reduce((accum, current) => accum + current);
}

sum(foo); // Argument of type 'number' is not assignable to parameter of type 'string[]'
```

Now we are cooking with gas: The build/editor picked up an error that might have caused tests to fail or gone into production.

_What if you have data coming into your program that has an unknown shape? For example, from an API?"_

This is where `unknown` comes in. Similar to `any`, all types are assignable to something typed as `unknown`, but \_only something typed as `any` or `unknown` is assignable to something already typed as `unknown`. Why is this useful? It means it can be passed around, but **before you can do anything with it that might be incorrect, you first have to give the TypeScript compiler some way of establishing its type**

How do we do that? We use **user defined type guards**

Consider an API that might return data with two different shapes:

```json
{
   "type": "predatorResponse",
   "name": "shark",
   "teeth" 1231
}
```

```json
{
   "type": "preyResponse"
   "name": "fish"
   "scales": 123
}
```

If you pass that through `JSON.parse()`, you are going to get a return value that is typed as `any`. This should set off alarm values in your head - ideally if you get something from an api that you don't control typed as `any`, you want to give it a strict type as soon as you can:

```TypeScript
const data = await myClient.get(); // Assuming this is a nice easy client that just gives us a string blob
const theThing: unknown = JSON.parse(data);
```

So now we've typed it as `unknown`, we are safe from doing anything dangerous with it.

```TypeScript
console.log(theThing.scales) // Error! Property of type 'scales' does not exist on type 'unknown'
console.log(theThing.teeth) // Error! Property of type 'teeth' does not exist on type 'unknown'
```

Ok, so how do we turn this into something useful? How about this:

```TypeScript
interface Fish {
  type: "fish";
  name: string;
  scales: number;
}

const isFish = (thing: unknown): thing is Fish => {
  return (thing as Fish).type === "fish"; // Type-casts should be used sparingly, but this is one of the good use cases for them
};

if(isFish(theThing)) {
   console.log(theThing.scales) // No error!
} else {
	console.log(theThing.scales) // Error! Property of type 'scales' does not exist on type 'unknown'
}
```
