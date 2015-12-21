# codegrid
An easy &amp; structured way to use events, streams and promises in Node AND browser, suitable expecially for large scale apps.
## Status
Under construction!

## Introduction

**codegrid** is a lightweight framework designed to simplify Javascript asynchronous programming, with emphasis on program readability. It can be used in both client (web browser) and server (node.js). It can even be mixed to send data from client to server and vice versa!

While promises implement asynchronous programming by giving emphasis on the application code, codegrid puts its weight on the application data. This gives codegrid a huge advantage when writing large scale applications, because the art of programming is more mature for creating large structures of data than creating large structures of code. 
There is usually a limit in creating complex code. Passing this limit leads to applications that are very hard to maintain. However, when we try to create complex data, we do not have this problem as long as we follow two important principles: Repeat and define our data structures. Furthermore, task automation is a lot more easier when managing data than managing code.

## How it works
To understand how codegrid works, imagine that your application can be drawn in a two-dimensional diagram:
- The horizontal axis in this diagram is your code flow, while the vertical axis is your data flow. 
- A sequence of code instructions (function) is drawn as a shape with a minimum height and a certain width. 
- A sequence of data objects (channel) is drawn as a shape with a minimum width and a certain height. 
- Arrows from functions to channels indicate the data messages that functions emit to the channels.
- Arrows from channels to functions indicate the event handlers that fire on channel events.

## Definitions
- __channel__  A FIFO buffer.
A channel receives messages, stores them in the FIFO buffer and processes them. 
Process may involve firing an event handler, firing an error or pausing temporarily until a resule message arrives.
- __handler__ 
A function that fires when a specific condition occurs in a channel.
Possible handlers are:
	- __receive handler__ Fires when a message is received
	- __error handler__ Fires when an error occurs inside a handler or an error message is received
	- __pause handler__ Fires when a pause message is received
	- __resume handler__ Fires when a resume message is received
- __message__ 
A data object with a specific structure. 
Messages are sent from code to channels. When they arrive, they fire event handlers.
They may be application messages (with user defined structure)
- __scope__  
A special data container for channel local variables. These local variables are available to the channel handlers
 
A message handler may process the incoming message and place some data here.  
	- __handler__ The function that fires when a specific event -usually message arrival- occurs in a channel.
	- __consume__ The action of calling the handler for incoming messages of a channel.
	- __emit__  The action of sending a message to a channel. Message will be consumed immediately.
	- __push__  The action of sending a message to a channel. Message will be stored and will not be consumed immediately.
	- __pop__ The action of 
- __structure__ A set or XML-like rules defining the properties of a message object.
	- __onMessage__ is a channel function that executes when a message arrives.
	- __pull__ is the action of requesting a message from a channel.

## Usage examples

Including in a Node.js module:
```js
var cg = require('codegrid');
```

Performing a simple callback without arguments:
```js
// A executes first, then B executes. 

function A() {
	// ... do a time consuming task ...
    cg.emit(channel);	// Send message to channel when task is finished
}

var channel = cg.channel({	// Create an anonymous channel
	onMessage: function B(){ // Fired when a message is received in channel
    	// ... do another task ...
    }
});
```

Two functions share the same callback:
```js
// B executes after each execution of A1 or A2

function A1() { 
	// ... do a time consuming task ...
	cg.emit('channel1');
}

function A2() { 
	// ... do a time consuming task ...
	cg.emit('channel1');
}

cg.channel({	// Create a named channel
	name: 'channel1',
	onMessage: function B(){
    	// ... do another task ...
    }
});
```

Passing a complex object from an asynchronous function to another function:
```js
// A executes first and sends a complex object to channel, 
// then channel validates the object and calls B passing the object

function A() {
	// ... Read a person from database to person ...
    person.source = 'database';
    cg.emit(persons,person);	// Send person to channel. It's ok if channel does not exist yet
}
var persons = cg.channel({	// Create an anonymous channel for persons
	structure: {	// Person properties:
    	id: 'integer',	// Person ID
        name: {type:'string',min:2,max:50},	// Person name (2...50 characters)
        age: {type:'integer',min:12,optional:true}, // Optional person age (12...)
        source: 'string',							/// Information source
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

cg.channel({name:'A-to-B', onMessage: B});
cg.channel({name:'B-to-C', onMessage: C});
cg.channel({name:'C-to-D', onMessage: D});

function A() {
	... Do A...	
	cg.emit('A-to-B');
}

function B() {
	... Do B...	
	cg.emit('B-to-C');
}

function C() {
	... Do C...	
	cg.emit('C-to-D');
}

function D() {
	... Do D...	
}

// Converting the above functions to promises would need a significant amount of effort
```

Serial execution #1 (collect and process data, then output result):
```js
// Add numbers from 1 to 100

var channel = cg.channel({
	structure:'integer',
    scope: {
    	sum:0
    },
    onMessage: function (n) { // Executes when a message is received
    	this.scope.sum += n;
    },
    onResume: function() {
    	console.log(sum);
    },
});

// Collect data
for (var i=0;i<100;i++) {
	cg.emit(channel,i);
}

// Execute onResume()
cg.resume(channel);	
```

Serial execution #2 (collect data, then process data, then output result):
```js
// Add numbers from 1 to 100

var channel = cg.channel({
	structure:'integer',
    scope: {sum:null},
    onMessage: function (n) { // Executes for each message when all messages are popped
    	this.scope.sum += n;
    },
    onResume: function() {
    	console.log(sum);
    },
});

// Place channel in pause state. Messages are received and stored, but they are not processed
cg.pause(channel);

// Collect data
for (var i=0;i<100;i++) {
	cg.push(channel,i);
}

// Prepare to execute onMessage()
cg.scope(channel).sum = 0;

// Execute onMessage() for all messages, then execute onResume()
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
