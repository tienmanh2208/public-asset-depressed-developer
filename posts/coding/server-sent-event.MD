# Server sent event (SSE) technique

## 1. What is server sent event
- What is this

Server sent event is a technical that server will send back to client multiple message before close the connection.
You can see the diagram bellow, compare between normal flow with server sent event.

<img alt="Flow of SSE" src="https://depresseddeveloper.cloud/media/images/sharing/server_sent_event_compare.png" style="width: 600px;">

- Pros
  - Client will be updated with the latest status of BE in real-time without make another request to BE to know.
  - Keep the connection alive, this can be very useful because some constraint about idle time between server and client (For example: If you use ALB to distribute request to server, the ALB will have an idle time, this is the maximum amount of time the connection between client and server remain idle - is the state that client and server does not send and data to each other - before timeout. If you use server sent event, you can keep the connection alive by send back to client a message after a period of time)
- Cons
  - The server has to make room for this connection so that this connection can remain alive. But server has limited alive connection at a time, so if too many connection alive, it may block other request that want to connect to server.
  - If you do not handle it well, this connection can alive forever.
- Use cases
  - Use to keep a connection alive
  - Get progress of long-running processes in server without making new request to know the status

## 2. How to use server sent event

### 2.1. Normal flow - using for get method

**Server side**

You need to define the `Content-type` of response to `text/event-stream`
And then you'll send the data back to client with following format:

```js
id: <event_id>\nevent: <event_name>\ndata: <event_data>\n\n
```

- event_id (optional): is the id of your event. This is not a mandatory field. If you do not send id of an event, it'll have the same id with the previous event.
- event_name (optional): type of the event. You can define any type of event you want depends on your use case needs. You may want to separate event so client can know the status of your process like `error`, `message`, `info`, `init`, `finish`, ...
- event_data (required): data of your event. This field is required so client can get your event. If you send event_data without keyword `data`, client will not be able to get your event.

This is the code example in server side using Nodejs

```js
// Import the http module
const http = require('http');

// Create a server object
const server = http.createServer((req, res) => {
    // Set the response HTTP headers
    res.setHeader('Access-Control-Allow-Origin', '*'); // Allow all origins (you can specify a specific origin)
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type'); // Allow Content-Type in request header
    res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS'); // Allow POST and OPTIONS methods

    if (req.url === '/sse') {
        // Handle SSE (Server-Sent Events) requests
        res.writeHead(200, {
            'Content-Type': 'text/event-stream', // SSE content type
            'Cache-Control': 'no-cache',         // Prevent caching
            'Connection': 'keep-alive'           // Keep the connection alive
        });

        res.write(`id: ${Date.now()}\nevent: init\ndata: Init flow stream\n\n`);
        res.write(`event: message\ndata: Event without id\n\n`); // This event has no id, so the id of this event will be the same as the above event
        res.write(`data: Event without id and event\n\n`); // This event has no id and type, so the id of this event will be the same as the above event and the type is setted to default value (message)
        res.write(`id: ${Date.now()}\nevent: error\ndata: Event type error\n\n`);


        // Send periodic updates every 2 seconds
        const interval = setInterval(() => {
            res.write(`id: ${Date.now()}\ndata: Default event type message\n\n`);
        }, 2000);

        setTimeout(() => {
            res.write(`id: ${Date.now()}\nevent: final\ndata: Final event\n\n`);
            clearInterval(interval);
            res.end();
        }, 7000)
    } else {
        // Handle other routes
        res.writeHead(200, { 'Content-Type': 'text/html' });
        res.write('<h1>Welcome to the Home Page!</h1>');
        res.end();
    }
});

// Define the port the server will listen on
const port = 3000;

// Start the server and listen for requests
server.listen(port, () => {
    console.log(`Server is running on http://localhost:${port}`);
});
```

To run server, save above code to server.js, run the command:

```bash
node server.js
```

And then, open browser and open the link `localhost:3000/sse`. To view the event server send to client, open the `network` tab and select `EventStream`, you'll the list event you sent from server.
There are 3 columns corresponding to the 3 pieces of information: Id, Type and Data we defined in the above.

<img alt="Example response" src="https://depresseddeveloper.cloud/media/images/sharing/eventstream_example_from_server.png" style="width: 800px">

**Client side**

In Nodejs, you can use `EventSource` to listen message sent from server using SSE.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>HTTP Request with Cookies</title>
</head>
<body>
<script>
  const source = new EventSource('http://localhost:3000/sse');

  source.addEventListener('init', (event) => {
    console.log('Received init event ===>', `data: ${event.data}, `, `id: ${event.lastEventId}`);
  });

  source.addEventListener('error', (event) => {
    console.log('Received error event ===>', `data: ${event.data}, `, `id: ${event.lastEventId}`);
  });

  source.addEventListener('final', (event) => {
    console.log('Received final event ===>', `data: ${event.data}, `, `id: ${event.lastEventId}`);
    source.close(); // Close the connection and stop receive msg
  });

  source.onmessage = (event) => {
    console.log('Received message event ===>', `data: ${event.data}, `, `id: ${event.lastEventId}`);

  };
</script>>
</body>
</html>
```

Open the console log, you can see log as follow:

```text
Received init event ===> data: Init flow stream,  id: 1727541083977
Received message event ===> data: Event without id,  id: 1727541083977
Received message event ===> data: Event without id and event,  id: 1727541083977
Received error event ===> data: Event type error,  id: 1727541083977
Received message event ===> data: Default event type message,  id: 1727541085978
Received message event ===> data: Default event type message,  id: 1727541087978
Received message event ===> data: Default event type message,  id: 1727541089979
Received final event ===> data: Final event,  id: 1727541090978
```

### How to use with others methods (POST/PUT/...)

SSE does not natively support others methods. But, you can implement this with special approach.

**Server side**

Create endpoint `sse` with `POST` method

```js
    else if (req.method === 'POST' && req.url === '/sse') {
        res.writeHead(200, {
            'Content-Type': 'text/event-stream',
        })

        const interval = setInterval(() => {
            res.write(`Hello ${Date.now()}\n`)
        }, 2000)

        setTimeout(() => {
            clearInterval(interval);
            res.end()
        }, 7000)
    }
```

**Client side**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HTTP Request with Cookies</title>
</head>
<body>
<script>
    const xhr = new XMLHttpRequest();
    xhr.open('POST', 'http://localhost:3000/sse', true);
    xhr.setRequestHeader('Content-Type', 'application/json');

    xhr.onreadystatechange = function () {
        if (xhr.readyState === XMLHttpRequest.HEADERS_RECEIVED) {
            // Start receiving the SSE stream
            const stream = xhr.responseText;
            console.log('Received stream:', stream);
        }
    };

    xhr.onprogress = function () {
        // const newEvent = xhr.responseText.split('\n').slice(-2, -1)[0];
        console.log('New SSE event:', xhr.responseText);
    };

    xhr.onerror = function () {
        console.error('SSE stream error');
    };

    xhr.onloadend = function () {
        console.log('Load end, full data:', xhr.responseText);
    }

    // Send the POST request with data
    xhr.send(JSON.stringify({ init: true }));
</script>>
</body>
</html>
```

The log in client will be:

<img alt="Log from sse using post method" src="https://depresseddeveloper.cloud/media/images/sharing/sse_post_method.png" style="width: 400px">

You can see that, each time server send an event to client, the `onprogress` is triggered and we can get the response text *up to that time*. It means you will get data as combination of all the previous response include this time. It's not like when using EventSource, it separates response between each event. You can fix this by splitting response and get the last element.

If you still want to use SSE using EventSource, you can split the process to two steps:

- The first api is `POST` request to server, server will return a `process_id` to client
- The client will use another api with `GET` method, pass the process_id as the param to get the progress of the process we requested above

### 2.3. Authentication with SSE

When you use SSE with EventSource, it does not let you change the headers, so some authentication flow using Bearer or Token in headers can not be used this time.

So to overcome this issue, you can change the authentication mechanism for only this endpoint, instead using token stored in header, we pass the token to params, and you create a custom middleware in server to authenticate this endpoint.