# codegrid
Implement messages and asynchronous programming in Node.js and browser Javascript.
An alternative to callbacks and promises, easy to read, easy to expand.
##### Status Under construction!
Do not use in production, it will not work yet.
Please contact the author if you have any suggestions or if you feel that you can contribute.

## Table of Contents

*	[Introduction](#introduction)

*   [Definitions](#definitions)
	* channel
	* data message
	* channel state
	* control message
	* handler
	* scope


*   [Examples](#examples)
	* Include _codegrid_ in a Node.js module
	* Recursively read folder contents
	* Perform a simple callback without arguments
	* Two functions that call the same callback function
	* Two functions that call the same callback function
	* Callback passing a plain string
	* Callback passing a complex object
	* Replacing a callback hell
	* Serial execution #1 (collect and process data, then output result)
	* Serial execution #2 (collect data, then process data, then output result)
	* Parallel execution
	* Recursively read folder contents
*   [API](#api)
	* Object returned by require('codegrid')
		* .channel([options])
		* .message(channel[,message])
		* .messages(channel,messages)
		* .pause([channel[,count]])
		* .resume([channel])
	* Structure object required in channel()
		* [type]
		* [structure]
		* [optional]
		* [min]
		* [max]
	* Channel object returned by channel()
		* .message([,message])
		* .messages([,messages])
		* .pause([count])
		* .resume() 
		* .onConsume(function) 
		* .onError(function) 
		* .onPause(function) 
		* .onResume(function) 



## Introduction

**codegrid** is a lightweight framework designed to simplify Javascript asynchronous programming, both in web browser and server (node.js). It is very easy to learn and very easy to build large scale applications with. It can be used to synchronize data between code in the same module, in different modules, between a server and web browser or between two servers.

While promises implement asynchronous programming by giving emphasis on code, _codegrid_ attempts a different approach by moving the weight on data. This gives _codegrid_ an advantage when building large scale applications, because the art of programming is more mature for creating large quantities of data than creating large quantities of code. Increasing the code can lead quickly to applications that are very difficult to maintain. On the other side, increasing the data is a simpler process, as long as we follow two important principles: Define well and repeat often our data structures.

Also, task automation becomes easier when information is kept in data rather than in code.

To understand how codegrid works, imagine that an application can be drawn in a two-dimensional diagram:
- The horizontal axis of the diagram is the code flow, while the vertical axis is the data flow. 
- A sequence of code instructions is drawn as a horizontal narrow rectangle: Code inside the sequence executes from left to right.
- A queue of data objects is drawn as a vertical narrow rectangle: Data items are received on the queue top and they are consumed sequentially on the queue bottom.
- Arrows from codes to queues indicate the points where functions send data to the queues.
- Arrows from queues to codes indicate the event handlers that fire on queue events.

## Definitions
- __channel__  A data buffer with event handlers.
A channel receives data messages, stores them in a FIFO buffer and then fires an event handler to consume them.
- __data message__ 
A data object with a specific structure. 
Messages are sent from code to channels, where they are consumed.
All data messages that arrive to a channel must have the same structure.
- __channel state__
A channel can be in one of the following operation states: 
	- __normal state__ Data messages are consumed as soon as possible.
	- __paused state__ Data messages are stored to be consumed later.
- __control message__
Channels switch their state when they receive one of the following control messages:
	- __pause__  Channel enters the paused state. Consuming is stopped.
	- __resume__ Channel returns to the normal state. All pending data messages are consumed.
	- __flush__ Channel returns to the normal state. All pending data messages are flushed without consuming.
- __handler__ 
A function that fires when a specific condition occurs in a channel:
	- __message handler__ Fires to consume a data message and remove it from the queue
	- __error handler__ Fires when an error occurs inside a handler
	- __pause handler__ Fires when a pause control message is received
	- __resume handler__ Fires when a resume control message is received
- __scope__  
A special data container in channel for storing local variables, accessible from handlers.
A message handler may process the incoming messages and keep some data here.

## Examples


##### Include _codegrid_ in a Node.js module
```js
var cg = require('codegrid');
```

##### Perform a simple callback without arguments
```js
// A executes first, then B executes

function A() {
	// ... 
    cg.message(channel);	// Send message to channel when A is finished
}

var channel = cg.channel({	// Create an anonymous channel
	onConsume: function B(){ // Fired to consume a received message
    	// ... 
    }
});
```

##### Two functions that call the same callback function
```js
// Callback B executes after either execution of A1 or A2

function A1() { 
	// ... 
	cg.message('channel1');
}

function A2() { 
	// ... 
	cg.message('channel1');
}

cg.channel({	// Create a named channel
	name: 'channel1',
	onConsume: function B(){
    	// ... 
    }
});
```

##### Callback passing a plain string
```js
// A executes first, then passes a string to B, then B executes

function A() {
	// ... 
    cg.message(channel,'done');
}

var channel = cg.channel({	// Create an anonymous channel
	structure: 'string',
	onConsume: function B(s) {
    	console.log(s);
    }
});
```

##### Callback passing a complex object
```js
// A executes first, then sends an object to channel, 
// then channel validates the object, then B consumes the object

function A() {
	// ... Read a person from database to ...
    person.source = 'database'; // Add some more info to person
    cg.message(persons,person);	// Send person to B
}
var persons = cg.channel({	// Create an anonymous channel for consuming persons
	structure: {	// Define properties of data message:
    	id: 'integer',	// Person ID
        name: {type:'string',min:2,max:50},	// Person name (2...50 characters)
        age: {type:'integer',min:12,optional:true}, // Optional person age (12...)
        source: 'string',	// Information about where the person came from
        address: {	// Person address 
        	type: 'object',			// address is an object...
            structure: {			//  ...with the following properties:
	        	street: 'string',
	            streetNo: 'string',
	            zip: 'string',
	            city: {type:'string', optional:true},
             },
        },
        activities: { // An optional array with activities info
        	type:'array',
			structure: {
        		activityId: 'integer',	// Activity ID
                startDate: 'date',		// Activity start date
                endDate: {type:'date',optional:true}, // Activity end date
                subActivities:{	// Optional sub-activities
                	type: 'array',
                    optional: true,
                    structure: 'integer',
                	},
                },
        	max:10,
            optional:true
        },
    },
	onConsume: function B(person){ // Called when a valid message is received
    	// Person is guaranteed to have the defined structure
    	console.log(person.name,person.address.street,person.address.streetNo);
    },
    onError: function (error) {	// Called if validation fails or if an exception occurs in message handler
    	console.log(error);
    }
});
```

##### Replacing a callback hell
```js
// A calls B, then B calls C, then C calls D

function A() {
	// ...
	cg.message('->B');
}

function B() {
	// ...
	cg.message('->C');
}

function C() {
	// ...
	cg.message('->D');
}

function D() {
	// ...
}

// Create channels, using names that help understanding the flow
cg.channel({name:'->B', onConsume: B});
cg.channel({name:'->C', onConsume: C});
cg.channel({name:'->D', onConsume: D});

A();

```

##### Serial execution #1 (collect and process data, then output result)
```js
// Add numbers from 1 to 100, show the result

var channel = cg.channel({ 	// Channel to perform the addition
	structure:'integer',	// Data messages will be integers
    scope: {sum:0},			// Calculate sum here
    onConsume: function (n,scope) { // Executes when a message is received
    	scope.sum += n;
    },
    onResume: function(scope) {
    	console.log(scope.sum);
    },
});

// Add numbers
for (var i=0;i<100;i++) {
	cg.message(channel,i);
}

// Show result
cg.resume(channel);	
```

##### Serial execution #2 (collect data, then process data, then output result)
```js
// Add numbers from 1 to 50, show the result

var channel = cg.channel({ // Channel to perform the addition
	structure:'integer',	// Data messages will be integers
    scope: {sum:null},  // We can init scope later
    onConsume: function (n,scope) { // Consume each data message (after resuming)
    	scope.sum += n;
    },
    onPause: function(scope) { // Executes when pausing
    	scope.sum = 0;
    },
    onResume: function(scope) { // Executes when resuming
    	console.log(scope.sum);
    },
});

// Pause channel. Numbers are stored, but they are not added yet
cg.pause(channel);

// Collect numbers
for (var i=0;i<50;i++) {
	cg.push(channel,i);
}

// Resume channel. Numbers are added, then sum is displayed
cg.resume(channel);
```

##### Parallel execution
```js
// Execute A 2 times and B 3 times, then execute B

```

##### Recursively read folder contents
```js
cg.channel({	
	name:'folderContents',
    structure: {
    	folders: {
        	type: 'array',
            structure: {
            	name: 'string',
                contents: {type:'folderContents'}
            },
        },
        files: {
        	type: 'array',
            structure: {
		    	name: 'string',
		        size: 'integer',
                modified: 'datetime',
            },
        },
    },
    onConsume: function(plainString) { //Called whenever there is an incoming message
		console.log(plainString);	// Perhaps do some asynchronous work here
		cg.message('channelB',plainsString);	// Done, send a message to another channel
    }
});

// Mix with callbacks
fs.readdir('*.pdf',function(err,files){
	files.forEach(function(file){
		cg.message('countFile');
    });
});

```

## API
Objects:
[require]:
-  __Object returned by require('codegrid')__
	- __Properties__

		- __.channel__([options])
			- __Purpose__
			Creates a new channel
			- __Usage__
			Call .channel(options) to create a new channel.
			- __Arguments__
            	- [__options__]
            	An optional argument to determine the channel properties
            	If omitted, an anonymous channel will be created that receives empty data messages
                If options is a string, an already existing channel with this name will be returned (or null if no such channel exists).
            	If options is an object, it may contain the following properties that define the channel behaviour:
	        		- [__name__]
					A unique string identifier for the new channel.
			      	If a channel with that name exists, an error will occur.
			        If omitted (anonymous channel), a unique name will be generated.
					- [__structure__]
		        	Defines the structure of the data messages handled by this channel.
                    If object, see Structure object below for available properties.
			        If string, data message must be a single variable of this type. See property .type of Structure object for available types.
					- [__scope__]
			        Object whose properties will be variables local to the channel and accesible by its event handlers.
                    If omitted, an empty scope {} will be created.
                    If present, scope will be initialized with this value.
		    		- [__onConsume__]
		        	function([message[,scope]]) is fired when a message is ready to be consumed and consumes that message.
					- [__onError__]
			        function([error[,scope]]) is fired when an error occurs.
		        	- [__onPause__]
		        	function([scope]) is fired when channel enters in paused state.
		    		- [__onResume__]
			        function([scope]) is fired when channel leaves paused state and all pending messages have been consumed.
		        	If sending a resume control message while in normal state, the channel will consume any previous data messages and then will fire the onResume handler immediately.
			- __Returns__
			A channel object.
            If an error occured in channel creation, channel.error holds the error details.

		- __.message__(channel[,message])
			- __Purpose__
			Sends a data message to a channel
			- __Usage__
				- .message(channel) sends an empty data message to a channel. Channel should allow only empty data messages.
            	- .message(channel,message) sends a data object to a channel. Data object must be valid according to the channel structure.
			- __Arguments__
            	- [__channel__]
            	The channel to send the data.
                If string, it is the channel name.
                If object, it is the channel object returned from .channel()
            	- [__message__]
            	The data object to send to the channel.
                When channel receives the message, it will perform validations to check if it complies to the declared structure for the objects handled by that channel.

		- __.messages__(channel,messages)
			- __Purpose__
			Sends an array of data messages to a channel
			- __Usage__
				- .messages(channel,n) sends n empty data message to a channel, where n is an integer. 
            	- .messages(channel,arr) sends some data messages to a channel, where arr is the array containing the data messages.
			- __Arguments__
            	- [__channel__]
            	The channel to send the data.
                If string, it is the channel name.
                If object, it is the channel object returned from .channel()
            	- [__messages__]
                If messages is a number, it is the number of empty data messages to send.
                If messages is an array, it contains the data messages to send.

		- __.pause__([channel[,count]])
			- __Purpose__
			Sends a pause control message to set the channel in paused state.
            Channel enters the paused state and onPause handler fires.
            When in paused state, channel will receive and store messages but will not consume them.
			- __Usage__
				- .pause(channel) sends a pause control message. Channel will resume to normal state after receiving a resume/flush control message.
            	- .pause(channel,count) sends a pause control message to set channel in paused state for a predefined number of data messages. After receiving the predefined number of data messages, channel will resume automatically and return to the normal state, without requiring a resume control message.
			- __Arguments__
            	- [__channel__]
            	The channel to send the data.
                If string, it is the channel name.
                If object, it is the channel object returned from .channel()
            	- [__count__]
            	The number of data messages to receive before resuming automatically.
                If count is 0 or null, a resume control signal will be added immediately in the queue.

		- __.resume__([channel])
			- __Purpose__
			Sends a resume control message to set the channel in normal state. 
            If there are any data messages pending, they are consumed in the order they were arrived.
            The onResume handler fires then and the channel enters the normal state.
			- __Usage__
				- .resume(channel) sends a resume control message.
			- __Arguments__
            	- [__channel__]
            	The channel to send the data.
                If string, it is the channel name.
                If object, it is the channel object returned from .channel()

- __Structure object required in channel()__
	- __Properties__
		- __[type]__
		String that indicates the data type. Available types are:
        	- __string__ Data must be a string
        	- __integer__ Data must be an integer
        	- __number__ Data must be a number
        	- __datetime__ Data must be a Datetime object
        	- __boolean__ Data must be a boolean
        	- __array__ Data must be an array. Structure of array items is determined by property _structure_
        	- __object__ Data must be an object. Object properties are defined in property _structure_
		- __[structure]__
		Variable with different meanings, depending on type:
        	- If type is __object__, structure is an object defining the available properties for data. 
        	Property key is the name of the property in data.
            Property value is a Structure object that defines the rules for the property value in data.
        	- If type is __array__, structure is a Structure object that defines the rules for each array item in data.
		- __[optional]__
		If exists and it is true, then this value is optional.
        If omitted, then this value is required.
		- __[min]__
		Variable with different meanings, depending on type:
        	- If type is __string__, min is an integer defining the minimum acceptable number of characters in data
        	- If type is __integer__, min is an integer defining the minimum acceptable value of data
        	- If type is __number__, min is a number defining the minimum acceptable value of data
        	- If type is __datetime__, min is a datetime defining the minimum acceptable value of data
        	- If type is __array__, min is an integer defining the minimum acceptable length of the array items
		- __[max]__
		Variable with different meanings, depending on type:
        	- If type is __string__, max is an integer defining the maximum acceptable number of characters in data
        	- If type is __integer__, max is an integer defining the maximum acceptable value of data
        	- If type is __number__, max is a number defining the maximum acceptable value of data
        	- If type is __datetime__, max is a datetime defining the maximum acceptable value of data
        	- If type is __array__, max is an integer defining the maximum acceptable length of the array items
       	
- __Channel object returned by channel()__
	- __Properties__

		- __.message__([,message])
		- __.messages__([,messages])
		- __.pause__([count])
		- __.resume__() 
        The above functions are equvalent to the corresponding library functions. 
        The only difference is that the channel argument is not required.
		- __.onConsume__(function) 
        Define the consume event handler
		- __.onError__(function) 
        Define the error event handler
		- __.onPause__(function) 
        Define the pause event handler
		- __.onResume__(function) 
        Define the resume event handler

