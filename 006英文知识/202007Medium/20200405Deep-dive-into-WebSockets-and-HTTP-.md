# How JavaScript works: Deep dive into WebSockets and HTTP/2 with SSE + how to pick the right path

Alexander Zlatkov

Nov 2, 2017 · 13 min read

[How JavaScript works: Deep dive into WebSockets and HTTP/2 with SSE + how to pick the right path](https://blog.sessionstack.com/how-javascript-works-deep-dive-into-websockets-and-http-2-with-sse-how-to-pick-the-right-path-584e6b8e3bf7)

This is post # 5 of the series dedicated to exploring JavaScript and its building components. In the process of identifying and describing the core elements, we also share some rules of thumb we use when building SessionStack, a lightweight JavaScript application that has to be robust and highly-performant in order to stay competitive.

If you missed the previous chapters, you can find them here:


An overview of the engine, the runtime, and the call stack


Inside Google’s V8 engine + 5 tips on how to write optimized code


Memory management + how to handle 4 common memory leaks


The event loop and the rise of Async programming + 5 ways to better coding with async/await


This time we’ll dive into the world of communication protocols, mapping and discussing their attributes and building parts on the way. We’ll offer a quick comparison of WebSockets and HTTP/2. At the end, we share some ideas on how to choose which way to go when it comes to network protocols.


Intro


Nowadays complex web apps that feature rich, dynamic UIs are taken for granted. And it’s not surprising — the internet has come a long way since its inception.


Initially, the internet wasn’t built to support such dynamic and complex web apps. It was conceived to be a collection of HTML pages, linking to one another to form the concept of「web」that contains information. Everything was largely built around the so-called request/response paradigm of HTTP. A client loads up a page and then nothing happens until the user clicks and navigates to the next page.


Around 2005, AJAX was introduced and a lot of people started to explore the possibilities of making connections between a client and а server bidirectional. Still, all HTTP communication was steered by the client, which required user interaction or periodic polling to load new data from the server.


Making HTTP「bidirectional」


Technologies that enable the server to send data to the client「proactively」have been around for quite some time.「Push」and「Comet」to name a few.


One of the most common hacks to create the illusion that the server is sending data to the client is called long polling. With long polling, the client opens an HTTP connection to the server which keeps it open until a response is sent. Whenever the server has new data that has to be sent, it transmits it as a response.


Let’s see how a very simple long polling snippet might look like:


This is basically a self-executing function that runs the first time automatically. It sets up the ten (10) second interval and after each async Ajax call to the server, the callback calls ajax again.


Other techniques involve Flash or XHR multipart request and the so-called htmlfiles.


All these workarounds share the same problem though: they carry the overhead of HTTP, which doesn’t make them well-suited for low-latency applications. Think multiplayer first-person shooter games in the browser or any other online game with a realtime component.


The introduction of WebSockets


The WebSocket specification defines an API establishing「socket」connections between a web browser and a server. In plain words: there is a persistent connection between the client and the server and both parties can start sending data at any time.


The client establishes a WebSocket connection through a process known as the WebSocket handshake. This process starts with the client sending a regular HTTP request to the server. An Upgrade header is included in this request which informs the server that the client wishes to establish a WebSocket connection.


Let’s see how opening a WebSocket connection looks like on the client side:


WebSocket URLs use the ws scheme. There is also wss for secure WebSocket connections which is the equivalent of HTTPS.


This scheme just starts the process of opening a WebSocket connection towards websocket.example.com.


Here is a simplified example of the initial request headers.


GET ws://websocket.example.com/ HTTP/1.1


Origin: http://example.com


Connection: Upgrade


Host: websocket.example.com


Upgrade: websocket


If the server supports the WebSocket protocol, it will agree to the upgrade and will communicate this through the Upgrade header in the response.


Let’s see how this can be implemented in Node.JS:


After the connection is established, the server replies by upgrading:


HTTP/1.1 101 Switching Protocols


Date: Wed, 25 Oct 2017 10:07:34 GMT


Connection: Upgrade


Upgrade: WebSocket


Once the connection has been established, the open event will be fired on your WebSocket instance on the client side:


Now that the handshake is complete the initial HTTP connection is replaced by a WebSocket connection that uses the same underlying TCP/IP connection. At this point, either party can start sending data.


With WebSockets, you can transfer as much data as you like without incurring the overhead associated with traditional HTTP requests. Data is transferred through a WebSocket as messages, each of which consists of one or more frames containing the data you are sending (the payload). In order to ensure the message can be properly reconstructed when it reaches the client each frame is prefixed with 4–12 bytes of data about the payload. Using this frame-based messaging system helps to reduce the amount of non-payload data that is transferred, leading to significant reductions in latency.


Note: It’s worth noting that the client will only be notified about a new message once all of the frames have been received and the original message payload has been reconstructed.


WebSocket URLs


We briefly mentioned before that WebSockets introduce a new URL scheme. In reality, they introduce two new schemes: ws:// and wss://.


URLs have scheme-specific grammar. WebSocket URLs are special in that that they do not support anchors ( #sample_anchor ).


The same rules apply to WebSocket style URLs as to HTTP style URLs. ws is unencrypted and has port 80 as default, while wss requires TLS encryption and has port 443 as default.


Framing protocol


Let’s take a deeper look at the framing protocol. This is what the RFC provides us with:


As of the WebSocket version specified by the RFC, there’s only a header in front of each packet. It’s quite a complex header, however. Here are its building blocks explained:


fin (1 bit): indicates if this frame is the final frame that makes up the message. Most of the time the message fits into a single frame and this bit will always be set. Experiments show that Firefox makes a second frame after 32K.


rsv1, rsv2, rsv3 (1 bit each): must be 0 unless an extension is negotiated that defines meanings for non-zero values. If a nonzero value is received and none of the negotiated extensions defines the meaning of such a nonzero value, the receiving endpoint must fail the connection.


opcode (4 bits): says what the frame represents. The following values are currently in use:


0x00: this frame continues the payload from the previous.


0x01: this frame includes text data.


0x02: this frame includes binary data.


0x08: this frame terminates the connection.


0x09: this frame is a ping.


0x0a: this frame is a pong.


(As you can see, there are enough values unused; they’ve been reserved for future use).


mask (1 bit): indicates if the connection is masked. As it stands right now, every message from a client to a server must be masked and the spec would want to terminate the connection if it’s unmasked.


payload_len (7 bits): the length of the payload. WebSocket frames come in the following length brackets:


0–125 indicates the length of the payload. 126 means that the following two bytes indicate the length, 127 means the next 8 bytes indicate the length. So the length of the payload comes in ~7bit, 16bit, and 64bit brackets.


masking-key (32 bits): all frames sent from the client to the server are masked by a 32-bit value that is contained within the frame.


payload: the actual data which most likely is masked. Its length is the length of the payload_len.


Why are WebSockets frame-based and not stream-based? I don’t know and just like you, I’d love to learn more, so if you have an idea, feel free to add comments and resources in the responses below. Also, a good discussion on the topic is available on HackerNews.


Data on the frames


As mentioned above, the data can be fragmented into multiple frames. The first frame that transmits the data has an opcode on it that indicates what sort of data is being transmitted. This is necessary because JavaScript has pretty much nonexistent support for binary data at the time the specification was started. 0x01 indicates utf-8 encoded text data, 0x02 is binary data. Most people will transmit JSON in which case you’d probably want to choose the text opcode. When you emit binary data it will be represented in a browser specific Blob.


The API for sending data through a WebSocket is very simple:


When the WebSocket is receiving data (on the client side), a message event is fired. This event includes a property called data that can be used to access the contents of the message.


You can easily explore the data in each of the frames in your WebSocket connection using the Network Tab inside Chrome DevTools:


Fragmentation


Payload data can be split up into multiple individual frames. The receiving end is supposed to buffer them up until the fin bit is set. So you can transmit the string「Hello World」in 11 packages of 6 (header length) + 1 byte each. Fragmentation is not allowed for control packages. However, the specification wants you to be able to handle interleaved control frames. That’s in case TCP packages arrive in arbitrary order.


The logic for joining frames is roughly the following:


receive the first frame


remember opcode


concatenate frame payload together until the fin bit is set


assert that the opcode for each package is zero


The primary purpose of fragmentation is to allow sending a message that is of unknown size when the message is started. With fragmentation, a server may choose a reasonable size buffer and, when the buffer is full, write a fragment to the network. A secondary use case for fragmentation is for multiplexing, where it is not desirable for a large message on one logical channel to take over the whole output channel, so the multiplexing needs to be free to split the message into smaller fragments to better share the output channel.


What is Hearbeating?


At any point after the handshake, either the client or the server can choose to send a ping to the other party. When the ping is received, the recipient must send back a pong as soon as possible. That’s a heartbeat. You can use it to make sure that the client is still connected.


A ping or pong is just a regular frame, but it’s a control frame. Pings have an opcode of 0x9, and pongs have an opcode of 0xA. When you get a ping, send back a pong with the exact same Payload Data as the ping (for pings and pongs, the max payload length is 125). You might also get a pong without ever sending a ping. Ignore it if it happens.


Heartbeating can be very useful. There are services (like load balancers) that will terminate idle connections. Plus, it’s not possible for the receiving side to see if the remote side has terminated. Only at the next send would you realize that something went wrong.


Handling Errors


You can handle any errors that occur by listening out for the error event.


It looks like this:


Closing the Connection


To close a connection either the client or server should send a control frame with data containing an opcode of 0x8. Upon receiving such a frame, the other peer sends a Close frame in response. The first peer then closes the connection. Any further data received after closing the connection is then discarded.


This is how you initiate the closing of a WebSocket connection from the client:


Also, in order to perform any clean up after the closing has completed, you can attach an event listener to the close event:


The server has to listen on the close event in order to process it if needed:


How do WebSockets and HTTP/2 compare?


While HTTP/2 has a lot to offer, it doesn’t completely replace the need for existing push/streaming technologies.


The first important thing to notice about HTTP/2 is that it’s not a replacement for all of HTTP. The verbs, status codes and most of the headers will remain the same as today. HTTP/2 is about improving the efficiency of the way data is transferred on the wire.


Now, if we compare HTTP/2 to WebSocket, we can see a lot of similarities:


As we have seen above, HTTP/2 introduces Server Push which enables the server to proactively send resources to the client cache. It does not, however, allow for pushing data down to the client application itself. Server pushes are only processed by the browser and do not pop up in the application code, meaning there is no API for the application to get notifications for those events.


This is where Server-Sent Events (SSE) become very useful. SSE is a mechanism that allows the server to asynchronously push the data to the client once the client-server connection is established. The server can then decide to send data whenever a new「chunk」of data is available. It can be considered as a one-way publish-subscribe model. It also offers a standard JavaScript client API named EventSource implemented in most modern browsers as part of the HTML5 standard by W3C. Note that browsers that do not support EventSource API can be easily polyfilled.


Since SSE is based on HTTP, it has a natural fit with HTTP/2 and can be combined to get the best of both: HTTP/2 handling an efficient transport layer based on multiplexed streams and SSE providing the API up to the applications to enable push.


To fully understand what Streams and Multiplexing are all about, let’s first have a look at the IETF definition: a「stream」is an independent, bidirectional sequence of frames exchanged between the client and server within an HTTP/2 connection. One of its main characteristics is that a single HTTP/2 connection can contain multiple concurrently open streams, with either endpoint interleaving frames from multiple streams.


We have to remember that SSE is HTTP-based. It means that with HTTP/2, not only can several SSE streams be interleaved onto a single TCP connection, but the same can also be done with a combination of several SSE streams (server to client push) and several client requests (client to server). Thanks to HTTP/2 and SSE, now we have a pure HTTP bidirectional connection with a simple API to let application code register to server pushes. Lack of bidirectional capabilities has often been perceived as a major drawback when comparing SSE to WebSocket. Thanks to HTTP/2 this is no longer the case. This opens up the opportunity to skip WebSockets and stick to an HTTP-based signaling instead.


How to choose between WebSocket and HTTP/2?


WebSockets will certainly survive the domination of HTTP/2 + SSE, mainly because it’s a technology already well adopted and, in very specific use cases, it has an advantage over HTTP/2 as it has been built for bidirectional capabilities with less overhead (e.g. headers).


Say you want to build a Massive Multiplayer Online Game that needs a huge amount of messages from both ends of the connection. In such a case, WebSockets will perform much, much better.


In general, use WebSockets whenever you need a truly low-latency, near realtime connection between the client and the server. Keep in mind that this might require rethinking how you build your server-side applications, as well as shifting the focus on technologies like event queues.


If your use case requires displaying real-time market news, market data, chat applications, etc., relying on HTTP/2 + SSE will provide you with an efficient bidirectional communication channel while reaping the benefits from staying in the HTTP world:


WebSockets can often be a source of pain when considering compatibility with existing web infrastructure as it upgrades an HTTP connection to a completely different protocol that has nothing to do with HTTP.


Scale and security: Web components (Firewalls, Intrusion Detection, Load Balancers) are built, maintained and configured with HTTP in mind, an environment that large/critical applications will prefer in terms of resiliency, security, and scalability.


Also, you have to take into consideration browser support. Have a look at the WebSocket:


It’s quite good actually, isn’t it?


The situation with HTTP/2, however, is not the same:


TLS-only (which is not so bad)


Partial support in IE 11 but only on Windows 10


Only supported on OSX 10.11+ in Safari


Only supports HTTP/2 if you can negotiate it via ALPN (which is something your server needs to support explicitly)


The SSE support is better though:


Only IE/Edge don’t provide support. (Well, Opera Mini supports neither SSE nor WebSockets so we can take it out of the equation altogether). There are some decent polyfills out there for SSE support in IE/Edge.


How we make the decision at SessionStack


We at SessionStack use both WebSockets and HTTP, depending on the case. Once you integrate SessionStack into your web app, it starts recording all DOM changes, user interactions, JavaScript exceptions, stack traces, failed network requests, and debug messages, allowing you to replay issues in your web apps as videos and see everything that happened to your users. It’s all taking place real-time and needs to happen with no performance impact for your web app.


This means that you can join a user session live, while the user is still in the browser. In this scenario, we have chosen to leverage HTTP, since there is no bidirectional communication (the server just「streams」the data to the browser). A WebSocket, in this case, will be an overkill really, harder to maintain and scale.


The SessionStack library that gets integrated into your web app, however, uses a WebSocket (if possible, otherwise falls back to HTTP). It’s batching and sending the data to our servers which is also a one-way communication. We chose WebSocket in this case because some of the product features that are on the roadmap would require a bidirectional communication.


If you’d like to try SessionStack to understand and reproduce technical and UX problems in your web apps, we provide a free plan that allows you to get started for free.


## Resources

http://lucumr.pocoo.org/2012/9/24/websockets-101/

http://blog.teamtreehouse.com/an-introduction-to-websockets

https://www.infoq.com/articles/websocket-and-http2-coexist

https://tools.ietf.org/html/rfc6455
