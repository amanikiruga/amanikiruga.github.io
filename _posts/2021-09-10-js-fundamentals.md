---
title: JavaScript Fundamentals
category: CISC275
layout: post
permalink: /software/js_fundamentals
---
# {{page.title}}
Apparently one of the first steps in web development is learning how to program in JavaScript. This is a language with a lot of quirks. Though, it is currently according to [w3techs.com](https://w3techs.com/technologies/overview/client_side_language) by far the most popular client-side programming language as of yet. Programming in JS may also be one of the most common tasks (pains) of front-side developers today! So, it may be wise to learn the fundamentals. 

Majority of what I will be doing comes from the official MDN Mozilla docs on [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/) 

Let's get started!

Side note: If you're following along with me, I include ``''use strict''`` at the beginning of my code to make JS run in *strict mode*. This basically means that it throws an error when you for example set undeclared variables eg. `` x = 34 `` instead of ``let x = 34``. 

### Basics
In JavaScript, instructions are called statements and are usually separated by semicolons (;). 

For multiple statements on a line: 
```js
//incorrect
let x = 34 const y = 4
//correct
let x = 34; const y = 4;
```

### Variables
Three kinds of variable declaration:  
* ``var`` - globally or locally scoped. Unsafe, can get hard to debug. 
* ``let`` - declares a local block-scoped variable
* ``const`` - declares a read-only, local block-scoped variable. Safe and recommended

Variable without a value has ``undefined`` value. 

Accessing a variable undeclared throws a ``ReferenceError``. 

### Javascript Variable Types

1. ``Boolean``. true and false.
2. ``null``. Special keyword, different from ``undefined``. 
3. ``undefined``. Property stating a value is undefined. 
4. ``Number``. An Integer or Floating Point number. 
5. ``BigInt``. Integer with arbitrary precision. 
6. ``String``. Sequence of characters that make up a text. Like ``Hello World!``
7. ``Symbol`` (new in ECMAScript 2015). A data type whose instances are unique and immutable.

### Strings and Numbers
String concatenation
```js
x = "Hello " + 42; // "Hello 42"
//But if you use a minus, converts string to number
x = "42" - 7; //35
```
Proper way to convert string to number
``parseInt()`` or ``parseFloat()``
```js
v = parseInt('34') // 34
//Using optional radix parameter, base 2 for this example
x = parseInt('101', 2) //5
y = parseFloat('38.04') // 38.04
```
Another way to get a number is to add unary ``+`` .
```js
+'7.6' // 7.6
```

### Arrays
Literal using ``[]`` Following example creates an array ``coffees`` with ``length`` 3. 
```js
coffees = ['Arabica', 'Columbian', 'Kona' ]
coffees.length // 3
//weird notation
let x = ['Hi', , 'Everyone!']
x.length //3
x[0] // 'HI'
x[1] // undefined
x[2] // 'Everyone!'
Trailing commas, however, are ignnored. 
let funny = ['A', , 'Man', 'Walked', ,]
funny.length //5, last comma only is ignored
bad practice to use commas, better use undefined
 ```

 ### Numbers
 ```js

```
### Variable Scope

**Global variables** are created either by using the keyword ``var`` or creating an undeclared global variable like ``x = 3``. What is much more prefered today, however, is **local variables (block-scoped)** which are variables that cannot be accessed outside of a block. Loosely speaking, a block is the code inside 


### Function declaration
### Arrays
```js
//arr.find(predicate) method
//returns true for first match where predicate is true or undefined otherwise
arr = [1, 2, 3]
arr.find((e) => e >=2) //2
```



### Objects
  
* check if object has key
```js 
Object.hasOwnProperty.call(object, key)  //returns boolean
```
### Functions

* keyword function ``nameOfFunction(parameters, ...){statements;}``
* Primitive parameters (such as a number) are passed to functions by value;
* objects such as Array or any other, pass by reference

```js

//Anonymous function
const square = function (number) {
  return number * number;
};
var x = square(4); // x gets the value 16

//Used when passing a function as an argument to another function eg.
function map(f, a) {
  let result = [];
  let i;
  for (i = 0; i != a.length; i++) result[i] = f(a[i]);

  return result;
}

let numbers = [1, 3, 5];

cube = map(function (x) {
  return x * x * x;
}, numbers);
console.log(cube); // [1, 27, 125]

//Anonymous functions called instantly (without assigning to var)
(function() { 
  console.log("This will run!")
})(); //cool 

//based on condition
var myFunc;
if (num === 0) {
  myFunc = function (theObject) {
    theObject.make = "Toyota";
  };
}



```

### Variables
```js
property = 3 // this is bad practicee, creates a global property

numberStr = "3.0"
//parses to float
parseFloat(numberStr) //3.0

//parses to int
parseInt(numberStr) //3

d = 3.34701243710392847102348

d.toFixed(2) // 3.34
```

### Forms 

```js
<form action="/team_name_url/" method="post">
    <label for="team_name">Enter name: </label>
    <input id="team_name" type="text" name="name_field" value="Default name for team.">
    
    <input type="submit" value="OK">
</form>
```
* keywords
  * type - what widget is shown
  * name, id - identification
  * value - initial value of field
  * label - label of field in associated "for" 
  * for - contains id of associated input* 
  * action - url where data is sent for processing when form is submitted
  * method - http method to use post or get
    * get - should be used for forms that don't change like search bar
    * post - for anything that changes databases 
* Types
  * submit - button by default 

### Data and APIs
* Fetch

![fetch-overview]({{ "res/fetch-overview.png" | relative_url}} )
* Returns a promise
* images fetched are blobs
  * can't use with html have to turn to url using
    * ``URL.createObjectURL()``