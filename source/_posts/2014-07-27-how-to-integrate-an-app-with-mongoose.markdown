---
layout: post
title: "How to Integrate an App with Mongoose"
date: 2014-07-27 20:19:32 -0700
comments: true
categories: [Node.js, Mongoose, MongoDB, JavaScript] 
---

This is the second of two posts about my mongoose plugin [category-tree](https://www.npmjs.org/package/category-tree). In the last post we looked at an algorithm that finds a path to a leaf node through a tree of categories. With that step done, we need to apply the path to the correct properties of the document before it is saved to the database. This post explains how I integrated my app with mongoose so that it is called every time a document is saved.<!--more-->
	
If we look at the documentation for [mongoose middleware](http://mongoosejs.com/docs/middleware.html) we see that that middleware can be called before or after the init, validate, save and remove methods. Because my app sets properties of the document before it is saved, I use the pre-save format. Additionally, the middleware must be wrapped in a function that takes the schema as an argument and is exported for use in Node. The basic outline for this construction looks like this:

```javascript
exports.middleware = function(schema){
	schema.pre('save', function(next, data, cb){

	});
};
```
The data parameter holds whatever is passed to the save method on the document, and the callback is passed into next at the end of the function. The body of the function is simple:

1. Get the category labels that correspond to the document properties.
1. Run the makePath algorithm to get the path through the tree.
1. Store a reference to the document schema.
1. Use the category labels to iterate over the schema properties and set them to the categories determined by the makePath algorithm.
1. Invoke next with the callback parameter.

This function will interrupt the save method and set all the category properties of the document before continuing on. To use the middleware we have to add it to the schema: where your mongoose schemas are declared, pass the middleware function into the plugin method of the schema. Here is an example:

```javascript
var categorytree = require('category-tree').autoCat;

var Product = new Schema({
	name: String,
	description: String,
	price: String,
	subDepartment: String,
	category: String,
	subCategory: String
})

Product.plugin(categorytree);
```
Now we can use the app by passing the value for the most specific category, in this case 'subCategory', into the save method when saving a new item to the database:

```javascript
new product({
    name: 'security cam',
    description: 'hi fi',
    price: 'one million dollars'
  }).save('security', function(err, data) {
    //do stuff
  })
```
We do not need to set any of the category properties of this item because they are all set automatically. The above code saves the following document to the database:
```javascript
{ __v: 0,
  subCategory: 'security',
  category: 'electronics',
  department: 'household',
  name: 'security cam',
  description: 'hi fi',
  price: 'one million dollars',
  _id: 1234omg }
```