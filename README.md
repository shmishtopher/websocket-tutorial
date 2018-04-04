# WebAPIs: WebSockets

Learn the inner workings of websocks by building a live chat application!  We'll be using JavaScript, but the concepts learned here will carry over to any language.  Note: this tutorial will not be using any external dependancies, as it is designed to teach the inner workings of WebSockets.  For live applications, I reccomend you use the popular ["ws"](https://www.npmjs.com/package/ws) package.

1. [Setup](#setup)
2. [Building the backend](#building%20the%20backend)
3. [Designing the frontend](#designing%20the%20frontend)
4. [Challanges](#challanges)

## setup
First things first, we need to setup our development space.  Since we'll be using JavaScript in this workshop, you'll need to install Node.  A version of Node >= 7.0 is recommended.  Binaries and source files for all major operating systems can be found [here](https://nodejs.org/en/download/).  Simply follow the relevant installation guide for your OS.  You can test to ensure successful installation by running `node -v`, which will echo out the currently installed version.  With node installed we can get to work!  Start by creating an empty directory and opening it in your favorite text editor.  In the new directory, run `npm init` and fill out each prompted field according to the chart below.

| Field  | Value |
| ------------- | ------------- |
| package name | sockets |
| version | 1.0.0 |
| description | websockets workshop |
| entry point | server.js |
| test command | node . |
| git repository | {your repo} |
| keywords | none |
| author | {your name} |
| licence | MIT |

You project is now set up to launch your application whenever you rn `npm test`.  Trying that now however, will result in an error since we havn't created `server.js`.  So hop back into your text editor and create a file named `server.js`.  We also want something to serve our users, so create `client.html`, `client.css`, and `client.js`.

## building the backend
It's time to start programming!  Let's start by creating our server.  Hop into `server.js` and write the following:
```javascript
const { createServer } = require('http')

const server = createServer((req, res) => {
  res.end('Hello, Universe!')
})

server.listen(80)
```
Let's take a look at what exactly is going on here:  Line one is our import statement.  It uses a little trick called "object destructuring" `require('http')` returns the built-in `http` object, which contains all functions containing to http servers and clients.  The only function we need however, is `createServer`.  Placing the name we want inside curly braces will extract the value and give it a name in our global context.  This is known as a destructuring assignment.  Lines three through five create our server.  The `createServer` function takes one function as its argument, which is invoked everytime our server recives a request.  The function we pass to `createServer` needs two arguments, a request and response (abriviated here to `req` and `res`).  Remember that this function is invoked on each new connection, so in this case, every new cient recives the text: "Hello, Universe".  To actually start the server, we need to listen on an available port.  `server.listen(80)` opens our server to port 80 so we can start reciving requests.  Run `npm test`, open your web browser, and navigate to [`http://localhost:80`](http://localhost:80).  You should see the text: "Hello, Universe".

Now lets try serving up some HTML.  Enter your `client.html` file and add some markup.  We also need to modify our server so that it responds to the propper requests:
```javascript
const { createServer } = require('http')
const { createReadStream } = require('fs')

const server = createServer((req, res) => {
  switch (req.url) {
    case '/':
      res.writeHead(200, {'Content-Type': 'text/html'})
      createReadStream(`${__dirname}/client.html`).pipe(res)
      break
      
    default:
      res.writeHead(404)
      res.end('404 - not found :(')
      break
  }
})

server.listen(80)
```
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta name="description" content="A WebSocket Workshop">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
  </head>
  <body>
    <p>Hello!</p>
  <body>
</html>
```
Now if we run `npm test` again (note: you can close your privious `npm test` with `ctrl`/`cmd`+`c`) and navigate to [`http://localhost:80`](http://localhost:80) you should see your html being served.  Now that we can serve HTML, we can make requests for other resources, like CSS and JS.  Just be sure to handle those cases in `server.js`:
```javascript
const { createServer } = require('http')
const { createReadStream } = require('fs')

const server = createServer((req, res) => {
  switch (req.url) {
    case '/':
      res.writeHead(200, {'Content-Type': 'text/html'})
      createReadStream(`${__dirname}/client.html`).pipe(res)
      break
      
    case '/style':
      res.writeHead(200, {'Content-Type': 'text/css'})
      createReadStream(`${__dirname}/client.css`).pipe(res)
      break
      
    case '/app':
      res.writeHead(200, {'Content-Type': 'application/javascript'})
      createReadStream(`${__dirname}/client.js`).pipe(res)
      break
      
    default:
      res.writeHead(404)
      res.end('404 - not found :(')
      break
  }
})

server.listen(80)
```

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta name="description" content="A WebSocket Workshop">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" type="text/css" href="style">
  </head>
  <body>
    <p>Hello!</p>
    <script src="app"></script>
  </body>
</html>
```

```css
html, body {
  padding: 0;
  margin: 0;
  width: 100%;
  height: 100%;
  
  display: flex;
  justify-content: center;
  align-items: center;
  
  font-size: 2em;
  font-family: Helvetica;
}
```

Restart the server (`npm test`) to make sure everything is working, you should be able to serve static assets now!  Note: you only need to restart the server with `npm test` when you make changes to `server.js`, changes made to your static assets only require a browser reload.  While static assets are cool, we need our server to be able to hadle websocket requests.  You can find details of the spec we'll be implementing [here](https://tools.ietf.org/html/rfc6455).  In this example, we'll be listning for requests made to `/chat`.  The client is going to be sending us an http GET request that looks something like this:
```
GET /chat HTTP/1.1
Host: example.com:8000
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: jqAGlX2N4P4BNdFTRvVP9g==
Sec-WebSocket-Version: 13
```
Let's add some logic to `server.js` to respond to these requests.  Lukily, our http server has an `upgrade` event we can listen for, which will trigger whenever the client tries to initilize a websocket connection:
```javascript
const { createServer } = require('http')
const { createReadStream } = require('fs')
const { createHash } = require('crypto')

const MAGIC_STR = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11'
const connected = []

const server = createServer((req, res) => {
  switch (req.url) {
    case '/':
      res.writeHead(200, {'Content-Type': 'text/html'})
      createReadStream(`${__dirname}/client.html`).pipe(res)
      break

    case '/style':
      res.writeHead(200, {'Content-Type': 'text/css'})
      createReadStream(`${__dirname}/client.css`).pipe(res)
      break

    case '/app':
      res.writeHead(200, {'Content-Type': 'application/javascript'})
      createReadStream(`${__dirname}/client.js`).pipe(res)
      break

    default:
      res.writeHead(404)
      res.end('404 - not found :(')
      break
  }
})

server.on('upgrade', (req, socket) => {
  const key = createHash('sha1')
    .update(req.headers['sec-websocket-key'] + MAGIC_STR)
    .digest('base64')

  const header = [
    `HTTP/1.1 101 Switching Protocols`,
    `Upgrade: websocket`,
    `Connection: Upgrade`,
    `Sec-WebSocket-Accept: ${key}`
  ]

  socket.write(header.concat('\r\n').join('\r\n'))
  connected.push(socket)
})

server.listen(80)
```
