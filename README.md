# codegrid
An easy &amp; structured way to use events, streams and promises in Node AND browser. Suitable for large scale apps.


### Introduction

**codegrid** is a lightweight framework designed to simplify Javascript asynchronous programming, with emphasis on program readability.


### Definitions

| Word       | Definition    | Details      |
| ---------- | ------------- | ------------------- |
| message    | Data to transfer asynchronously between two pieces of code. | JS object that complies to a specific structure. |
| structure  | Class that defines a message type. | Object with well defined, XML-like properties |
| channel    | Messages container. | Acts mainly (but not solely) as a FIFO queue. transports messages between handlers. Implements promise-like behaviour. |
| handler    | Code that emits (and waits for) messages. | JS function. Sends messages to channels. Waits for messages from channels. |

## Usage example

Include on each module (Node.js):
```js
var cg = require('codegrid');
```
Declare structures to handle specific purpose messages:
```js
// User information
cg.structure({
	name: 'user.info',    // Sructure name
    fields: {             // Structure fields. Message may have fields with those names only
		username: {type:'string'},					    // A field should have a specific type only				
        password: {type:'string', optional:true},	    // Some fields may be optional.
        lastgizmo: {type:'user.gizmo', optional:true},  // A field may also be an object of another structure...
        allgizmos: {type:'array[user.gizmo]', min:0, max:50} // ... or an array of another structure items
    }
});

// A gizmo!
// Structure hoisting is allowed, as long as structure is not used yet
cg.structure({
	name:'user.gizmo',
    type:'string',  // Name and type is the minimum information to declare about a structure
});
```
Declare a channel to send and receive specific messages:
```js
cg.channel({
	name: 'incomingUsers',
    structure: 'user.info',
});
```
Declare functions to send and receive messages using channels:
```js
cg.handler({
	name: 'mainReceiver',		// Handler name is currently used for debugging and logging purposes only
    channel: 'incomingUsers',	// Select the channel to listen for messages
    handler: function(userInfo) { // Actual handler. Called whenever there is a message available in channel
    	console.log(userInfo);		// Do the handler job here
        var result = userInfo;		// When done, will send a message to another channel
        result.processed = true;
        cg.emit({					// Send a message to another handler
        	channel: 'processedUsers',
            message: result,
        });
    },
});
```

## API