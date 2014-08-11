---
layout: post
title: "JavaScript Event Systems Unraveled"
date: 2014-08-08 18:07:13 -0700
comments: true
categories: ['Events','JavaScript']
---
Events are an extremely useful aspect of many JavaScript frameworks, but their functionality can seem bizarre if you don't know what's going on under the hood. In this blog post I will explain the inner workings of an event system, in particular how it can function as part of a system of classes. Because this explanation involves some discussion of classes you should definitely read my blog post on pseudo-classical instantiation first if you are unclear about that subject.

Let's imagine a production system involving a manufacturing center and a shipping service. When the factory is finished making the products, shipping needs to be informed so that they can come and pick them up for distribution. We can start with a simple set of JavaScript objects to represent these units:
```javascript
var factory = {
	//notifies shipping when products are completed
}

var shipping = {
	//picks up products when they are completed
}
```
At this point we need to add a system to convey messages. Our messaging system will consist of a method 'on' which sets up a callback to be invoked in response to a given event, and a method 'trigger' which triggers an event, causing the callback functions to be invoked.
```javascript
shipping.ship = function(){
    console.log('Picked up!');
};
//using the 'on' we set up the shipping object to listen for the 'completed' event
factory.on('completed', shipping.ship);

//set our event in motion
factory.trigger('completed') //=> 'Picked up!'
```
To review, our event system is now made up of two methods:
1. object.on('event', callback) set callback to listen to 'event'
1. object.trigger('event') triggers 'event', invoking all the callbacks listening to it

When you have a set of objects that all share a basic set of methods it makes sense to set up a class that they will delegate to for those methods. Before I discuss this class in detail, take a look at the code:
```javascript
var Unit = function () {
  //each instance an object container for events and their callbacks
  this.events = {};

  //store the callbacks that listen for events
  this.on = function(e, cb) {
    this.events[e] = this.events[e] || [];
    this.events[e].push(cb);
  };

  this.trigger = function(e) {
    //when event 'e' is triggered iterate over the callbacks listening to that event and invoke them
    var callbacks = this.events[e];
    for(var i = 0, k = callbacks.length; i < k; i++) {
      callbacks[i]();
    }
  };
};

//instantiate our units as instances of the Unit class
var factory = new Unit();
var shipping = new Unit();
```
Our units now come prepackaged with the methods necessary for a basic event handling system. Our goal is to store functions that will be called when the object triggers a given event. To achieve this, we use the 'on' method, which takes the name of an event as its first parameter and the callback function as its second. This method finds the event name in the events object or sets a new key by that name and then pushes the callback into an array referenced by that key. Now when we call 'trigger' on an object, it uses the event name passed into trigger to find the array and iterate over the callbacks invoking each one. Lets examine this process more closely:
```javascript
var shipping = new Unit();
var factory = new Unit();

shipping.ship = function(){ console.log("Coming to pick up.") };

factory.events //=> {}

factory.on('completed', shipping.ship);

factory.events //=> { 'completed' : function() { console.log("Coming to pick up.") } }

factory.trigger('completed') //=> "Coming to pick up."
```
Looks good right? Our events system is now working how we wanted. You might, however, notice a problem here: the function referenced by 'shipping.ship' is stored in the events object of the factory without any reference to the shipping unit itself. We may want to refer to properties of the shipping unit but those properties are already long gone when the callback is invoked. Luckily, this problem can be fixed with an anonymous function:
```javascript
var shipping = new Unit();
var factory = new Unit();
//give shipping a 'name' property
shipping.name = "Everest Shipping";
//refer to the name property in the ship function
shipping.ship = function(){ console.log (this.name + " is coming to pick up.") }
//store the method invocation inside an anonymous function
factory.on('completed', function(){
  shipping.ship()
});
//inspecting the events object we see the shipping object is preserved inside the callback
factory.events //=> { 'completed': function(){ shipping.ship() } }
//now when the event is triggered, we have access to properties of the shipping object
factory.trigger('completed') //=> "Everest Shipping is coming to pick up."
```
Awesome! Now we have a reference to the shipping object inside the callback function. But we can make our system even better. What if we could pass data into our callback when we trigger an event? This requires just a small modification to our events system:
```javascript
var Unit = function () {
    //this.events and this.on() remain unchanged
    //we add a second parameter to our trigger method called 'data'
    this.trigger = function(e, data) {
      var callbacks = this.events[e];
      for(var i = 0, k = callbacks.length; i < k; i++) {
        //for each event in the callbacks array, we pass in data as an argument on invocation
        callbacks[i](data);
      }
  };
};
```
Now that our trigger function can take data as a second parameter and pass it into each callback, our events system effectively manages references to both the object that owns the callback and the object that triggers the event. If we assume that the above modification has been made to our Unit class, we can trigger events that have access to both sides of the event transaction:
```javascript
var factory = new Unit();
var shipping = new Unit();

shipping.name = "Everest Shipping";
factory.product = "smartphone";

//give the callback a 'product' parameter to recieve the data argument
shipping.ship = function(product){ console.log (this.name + " is coming to pick up " + product) };

//the callback registered to the event recieves the data and passes it into the ship method
factory.on('completed', function(product){
  shipping.ship(product)
});

//inspecting the events object reveals the function to be invoked when the event is triggered
factory.events //=> { 'completed': function(product){ shipping.ship(product) } }

factory.trigger('completed', factory.product) //=> "Everest Shipping is coming to pick up smartphone."
```
This concludes my discussion of the events system in JavaScript. For clarity, I will finish with an example demonstrating the use of multiple callbacks for a single event.
```javascript
var factory = new Unit();
var shipping = new Unit();
var retailer = new Unit();
var consumer = new Unit();

shipping.name = "Everest Shipping";
factory.product = "smartphone";

shipping.ship = function(product){ console.log (this.name + " is coming to pick up " + product) };
retailer.sell = function(product){ console.log ("Now selling the new " + product) };
consumer.use = function(product){ console.log ("This " + product + " is awesome!") };

factory.on('completed', function(product){
  shipping.ship(product)
});

factory.on('completed', function(product){
  retailer.sell(product)
});

factory.on('completed', function(product){
  consumer.use(product)
});

factory.trigger('completed', factory.product) //=>
// "Everest Shipping is coming to pick up smartphone"
// "Now selling the new smartphone"
// "This smartphone is awesome!"
```