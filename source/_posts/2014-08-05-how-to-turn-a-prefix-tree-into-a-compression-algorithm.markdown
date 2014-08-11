---
layout: post
title: "Turning a Prefix Tree Into a Compression Algorithm"
date: 2014-08-05 08:46:05 -0700
comments: true
categories: ["Data","Structures","Algorithms","JavaScript","T9","Tree"]
---

My last post talked about the prefix-tree and went over some basic concepts and implementation. In this post I will explain how I turned a simple prefix-tree into an algorithm for efficiently storing long strings. If you want more info about the basic functionality of a prefix-tree take a look at my previous post or at the [github repo](https://github.com/incrediblesound/radix).<!--more-->

When I was experimenting with my prefix tree I realized that it only stored data efficiently if smaller strings were inserted before longer ones. This is because the tree checks to see if there are any nodes that already contain the beginning of the word being inserted, and if so, it chops off that chunk of the word and stores the remainder as a child of that node. So if we are storing "dreams", and the tree already has a node "dream", it will store "s" as a child of that node. This efficiency is lost if I already have "dreams" in the tree and I want to store "dream", it would store "dream" as a new node at the same level unless I did some clever re-balancing.

When I realized this point I had the idea of adding a function that takes an array of words, sorts them by length, and then inserts each word into the tree in order from shortest to longest. Then I realized that if I could take a string as input, split it into an array, sort the array, insert it into the tree, and then reconstruct the original input string, I would have a compression algorithm. With that idea in mind I came up with a plan for modifications to my original function:

1. Add a new insert function that accepts a string, turns it into a sorted array and inserts the contents of the array into the tree.
1. Remember the position of each word in the original string and store it in the tree
1. Adjust the reconstruct function so that it returns the original input string exactly.

Step one was easy: the new insert function, which I called documentInsert, simply adds a layer of processing on top of the original insert function. To accomplish step two I couldn't just insert the word itself, I had to bundle each word up with its position in the original string and insert that into the tree. Those two steps are both accomplished in the following function:
```javascript
Radix.prototype.documentInsert = function(string){

  //turn the input string into an array
  var splitString = string.split(' ');
  //iterate through the array and save each word in an object together with the 
  //original index of that word 
  var array = [];
  for(var k = 0; k < splitString.length; k ++) {
      array.push({ string: splitString[k], index: k });
  }

  var sort = function(left, right) {...};
  var merge = function(array){...};
  //this merge sort is modified to look into the objects stored in the array 
  //and sort them by string length, producing an array of objects 
  //storing words in order from shortest to longest.
  var sorted = merge(array);
  //iterate over the sorted array and insert each object into the tree
  for(var i = 0; i < sorted.length; i++){
    this.insert(sorted[i]);
  }
};
```
Storing the location of each word is simple: each node of the tree is initialized with an array called 'locations' that stores the index of the word in the pre-sorted input array. Words that are repeated in the original string will have different locations, so I modified the insert function such that when the inserted word is found to have been previously stored in the current tree I push the index of the inserted word onto the locations array of that node. This way each node stores a single word along with all the locations of that word in the original string.

The reconstruct function also has to be modified. I found the easiest way to do this was with a subroutine function defined within the reconstruct function. The reconstruct function holds a result array and a search function. It calls the search function on the root node which traverses the entire tree. At each node of the tree, the search function iterates over the locations array and sets the indexes of the results array corresponding to those locations to the word stored at that node. Before you look at the code, consider this simple example:
```javascript
var results = [];

var node = {
  value: 'a',
  locations: [ 2, 5, 7 ]
}

//if we set the indexes of results defined by the locations array 
//to the value of the node we get this:
results = [ undefined, 'a', undefined, undefined, 'a', undefined, 'a' ];
```
Because the locations correspond to the original locations of the words, it is not important in what order the node are visited, so long as the correct word for each node can be reconstructed. Now take a look at the code:

```javascript
Radix.prototype.reconstruct = function(value){

  var result = [];

  var search = function(context, value){
    if (value !== undefined){
      //each word is constructed by the current node value appended to the sum of the 
      //previous node values.
      value = value + context.value;
    } else {
      //at the root we set value to the empty string
      value = context.value;
    }
    //if value is not an empty string, iterate over the locations array and add the current
    //value to the result array at the indexes stored there
    if (value.length){
      for(var k = 0; k < context.locations.length; k++){
        result[context.locations[k]] = value;
      }
    }
    //if the current node has children iterate over them and call search recursively on each child
    if (context.children.length){
      var node;
      for(var i = 0; i < context.children.length; i++){
        node = context.children[i];
        search(node, value);
      }
    }
  }

  //call search on the root node and then return the result array after 
  //turning it back into a string with the join method

  search(this, value);
  return result.join(' ');
};
```
Now that all three steps are complete the new function can accept a string, store it in a compressed form, and reconstruct the original input. Compression rates will vary based on the amount of redundant information in the string. I have seen rates of between 2:1 and 3:1 in my experiments. I encourage readers to experiment with or improve on the function, which can be downloaded from npm:
    >npm install radix-compression