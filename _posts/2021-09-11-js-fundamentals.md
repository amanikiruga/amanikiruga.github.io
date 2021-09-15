---
title: JavaScript Fundamentals
category: CISC275
layout: post
permalink: /software/js_fundamentals
---
# {{page.title}}

Apparently one of the first steps in web development is learning how to program in JavaScript. This is a language with a lot of quirks. Though, it is currently according to [w3techs.com](https://w3techs.com/technologies/overview/client_side_language) by far the most popular client-side programming language as of yet. Programming in JS may also be one of the most common tasks (pains) of front-side developers today! So, it may be wise to learn the fundamentals. 


In order to get started quickly, I read the official MDN Mozilla docs on [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/) which gives an overview of the main concepts of the language (and how quirky it can get ). I summarize important points I picked up below. 



### Variables & Types
Coming from Java and Python, both strongly typed languages, the way JavaScript handles types is very interesing. First of all, it has no ``Integer`` type but instead has ``Number`` which is basically a floating point number. This is bad because of the [imprecision of floating point ](https://floating-point-gui.de/basic/). 

I also am somewhat confused of the ``undefined`` type, since it is apparently different from ``null`` but used in instances where ``null`` would be used in another language like Java such as uninitialzed  variables eg. ``String thisIsNull; ``. 

Lastly, I found it kind of cool that you could go from string to int using unary ``+`` in a seamless fashion: 
```js
+'7.6' // 7.6
```

### Functions
Functions are one of the most interesting topics to me since they seem very convenient especially with arrow functions. 

That is, we can write things like: 
```js
let map = (f, a) => {
  let result = [];
  let i;
  for (i = 0; i != a.length; i++) result[i] = f(a[i]);

  return result;
}
let nums = [1, 3, 5];
numsCubed = map((x) => {
  return x * x * x;
}, numbers);
// -> [1, 27, 125]
```
which take the form of ``(args, ...) => {statements}`` instead of ``nameOfFunc(args, ... ) {statements}``. 

I also found Promises to be very compelling! The main reasons one would use them instead of traditional call-back functions are: 
* Callbacks added with then() will never be invoked before the completion of the current run of the JavaScript event loop. In essense it feels as if you are writing synchronously preventing callback-hell!🔥
```js
//call-back hell
doSomething(function(result) {
  doSomethingElse(result, function(newResult) {
    doThirdThing(newResult, function(finalResult) {
      console.log('Got the final result: ' + finalResult);
    }, failureCallback);
  }, failureCallback);
}, failureCallback);
```
* They can be **chained** and sequenced. 

Example: 
```js
//or even shorter using arrow notation
doSomething()
.then(result => doSomethingElse(result))
.then(newResult => doThirdThing(newResult))
.then(finalResult => {
  console.log(`Got the final result: ${finalResult}`);
})
.catch(failureCallback);
//.catch (failureCallback) is short for .then(null, failureCallback)
```

### Objects
I also found objects very cool since I have used JSON before and this is now the defacto method of creating objects. Creating objects is really simple: 

```js
var myCar = {
    make: 'Ford',
    model: 'Mustang',
    year: 1969
};
```
and modifying or accessing properties is just as easy
```js
//modifying
myCar.make= 'Toyota'
//accessing notation
myObj.type              = 'Dot syntax';
myObj['date created']   = 'Bracket notation';
```

### Working with HTML
I also found it interesting how seamless using JS to manipulate the DOM (Document Object Model) is. I have yet much to learn in this re

### Summary
JavaScript seems like an incredibly useful (and risky) language to use for Web Development. Especially when paired with libraries like TypeScript, React, etc. I hope to get better at it with time! 
