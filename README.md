# codegrid
An easy &amp; structured way to use events, streams and promises in Node AND browser. Suitable for large scale apps.
## Status
Under construction!

### Introduction

**codegrid** is a lightweight framework designed to simplify Javascript asynchronous programming, with emphasis on program readability. It can be used in both client (web browser) and server (node.js).

### Definitions
- __message__ A data object to transfer asynchronously between two pieces of code.
- __structure__ A set or XML-like rules defining the properties of a message object.
- __channel__  A FIFO message buffer with awaiting and promise-like behaviour.
	- __onMessage__ is a channel function that executes when a message arrives.
	- __push__  is the action of sending a message to a channel.
	- __pull__ is the action of requesting a message from a channel.

## Quick usage

Include in modules of a Node.js application:
```js
var cg = require('codegrid');
```
A minimal usage:
```js
// A simple structure
cg.structure({
	name:'plainString',
    type:'string'
});

// A simple channel
cg.channel({
	name:'channelA',
    structure: 'plainString',
    onMessage: function(plainString) { //Called whenever there is an incoming message
		console.log(plainString);	// ... or do some asynchronous work here
		cg.emit('channelB',plainsString);	// Done, send a message to another channel
    }
});

```
Execute in series, use inside callbacks:
```js
// Merge some files
cg.channel({	
	name:'mergeFiles',
    structure: 'plainString',
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
