---
layout: post
title: "What is a Compact Prefix Tree?"
date: 2014-08-02 14:05:48 -0700
comments: true
categories: 
---

There are a lot of data-structures out there and sometimes it can be hard to figure out what is worth learning or, even more important, what is useful for a given application. The compact prefix tree is a data-structure that I discovered recently and that I think it is interesting, fun to implement, and potentially very useful for a range of applications. In this post I will go over the basic implementation and uses of the compact prefix tree.<!--more-->

The compact prefix tree, or CPT, is basically a method for storing strings, most often individual words. The CPT economises by saving the difference between strings instead of the full string. The constructor function looks just like a normal tree:
```javascript
var Radix = function(value){
  value = value || '';
  this.value = value;
  this.children = [];
};
```
First we initialize the tree with no value. This will set our root as a base for saving a set of distinct branches, we'll see why we do this in a second. The special thing about the CPT is the insert method. Lets start by inserting the string 'to' into the tree:
```javascript
tree.value //=> ''
tree.children[0] //=> Tree {value: 'to', children: []}
```
Now our root has a single child with value 'to'. What if we insert 'tons' into our root?
```javascript
tree.insert('tons');
tree.children[0] //=> Tree {value: 'to', children: []}
tree.children[0].children[0] //=> Tree {value: 'ns', children: []}
```
Our CPT finds that the beginning of our input string is already stored in the tree and it stores the difference between this value and the input value in a child node of the node containing the "prefix" value. Ok so can you guess what happens if we insert "tonsils"?
```javascript
tree.insert('tonsils');
tree.children[0].children[0].children[0] //=> 'ils'
```
Because 'to' and 'ns' are already in the tree, our CPT simply stores the remaining value after these strings are removed in a new child node. Now we can call our reconstruct function which will reconstruct our input like this:

1. Starting at our root, no value, call reconstruct recursively on all children
1. At first child, return value + previous value, so we get ''+'to' = 'to'
2. At second child, return value + previous value 'to'+'ns' = 'tons'
3. And again, at the third child 'tons'+'ils' = 'tonsils'

I also gave my CPT a countChars function to count how many characters are actually stored in the tree, but here it is quite easy to see that we are storing just seven characters for a total input value of thirteen characters. This is the first advantage of the CPT, it compresses strings by eliminating the storage of redundant information. In any situation where a lot of words have to be stored economically, such as in a dictionary, the CPT is a highly economical choice. But there is another use to the CPT that hearkens back to the days of nokia flips phones.

What if we give a CPT an input string and tell it to return all the words in the tree starting with the first node containing a string that is longer than the input but has the input as a prefix. So, for example, if we give our tree the input 'ton', it will return every string longer than three characters that starts with 'ton'. And voila, we have auto-complete! Lets go over how that algorithm works step by step.

1. Make 'current string' by adding the current node's value onto the sum of previous values
2. Iterate over the child nodes of the current node
	a. If 'current string' plus this child's value matches part of the beginning of our target string,
		call auto-complete recursively on this child
	b. If our target string matches part of the beginning of 'current string' plus this child's value,
		call reconstruct on this child.

For each node, we check to see if the word represented by this node matches any amount of characters in our target starting at the beginning. If so, this node is a 'prefix' node of our target, and we have to go deeper down the tree along that branch. If, on the other hand, the word represented by this node is longer than our target and our target is a prefix of this word, we start reconstructing the tree with this node as the root.

To sum up, I'll show give you the code for this last function to look at, but I encourage you to check out my [github repository](https://github.com/incrediblesound/radix) to see the full code for a compact prefix-tree.
```javascript
Radix.prototype.complete = function(target, previous){
  var current;
  if(current === undefined){
    current = this.value;
  }
  if(previous){
    current = previous + this.value;   
  }
  for(var i = 0; i < this.children.length; i++){
      var node = this.children[i];
      var chk = new RegExp('^' + current + node.value + '\.*');
      var tst = new RegExp('^' + target + '\.*');
      if(chk.test(target)){
        return node.complete(target, current);
      }
      else if(tst.test(current + node.value)){
        return node.reconstruct(current);
      }
  }
  return null;
};
```