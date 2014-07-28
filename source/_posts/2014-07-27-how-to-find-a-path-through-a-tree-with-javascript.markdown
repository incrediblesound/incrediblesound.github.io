---
layout: post
title: "How to Find a Path Through a Tree with JavaScript"
date: 2014-07-27 19:17:52 -0700
comments: true
categories: [ JavaScript, Tree, Algorithm ]
---

This is the first of two blog posts about my mongoose plugin [category-tree](https://www.npmjs.org/package/category-tree). The plugin is designed to automate save a series of hierarchical catagories by passing the most specific category to the save callback when saving an item to the database. <!--more--> In order to accomplish this task, the app needs to have a taxonomic tree containing all the category names in the system, and a set of names for each of the categories. A code example will serve to illustrate what I mean:
```javascript
var productTree = { 
	products:{    
    	household:{ appliance:{}, electronics: {     
        	sound_video:{ speakers:{}, tvs:{} },    
        	security:{}     
    	}     
 	},    
		office:{ 
			computers:{}, 
			desk:{ pens:{}, stationary:{} } 
		}
	} 
};    
var categories = ['department','subDepartment','category','subCategory']
```
Each property of the 'productTree' object is a potential label for one of the categories, and the categories correspond exactly to properties of a mongoose schema. To set the categories properly you need to be able to find a path through the productTree, and that problem will serve as the subject of this post. My algorithm for finding a path through a tree is described below. The following assumes we are keeping track of an array 'path', a boolean 'done' and a 'result' variable.

1. Iterate over the child nodes of the current node.
1. If done is false, check to see if the current child is our target node.
1. If the current child is our target push it to the path array, set result to path and done to true.
1. If the current child is not our target, examine the current child for children
1. If the child has children, set the child to current node and start over at step one. Otherwise do nothing.
1. If the iteration of the current level finished and done is still false, pop the last node out of the path aray.

The above algorithm tries every path through the tree, keeping a record of each path as it progresses. If a given path ends with a negative result, the path leading to that result will be popped off the record. Here is the code in JavaScript:

```javascript
function makePath(tree, target) {
    
  var result,
      done = false,
      path = {};

  function traverse(tree, target, root) {
    var keys = Object.keys(tree);
    forEach(keys, function(key) {
      if (!done) {
        if (key === target) {
          //if we found our target push it to the path
          path[root].push(target);
          //set result to the completed path
          result = path[root];
          //set done to true to exit the search
          done = true; 
          return;
        } else {
          //if the node does not match we need to check for children
          var newRoot = tree[key];
          if(Object.keys(newRoot).length > 0) {
            //if node has children, push the key into our path and check the children for our target
            path[root].push(key);
            return traverse(tree[key], target, root);  
          }
          //no children means our search of this branch is over
          return;
        }
      } 
    });
    //if we leave our for loop but we are not done that means we failed to find our target
    //in this branch, as a result we need to pop each node out of our path before we return
    if (!done){ 
      path[root].pop(); 
    }
    return;
  };

  //set an array of the root nodes of our product tree. These are super-categories that are
  //not saved in the item schema, possibly representing types of items, i.e. different schemas.
  var roots = Object.keys(tree);
  forEach(roots, function (root) {
    path[root] = [];
    //traverse our tree, going through each root node until the target leaf is found in the
    //tree defined by that root node.
    traverse(tree[root], target, root);
  });

  return result;
};
```
When a new item is saved to the database none of the categories need to be filled out, simply passing the most specific category as the first argument to the save function is sufficient. How this is accomplished is covered in the next blog post, which covers the integration of an app with the Node.js mongoose module.