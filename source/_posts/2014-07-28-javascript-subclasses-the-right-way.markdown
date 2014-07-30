---
layout: post
title: "JavaScript Subclasses the Right Way"
date: 2014-07-28 12:00:04 -0700
comments: true
categories: [JavaScript, Pseudoclassical instantiation]
---
There is a lot of misinformation on the web about making classes and subclasses in JavaScript. Since the right way is both simple and logical, clearing the air about this issue will not be hard. The problem we face has three steps...<!--more-->

1. How do I make a object delegate to another object?
1. How do I make a class* in JavaScript?
1. How do I make a subclass that delegates to another class?

*javascript doesn't really have classes but there is a way to hack it

The first problem is rather simple, lets say we define an object called 'orator'.

```javascript
var orator = {
	oratory: 'And so it was that...',
	orate: function(){
		return this.oratory;
	}
}
```
Now that we have an orator object we can create an orator and call the methods we just defined:
```javascript
var me = orator;

me.orate() //=> 'And so it was that...'
```
Great, now I am an orator. But what if I want to be a professor? A professor is like an orator, except that there is a specific subject that she talks about. To do this we have to

1. Define a function that takes the 'subject' parameter as an argument
1. Within the function body, create an object that delegates to the orator
1. Modify the oratory so that it returns a professor specific oration
1. Return the object

Step two is the most important part of this process. In JavaScript, if you make a new object with Object.create() and pass an existing object into the create method like this...
```javascript
var newobject = Object.create(existingObject);
```
then the new object delegates to the existing object for all failed property and method lookups. That means that if we check our new object for any given property or method and the check <strong>fails</strong>, that is, the property or method does not exist on the new object, then the javascript interpreter will check the existing object for that property or methods and return it if the lookup on that object is successful. That means we can do this:
```javascript
var first = {
	first: 'I\'m number one!'
};
var second = Object.create(first);

console.log(second.first); //=> 'I\'m number one!'
```
Now that we can delegate properties and methods we can focus on making our (kindof) class. JavaScript has a very simple pattern for making a pseudoclass:

```javascript
var Class(prop1, prop1) {
	this.prop1 = prop1;
	this.prop2 = prop2;
}

Class.prototype.method = function(){};

var instance = new Class;
```
It is important to understand exactly what is going on here. When we use the 'new' keyword on line eight, the javascript interpreter adds two lines to our code before making our object.
```javascript
var Class(prop1, prop2) {
	this = Object.create(Class.prototype);
	this.prop1 = prop1;
	this.prop2 = prop2;
	return this;
}
```
It should now be fairly clear now how this object is constructed. But what about line six on the example before this one? It turns out that javascript gives us a built-in property specifically for storing <strong>methods</strong> for our class. Any method stored on the Class.prototype will be available on a new instance of the class because, as we see on line 2 of this example, the object returned by the constructor function is created with the prototype as its delegate. Our constructor function will not generate new functions every time an instance is created, and instead will delegate to the prototype of the class. Here is our orator constructor:
```javascript
var Orator = function(){
	this.oratory = 'And so it was that...',
	this.verbose = true;
}

Orator.prototype.orate = function(){
	return this.oratory;
}

```
To create a professor sub-class, we simply make a new professor class, define the properties that distinguish the professor from the orator and then set the professor prototype to a new object that delegates to the orator prototype. Now all failed calls to methods on a professor will delegate up the chain to the orator prototype.

```javascript
var Professor = function(subject){
	this.oratory = 'Today we will discuss '+subject+'.';
}

Professor.prototype = Object.create(Orator.prototype);
```
This will allow our professor to inherit the methods of the orator. If we also want to inherit the properties of the orator, we have to call its constructor function within the professor constructor:
```javascript
var Professor = function(subject){
	Orator.call(this);
	this.oratory = 'Today we will discuss '+subject+'.';
}

Professor.prototype = Object.create(Orator.prototype);
```
This will copy all the orator's properties into the professor. Notice that the oratory property will be copied and then overwritten by the professor constructor function during instantiation. And now we have a full subclass that inherits both properties and methods of the super-class.