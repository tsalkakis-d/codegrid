# codegrid
Implement asynchronous programming in Node.js and browser Javascript, using event queues.
An alternative to callbacks and promises, easy to read, easy to expand.
## Status
Under construction!

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

## Usage examples

Include _codegrid_ in a Node.js module:
```js
var cg = require('codegrid');
```

Perform a simple callback without arguments:
```js
// A executes first, then B executes

function A() {
	// ... 
    cg.emit(channel);	// Send message to channel when A is finished
}

var channel = cg.channel({	// Create an anonymous channel
	onMessage: function B(){ // Fired to consume a received message
    	// ... 
    }
});
```

Two functions that call the same callback function:
```js
// Callback B executes after either execution of A1 or A2

function A1() { 
	// ... 
	cg.emit('channel1');
}

function A2() { 
	// ... 
	cg.emit('channel1');
}

cg.channel({	// Create a named channel
	name: 'channel1',
	onMessage: function B(){
    	// ... 
    }
});
```

Callback passing a plain string:
```js
// A executes first, then passes a string to B, then B executes

function A() {
	// ... 
    cg.emit(channel,'done');
}

var channel = cg.channel({	// Create an anonymous channel
	structure: 'string',
	onMessage: function B(s) {
    	console.log(s);
    }
});
```

Callback passing a complex object:
```js
// A executes first, then sends an object to channel, 
// then channel validates the object, then B consumes the object

function A() {
	// ... Read a person from database to ...
    person.source = 'database'; // Add some more info to person
    cg.emit(persons,person);	// Send person to B
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
	onMessage: function B(person){ // Called when a valid message is received
    	// Person is guaranteed to have the defined structure
    	console.log(person.name,person.address.street,person.address.streetNo);
    },
    onError: function (error) {	// Called if validation fails or if an exception occurs in message handler
    	console.log(error);
    }
});
```

Replacing a callback hell:
```js
// A calls B, then B calls C, then C calls D

cg.channel({name:'A-B', onMessage: B});
cg.channel({name:'B-C', onMessage: C});
cg.channel({name:'C-D', onMessage: D});

function A() {
	// ...
	cg.emit('A-B');
}

function B() {
	// ...
	cg.emit('B-C');
}

function C() {
	// ...
	cg.emit('C-D');
}

function D() {
	// ...
}

// Converting the above functions to promises would surely need more effort
```

Serial execution #1 (collect and process data, then output result):
```js
// Add numbers from 1 to 100, show the result

var channel = cg.channel({ 	// Channel to perform the addition
	structure:'integer',	// Data messages will be integers
    scope: {sum:0},			// Calculate sum here
    onMessage: function (n,scope) { // Executes when a message is received
    	scope.sum += n;
    },
    onResume: function(scope) {
    	console.log(scope.sum);
    },
});

// Add numbers
for (var i=0;i<100;i++) {
	cg.emit(channel,i);
}

// Show result
cg.resume(channel);	
```

Serial execution #2 (collect data, then process data, then output result):
```js
// Add numbers from 1 to 50, show the result

var channel = cg.channel({ // Channel to perform the addition
	structure:'integer',	// Data messages will be integers
    scope: {sum:null},  // We can init scope later
    onMessage: function (n,scope) { // Consume each data message (after resuming)
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


Parallel execution:
```js
// Execute A 2 times and B 3 times, then execute B

```

Recursively read folder contents:
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
    onMessage: function(plainString) { //Called whenever there is an incoming message
		console.log(plainString);	// Perhaps do some asynchronous work here
		cg.emit('channelB',plainsString);	// Done, send a message to another channel
    }
});

// Mix with callbacks
fs.readdir('*.pdf',function(err,files){
	files.forEach(function(file){
		cg.emit('countFile');
    });
});

```

## API
<table>
	<tr>
		<th>Object</th>
		<th>Method</th>
		<th colspan="2">Arguments</th>
		<th>Description</th>
		<th>Returns</th>
	</tr>
    <tr>
    	<td colspan="6">require('codegrid')</td>
    </tr>
    <tr>
    	<td rowspan="8"></td>
    	<td rowspan="8">channel([options])</td>
    	<td colspan="2"><b>options</b>: Object with properties:</td>
    	<td rowspan="8">Creates a new channel</td>
    	<td rowspan="8">A channel object.</td>
    </tr>
    <tr>
    	<td></td>
    	<td><b>name</b> (optional):
        A unique string identifier for the new channel.
        If a channel with that name exists, an error occurs.
        If omitted, a unique name is generated.
        </td>
    </tr>
    <tr>
    	<td></td>
    	<td><b>structure</b> (optional):
        Defines the data structure (see Structure object).
        If omitted, data messages must be empty.
        </td>
    </tr>
    <tr>
    	<td></td>
    	<td><b>scope</b> (optional): 
        Object whose properties are variables local to the channel and accesible by its event handlers.
        </td>
    </tr>
    <tr>
    	<td></td>
    	<td><b>onMessage</b> (optional):
        function([message[,scope]]) that consumes a message.
        </td>
    </tr>
    <tr>
    	<td></td>
    	<td><b>onError</b> (optional):
        function([error[,scope]]) that is called when an error occurs.
        </td>
    </tr>
    <tr>
    	<td></td>
    	<td><b>onPause</b> (optional):
        function([scope]) that is called when channel enters in paused state.
        </td>
    </tr>
    <tr>
    	<td></td>
    	<td><b>onResume</b> (optional):
        function([scope]) that is called when channel leaves paused state and all pending messages have been consumed.
        If sending a resume control message while in normal state, the channel will consume any previous data messages and then will fire the onResume handler immediately.
        </td>
    </tr>
    
</table>

