# codegrid
An easy &amp; structured way to use events, streams and promises in Node AND browser. Suitable for large scale apps.


### Introduction

**codegrid** is a lightweight framework designed to simplify Javascript asynchronous programming, with emphasis on program readability.


### Definitions

| Word       | Definition    | Details      |         
| ---------- | ------------- | ------------------- |
| message    | Data to transfer asynchronously between two pieces of code. | JS object. May have a name. Belongs to a structure.  |
| structure  | Class that defines a message structure | Special structured object, with XML-like properties |
| channel    | Messages container. | Acts mainly (but not solely) as a FIFO queue. transports messages between handlers. Implements promise-like behaviour. |
| handler    | Code that emits (and waits for) messages. | JS function. Sends messages to channels. Waits for messages from channels. |
