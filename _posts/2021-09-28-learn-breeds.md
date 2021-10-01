---
title: Learn Breeds (Project)
category: CISC275
layout: post
permalink: /software/learn-breeds
---
# {{page.title}}
[demo](https://amanikiruga.github.io/learn-breeds/) | 
[github](https://github.com/amanikiruga/learn-breeds) | 

## Overview
I had a game in mind to learn different dog breeds. I decided to implement it using React and TypeScript. This is my first solo project using React.

The game is simple, all you have to do is guess which picture has the dog with the right breed. Only there is a 15 second timer, and if you get it wrong or the timer finishes, the game ends! It is pretty challenging! 

I used the following technologies: 
* Github project board and issues for project tracking
* Localstorage for database and storing of past scores
* React and TypeScript for backend and frontend. 
* API calls to retrieve random dog images by breed
* Asynchronous programming using ``useEffect()``and ``fetch()``
* ``setTimeout()`` in ``useEffect()``
* Flexbox for styling

## Things I learned
### Project Tracking
In order to track progress and chunk out todos, I decided to use **Github Project Board** as well as starting and resolving **issues**. In addition, I mostly worked on the features-bugs branch and merged to main only when I had completed a functionality. 
### Mockup Creating
Although not required, I implemented a small mockup on **Figma**, shown here: 
### MVVM
I decided to use an application programming model similar to Model-View-Viewmodel when creating the application. 
#### Repository
The respository was responsible for the calls to the local database (`localstorage`) and calls to the external call to the [dog.ceo](dog.ceo) API. 
#### View
The view was contained in the ``App`` component which was responsible for the showing of different UI screens as well as for passing props to children components. 

## Takeaways
This was definitely a challenging project but I thoroughly enjoyed leveraging these awesome technologies. I know I have barely scratched the surface on what you can possibly do with them. I am very glad to be in a class where I can try things out since on my own, I would have been too intimidated to. 

