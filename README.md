# The P3 Network Protocol
> Note: the P3 network and protocol were developed by [Sys36](https://sys36.net/), creators of [Windows 96](https://windows96.net/).
> This documentation is unofficial.

## What is P3?
> Fun fact: during the development of Windows 96 v3, P3 was intended to be replaced by a new WIP system called "Bell". When the release of v3 was nearing, Bell was nowhere near finished, so it was removed from some of the later beta builds, as well as the initial relase, and P3 was left in. 
P3 (aka PPP/Psuedo P2P) is a network developed by Sys36 for use in the Windows 96 web OS.
A P3 network consists of 2 types of devices: the P3 server, and the devices using the P3 network.
P3 uses SocketIO to simulate a P2P network over WebSockets and HTTP(s). The main goal is to have P2P-like functionality within Windows 96.

## P3 Network "Relay" Server
The server is the most important part. It operates basically the entire network.
Without a P3 network "relay" server, you wouldn't be able to simulate a P2P network.

## P3 Keys
A P3 key is base-64 encoded text. These keys are usually generated by generating random hexidecimals or text, and then encoding them with base-64.
A P3 key is used to allow the relay server to identify which device you are on.
The server then assigns you a P3 address, a collection of random letters and numbers that ends in `.ppp`.
Your P3 address is associated with your P3 key, so transferring your P3 key to a different device allows you to retain your current P3 address.

## Connection and Handshake
When your device connects to a P3 network, it sends the `hello` event to the relay server.
It includes one paramater, which is your P3 key.
> Fun fact: when decoded, this base-64 data says "your key here".
```json
"eW91ciBrZXkgaGVyZQ=="
```

Your device then recieves a `hello` event from the relay server.
If your device successfully connects to the P3 network, the response will contain data similar to this:
```json
{
  "success": true,
  "address": "ba9odkq4d.ppp"
}
```
If your device doesn't connect successfully, the relay server will send data similar to this:
> Fun fact: the most commonly recieved error message is `PPP Server Error: Address already in use`, which means that your trying to use a key that is being used on a device already connected to the P3 network.
```json
{
  "success": false,
  "message": "PPP Server Error: <insert error message here>"
}
```

## Hosting a server over P3
When someone tries to connect to your P3 address, they also specify a port to connect to. To my knowledge, a P3 port can be any number, and isn't limited, unlike ports over typical IP connections (this isn't known for sure, though).
Your device recieves data formatted like the following through the `packet` event:
```json
{
  "port": 1234,
  "data": {
    "type": "connect",
    "responsePort": 19419
  },
  "source": "ak4jd7xa0e.ppp"
}
```
If your server wants to allow the connection, it'd send a similar response back through the `packet` stream:
```json
{
  "data": {
    "type": "ack",
    "message": "Connection accepted",
    "heartbeat": 15000,
    "nonce": 0,
    "peerID": "aGRrSkZJbmRERihFN2ViIGprLEhKTURoa2RvSk1DTVM=",
    "code": 100,
    "success": true
  },
  "dest": "ak4jd7xa0e.ppp:19419",
  "nonce": 0
}
```
The peer ID is generated using the same method as the P3 key is, except it uses more letters. The peer ID uniquely identifies the client-server connection to avoid confusing it with other possible connections.

## Sending messages
> Fun fact: the peer ID isn't required when the server sends a message to the client, because response ports are only used for one connection (unlike server ports)
Messages can be sent using the same `packet` stream. If you're sending from a server to a client, it might look like this:
```json
{
  "data": {
    "type": "message",
    "peerID": null,
    "success": true,
    "data": "<insert valid JSON data here>",
    "nonce": 0
  },
  "dest": "ak4jd7xa0e.ppp:19419",
  "nonce": 7
}
```
The client will recieve data like the following:
```json
{
  "data": { ... },
  "source": "ba9odkq4d.ppp",
  "port": 19419
}
```


If your sending to the server from the client, it might look a bit different:
```json
{
  "data": {
    "type": "message",
    "peerID": "aGRrSkZJbmRERihFN2ViIGprLEhKTURoa2RvSk1DTVM=",
    "success": true,
    "data": "<insert valid JSON data here>",
    "nonce": 7
  },
  "dest": "ba9odkq4d.ppp:123",
  "nonce": 0
}
```

The server will recieve data in a format similar to this:
```json
{
  "data": { ... },
  "source": "ak4jd7xa0e.ppp",
  "port": 123
}
```

## Disconnecting from a server
You can send the following format to disconnect from the server, still using the `packet` stream:
```json
{
  "data": {
    "peerID": "aGRrSkZJbmRERihFN2ViIGprLEhKTURoa2RvSk1DTVM=",
    "success": true,
    "type": "disconnect",
    "nonce": 0
  },
  "dest": "ba9odkq4d.ppp:123",
  "nonce": 0
}
```

On a server, you can use similar data to kick the client off:
```json
{
  "data": {
    "peerID": null,
    "success": true,
    "type": "disconnect",
    "nonce": 0
  },
  "dest": "ak4jd7xa0e.ppp:19419",
  "nonce": 0
}
```

To know when you're disconnected, look for the same thing you'd look for when recieving a message, except there will be no data and the type is `disconnect`.

## Libraries for connecting to P3 networks
If you don't feel like writing the libraries to interact with P3 yourself, here are some libraries you can use:
### [The Official P3 Network API](https://apidocs.windows96.net/)
The official library that comes packaged with Windows 96. There has been no known documentation, but hopefully the Windows 96 API Docs will contain documentation on the P3 API once their server upgrade is finished.
#### Pros
* Built directly into Windows 96
* Complete integration with Windows 96 and the Mikesoft-hosted P3 network
#### Cons
* Complicated API
* Only on Windows 96 v2sp2 and newer; also proprietary
* No known documentation

### [mikesoft-p3-node](https://github.com/themirrazz/mikesoft-p3-node)
The `mikesoftp3` Node.JS library is a very straightforward and easy-to-use API made by [themirrazz](https://github.com/themirrazz) for interacting with P3 using NodeJS.
It only requires the `socket.io-client` library as a dependency, which is very nice. 
It lets you have multiple P3 instances, which is useful for people that might need multiple P3 servers for different purposes, but are all related and might need access to each other without having to use IPC or something similar to that. Some errors in left over TypeDoc classes cause an error, however, there is a fixed version [here](https://github.com/themirrazzalt/ssh-over-p3/blob/main/node-p3.js) and [here](https://github.com/themirrazzalt/node-p3fs/blob/main/node-p3.js). You can install it using `npm install mikesoftp3`, but that version might be broken.

#### Pros
* Works perfectly with the official P3 network
* Easy-to-use
* Open-source (GNU GPLv3)
* Allows multiple P3 connections at once
* Works on Windows, Mac, and Linux
#### Cons
* Only supported in NodeJS
* Documentation is only accessed via API typings
