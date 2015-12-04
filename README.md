[![Build Status](https://travis-ci.org/canjs/can-wait.svg?branch=master)](https://travis-ci.org/canjs/can-wait)
[![npm version](https://badge.fury.io/js/can-wait.svg)](http://badge.fury.io/js/can-wait)

# can-wait

A library that tracks asynchronous activity and lets you know when it has completed. Useful when you need to call a function and wait for all async behavior to complete, such as when performing rendering.

## Install

```
npm install can-wait --save
```

## Usage

```js
import wait from "can-wait";
import { waitData } from "can-wait/waitfor";

wait(function(){

	setTimeout(function(){
		waitData("a");
	}, 29);

	setTimeout(function(){
		waitData("b");
	}, 13);

	var xhr = new XMLHttpRequest();
	xhr.open("GET", "http://chat.donejs.com/api/messages");
	xhr.onload = function(){
		waitData("c");
	};
	xhr.send();

}).then(
	function success(responses) {
		// responses -> ["b", "a", "c"]
	},
	function error(errors) {
		// errors -> [err, err, err]
	}
);
```

## Tasks

JavaScript uses various task queues (and a microtask queue) to run JavaScript in the event loop. See [this article](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/) and [this StackOverflow answer](http://stackoverflow.com/questions/25915634/difference-between-microtask-and-macrotask-within-an-event-loop-context) to learn more.

For can-wait to work we have to override various task-creating functionality, this is the list of what we currently implement:

**Macrotasks**

* setTimeout
* XMLHttpRequest

**Microtasks**

* requestAnimationFrame
* Promise
* process.nextTick

## API

In Node there are various async methods that do not fall into the most common macrotasks, but still might need to be tracked. To accomodate this we expose a global `canWait` function that can be used to wait on the outcome of an asynchronous request.

```js
var fs = require("fs");

fs.readFile("some/file", canWait(function(err, file){
	// We've waited
}));
```

### waitFor

**waitFor** is a function that creates a callback that can be used with any async functionality. Calling waitFor registers a wait with the currently running request and returns a function that, when called, will decrement the wait count.

This is useful if there is async functionality other than what [we implement](#tasks). You might be using a library that has C++ bindings and doesn't go through the normal JavaScript async APIs.

```js
import waitFor from "can-wait/waitfor";
import asyncThing from "some-module-that-does-secret-async-stuff";

asyncThing(waitFor(function(){
	// We waited on this!
}));
```

### waitData

You might want to get data back from can-wait, for example if you are using the library to track asynchronous rendering requests. Calling **waitData** with data or a Promise will register that data within the current request.

```js
import { waitFor, waitData } from "can-wait/waitfor";

// Note that waitData === waitFor.data === waitFor.waitData

let xhr = new XMLHttpRequest();
xhr.open("GET", "http://example.com");
xhr.onload = function(){
	waitData(xhr.responseText);
};
xhr.send();
```

Or with a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise):

```js
var promise = $.ajax({
	url: "http://example.com"
});

waitData(promise);
```

The data you register with **waitData** will be returned from the Promise as an array:

```js
import wait from "can-wait";

wait(function(){
	doSomeXHRThatCallsWaitData();
})
.then(function(responses){
	console.log(responses); // -> [{ foo: 'bar' }]
});
```

### waitError

Like **waitData** but pushes an error into the errors array for the current request. Most likely you can just throw an error to have it propagate, this is only useful in limited scenarios.

```js
import { waitError } from "can-wait/waitfor";
import wait from "can-wait";

wait(function(){

	setTimeout(function(){
		waitError(new Error("oh no"));
	}, 100);

}).then(null, function(errors){
	errors; // -> [{message: "oh no"}]
});
```

## License

MIT
