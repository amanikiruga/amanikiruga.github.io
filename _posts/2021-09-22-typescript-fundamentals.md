---
title: TypeScript Fundamentals
category: CISC275
layout: post
permalink: /software/typescript-fundamentals
---
# {{page.title}}

## Motivation
Although there are multiple types in JavaScript, Type rules are not enforced by default. TypeScript solves this by it's type system. I wanted to learn TS (as they usually call it) so I followed the official intro [tutorial](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html). 

## Details
### Types by Inference
TS can infer types based on usage in JS code. 
```js
let helloWorld = "Hello World"; // let helloWorld : string 
```

### Explicit Typing 
You can use interfaces on objects to declare a type. For example: 
```js
interface User {
  name: string;
  id: number;
}
```
then declare an object conforming to the interface. 
```js
const user : User{ 
    name: "Hello World", 
    id: 3
}
```
Can also be used together with classes: 
```js 
class UserAccount {
  name: string;
  id: number;
 
  constructor(name: string, id: number) {
    this.name = name;
    this.id = id;
  }
}
 
const user: User = new UserAccount("Murphy", 1);
```
or functions: 
```js
function getAdminUser(): User {
  //...
}
```

### Type Composition
#### Unions
Could be one of many types. 
```js
type myBool = true | false 
type positiveOddNumbers = 1 | 3 | 5 | 7 | 9; 
type lockStates = "locked" | "unlocked";  
```
or even when defining a function. 
```js
function getLength(obj: string | string[]) { // obj can either be string or array of strings. 
  return obj.length;
}

```
### typeof
Command to learn type of objects. eg. 
``typeof b === "boolean"`` or ``typeof f === "function"``. 
can be used for conditioning return types. 
```js
function wrapInArray(obj: string | string[]) {
  if (typeof obj === "string") {
    return [obj];
            
(parameter) obj: string
  }
  return obj;
}
```

### Generics 
Provide variables to types. eg: 
```js 
interface Backpack<Type> {
  add: (obj: Type) => void;
  get: () => Type;
}
 
// This line is a shortcut to tell TypeScript there is a
// constant called `backpack`, and to not worry about where it came from.
declare const backpack: Backpack<string>; // we didn't have to explicitly specify backpack took in string, alternative to specifying any type. 
 
// object is a string, because we declared it above as the variable part of Backpack.
const object = backpack.get();

//will fail 
const num = backpack.add(3) //but should be of type Type which in this case is a string. 

```

### Structural Type System 
TS can infer what you mean based on the *shape* of the object, even if you don't explicitly declare the object to follow a certain interface.
```js
interface Point {
  x: number;
  y: number;
}
 
 //here logPoint requires argument to of type Point
function logPoint(p: Point) {
  console.log(`${p.x}, ${p.y}`);
}
 
// logs "12, 26"
// here we never specify expliclty that point is of type Point. TS infered that through checking the shape matching. 
const point = { x: 12, y: 26 };
logPoint(point);
```
More examples: 
```js
//only requires a subset of the object fields to match
const point3 = { x: 12, y: 26, z: 89 };
logPoint(point3); // logs "12, 26"
 
const rect = { x: 33, y: 3, width: 30, height: 80 };
logPoint(rect); // logs "33, 3"
 // has no match, will fail. 
const color = { hex: "#187ABF" };
logPoint(color);
```
