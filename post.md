Web developers in 2015 find themselves with an embarrasment of riches in
open-source technologies for developing real-time apps:

- [RethinkDB][], on top of being a *really* well-designed document store in
  general, provides the killer feature of [changefeeds][], allowing servers to
  have the document store notify them whenever activity on a query occurs
  without having to continuously poll the database.
- [Node.JS][] allows server code to effortlessly switch between tasks while
  waiting on I/O.
- [engine.io][] allows developers to use WebSockets for real-time code, while
  still working through simpler mechanisms if the upgrade to WebSockets fails.
- [Primus][] allows developers to code against a number of real-time libraries
  (such as engine.io) using one simple WebSocket-like model, without having to
  worry about potential differences between implementations in terms of
  low-level details like reconnection.

[RethinkDB]: http://rethinkdb.com/
[changefeeds]: http://rethinkdb.com/docs/changefeeds/javascript/
[Node.JS]: https://nodejs.org/
[engine.io]: https://github.com/Automattic/engine.io
[Primus]: https://github.com/primus/primus

Using these technologies, not only can you *build* real-time apps easily, you
can *deploy* them easily on [Heroku][], with [Compose][] providing a RethinkDB
cluster.

[Heroku]: https://www.heroku.com/
[Compose]: https://www.compose.io/

Assuming you have a working knowledge of running shell commands, this should
give you a step-by-step breakdown of exactly what's required to deploy such an
app in production, as seen at https://chatroom-of-requirement.herokuapp.com/.

## My development environment

I'm using a free public workspace provided by Cloud9. You can create your own
at https://c9.io/ (I recommend creating a "Custom" workspace, or creating a
fresh README-initialized repo on GitHub and then cloning that), or you can take
a look at the workspace I'm using at
https://ide.c9.io/stuartpb/chatroom-of-requirement (all the files should look
the same as what you'd see [in the GitHub repo I'm pushing it to][repo]).

[repo]: https://github.com/stuartpb/chatroom-of-requirement

I've been using Cloud9 for all my development needs for the last two and a half
years, for [a wide number of reasons][SeattleJS slides]: suffice it to say that
it is, in my opinion, the only way to code.

[SeattleJS slides]: https://docs.google.com/presentation/d/1ckoGhFPK7mYdda58EBSKQAGz0qE2eKhRCQOUr9BOYBY/edit#slide=id.g538fb0397_0126

If you aren't going to use Cloud9, you should still be able to follow along, so
long as you are in a development environment with a [Bash][] shell and the
following dependencies installed:

- [Git][]
- [OpenSSH][]
- the [Heroku toolbelt][]
- [npm][] (which depends on, and comes with, Node.JS)

[Bash]: https://en.wikipedia.org/wiki/Bash_(Unix_shell)
[Git]: https://git-scm.com/
[OpenSSH]: https://en.wikipedia.org/wiki/OpenSSH
[Heroku toolbelt]: https://toolbelt.heroku.com/
[npm]: https://www.npmjs.com/package/npm

## Creating and configuring our app

First, you will need to create accounts for [Heroku][] and [Compose][].

If you haven't used Heroku before, you will need to
[add an SSH key to your Heroku account][Heroku SSH] - if you already have
[an SSH key you use for GitHub][GitHub SSH], I recommend using that.

[Heroku SSH]: https://devcenter.heroku.com/articles/keys#adding-keys-to-heroku
[GitHub SSH]: https://help.github.com/articles/generating-ssh-keys/

As for Compose, I recommend creating new credentials for accessing RethinkDB,
since you'll be including the private key of the keypair as part of your app.
We can create them by running this command:

```
ssh-keygen -t rsa -N '' -f compose_rsa
```

We can then paste the content of the newly-created `compose_rsa.pub` file
into the "Add a User" screen for your deployment (at
`https://app.compose.io/` + the name of your account + `/deployments/` +
the name of your deployment + `/users/new`).

After installing the Heroku toolbelt (if you haven't already done so),
we create the app (you may be prmopted for your username/password):

```
$ heroku apps:create
```

Then, we configure the new app with all the variables needed to connect to your
RethinkDB deployment through an SSH tunnel on Compose (shown here with example
values, as a single command broken across multiple lines with backslashes):

```
$ heroku config:set \
  COMPOSE_SSH_PUBLIC_HOSTNAME=portal.1.dblayer.com \
  COMPOSE_SSH_PUBLIC_PORT=10000 \
  COMPOSE_RETHINKDB_INTERNAL_IPS="10.0.0.98 10.0.0.99" \
  COMPOSE_SSH_KEY="$(cat compose_rsa)"
```

### Config variable breakdown

```
  COMPOSE_SSH_PUBLIC_HOSTNAME=portal.1.dblayer.com \
```

The public hostname of the SSH proxy for your deployment. This can be found in
the last column of the "Deployment Topology" table of your deployment's
dashboard page, or in the first line of the suggested SSH tunnel command in
"Connect Strings" on that page (after `compose@`). It should be a domain name
that ends with "dblayer.com".

```
  COMPOSE_SSH_PUBLIC_PORT=10000 \
```

The public port of the SSH proxy for your deployment. This can be found in the
same places as the proxy's hostname, and should be a number somewhere above
10000.

```
  COMPOSE_RETHINKDB_INTERNAL_IPS="10.0.0.98 10.0.0.99" \
```

A space-separated list (make sure the string is quoted) of the internal IP
addresses of your RethinkDB members (cluster nodes). These can be found in the
"Internal IP" column of the "Deployment Topology" table, on the first few rows
(make sure to *only* include the internal IPs of the RethinkDB members, and
*not* the internal IP of the SSH proxy). These should be IP addresses that
start with `10.`.

```
  COMPOSE_SSH_KEY="$(cat compose_rsa)"
```

This adds the private SSH key you generated for Compose. Make sure to quote the
value, as the file's contents will be expanded to a multi-line string (which
would otherwise be separated and interpreted as multiple arguments).

## The codebase

Once you have configured your app, you can proceed to dive into its code. I'm
going to list the contents of each file (or the commands involved in creating
them) here, which you are free to copy and paste to recreate the project from
scratch; however, you may find it easier to clone the finished project from
its GitHub repo at https://github.com/stuartpb/chatroom-of-requirement and
follow along by inspecting the checked-out files. Also, the word "indubitably"
is not supposed to appear in the published version of this article.

## What our app will look like

The app we're constructing presents a simple chat interface with four elements:

- A freeform room name, which all messages are associated with
- A scroll of messages that have been sent associated with the room name
- A username to associate sent messages with
- A field to compose and send messages from

I'm titling it the Chatroom of Requirement, after the [Room of Requirement][]
in the [Harry Potter][] books, which magically transforms into whatever room
you need it to be just by thinking of it.

[Room of Requirement]: http://harrypotter.wikia.com/wiki/Room_of_Requirement
[Harry Potter]: http://harrypotter.wikia.com/wiki/Main_Page

Of course, this design is far from a comprehensive chat client: it's very
sparingly styled, and it lacks any kind of anti-spam, moderation,
authentication or ownership mechanisms (which all sufficiently complex chat
services [eventually grow to need][IRC]). However, this is sufficiently useful
for private use, where any user who knows of the server can be trusted 
[(much like the Room of Requirement itself)][Protection]. This minimal design
also helps keep the amount of code not necessary to demonstrate what's needed
to implement a real-time app with these technologies to a minimum.

[IRC]: https://en.wikipedia.org/wiki/Internet_Relay_Chat_services
[Protection]: http://harrypotter.wikia.com/wiki/Room_of_Requirement#Protection

Introducing the extended features one may wish for in a more full-featured chat
application is left as an exercise for the reader: I recommend looking into [Passwordless][] and the other tutorials from [RethinkDB's documentation][]
for inspiration and guidance as to where and how to proceed.

[Passwordless]: https://passwordless.net/
[RethinkDB's documentation]: http://rethinkdb.com/docs/

## Our app's overall entrypoint: tunnel-and-serve.sh

Because Compose provides access to the RethinkDB servers only through SSH
tunneling (the most viable approach to secure connections until RethinkDB
[implements TLS natively][rethinkdb-3158]), we create this Bash wrapper to
instantiate an SSH tunnel for the lifetime of our app:

[rethinkdb-3158]: https://github.com/rethinkdb/rethinkdb/issues/3158

```bash,linenums=true
DYNO=${DYNO##*.}
DYNO=${DYNO:-$RANDOM}

ips=($COMPOSE_RETHINKDB_INTERNAL_IPS)
nodes=${#ips[@]}

identity=$(mktemp)
echo "$COMPOSE_SSH_KEY" >$identity

ssh -NT compose@$COMPOSE_SSH_PUBLIC_HOSTNAME -p $COMPOSE_SSH_PUBLIC_PORT \
  -o StrictHostKeyChecking=no -o LogLevel=error \
  -i $identity \
  -L 127.0.0.1:28015:${ips[$((DYNO % nodes))]}:28015 \
  -fo ExitOnForwardFailure=yes

node server.js & wait %%

trap exit SIGTERM SIGKILL
trap "kill 0" EXIT
```

### Getting (or faking) the dyno number

```bash
DYNO=${DYNO##*.}
DYNO=${DYNO:-$RANDOM}
```

These lines convert the [Heroku-provided dyno identifier][dyno vars] to the
dyno's number, or (in the event that Heroku discontinues the `$DYNO`
environment variable or it is otherwise unavailable) picks a number at random.
This dyno (or random) number is used later when picking which Compose-provided
RethinkDB cluster node to connect to.

[dyno vars]: https://devcenter.heroku.com/articles/dynos#local-environment-variables

### Listifying the internal IP addresses

```bash
ips=($COMPOSE_RETHINKDB_INTERNAL_IPS)
nodes=${#ips[@]}
```

This converts the `COMPOSE_INTERNAL_IPS` variable into an [array][Bash arrays]
that can be used by Bash, and saves the number of nodes as the length of the
IP array.

[Bash arrays]: http://tldp.org/LDP/abs/html/arrays.html

### Saving the SSH identity to a temporary file

```bash
identity=$(mktemp)
echo "$COMPOSE_SSH_KEY" >$identity
```

This saves the SSH key we added to our deployment (and our app's config
environment) earlier into a temporary file (due to a
[weird SSH behavior][so-101900] that makes it impossible to use
[process substitution][] with `ssh`).

[so-101900]: http://unix.stackexchange.com/questions/101900
[process substitution]: http://www.tldp.org/LDP/abs/html/process-sub.html

### The SSH command parameters (line 1)

```bash
ssh -NT compose@$COMPOSE_SSH_PUBLIC_HOSTNAME -p $COMPOSE_SSH_PUBLIC_PORT \
```

This tells the SSH client to connect to the SSH proxy Compose runs for you to
access your RethinkDB nodes through, while **N**ot running a command or
allocating a **T**erminal as the SSH client normally would (since we only want
to create the tunnel).

### The SSH command parameters (line 2)

```bash
  -o StrictHostKeyChecking=no -o LogLevel=error \
```

This tells the SSH client not to throw an error on encountering the remote
server for the first time and not having anybody available to approve it,
and to not print a warning that it's trusting this remote server.

### The SSH command parameters (line 3)

```bash
  -i $identity \
```

This tells the SSH client to use our configured SSH key as its identity to
connect to the remote server.

### The SSH command parameters (line 4)

```bash
  -L 127.0.0.1:28015:${ips[$((DYNO % nodes))]}:28015 \
```

This tells the SSH client to open a tunnel to the **L**ocal host, specifically
on the `127.0.0.1` loopback address on port 28015 (the default address and port
for RethinkDB driver connections). The other end of the tunnel is hooked up on
the remote server, to one of the RethinkDB nodes it can privately access via
their internal IPs (on the RethinkDB port `28015`).

The IP to connect to is picked from our list, based on the dyno/random number
we configured earlier. This works as a kind of low-rent load balancing
mechanism in the event that the app is scaled up to use multiple front-end
dynos, assuring that evenly-numbered Heroku dynos will connect to one cluster
node, and that odd-numbered nodes will connect to the other (or, if the dyno
number is not specified, the connections to nodes should be roughly evenly
distributed at random).

### The SSH command parameters (line 5)

```bash
  -fo ExitOnForwardFailure=yes
```

This tells the SSH client to,
[after establishing the tunnel][ExitOnForwardFailure], go to the background as
we run our next task (the Node.JS server script).

[ExitOnForwardFailure]: http://stackoverflow.com/a/12868885/34799

### Running the Node server

```bash
node server.js & wait %%
```

The Node server is also backgrounded, followed by a `wait` call, so that the
script can handle [signals][] in the event that the server does not exit on its
own (in other words, if the server doesn't crash).

[signals]: http://unix.stackexchange.com/questions/146756/forward-sigterm-to-child-in-bash

### Configuring traps

```bash
trap exit SIGTERM SIGKILL
trap "kill 0" EXIT
```

These configure the script to exit when receiving a termination signal,
and to clean up all its lingering jobs (ie. the tunnel and the server) when
exiting.

## Bringing up the app's scaffolding

To tell Heroku to use this script we've written to start the server, we can
create a simple, one-line Procfile:

```
$ echo 'web: bash tunnel-and-serve.sh' > Procfile
```

We can then use `npm init` to create an empty `package.json`, and then use
`npm install --save` to populate that `package.json` with our app's
dependencies (installing them locally in the process):

```
$ npm init
$ npm install --save engine.io primus rethinkdb endex express jade
```

I described `rethinkdb`, `engine.io`, and `primus` above. Of the other three,
`express` is used to simplify writing the plain HTTP routes of our app,
`jade` is used for simplifying our HTML syntax, and `endex` is used for
simplifying our database initialization.

(Note that you may have to specify `rethinkdb@2.0.0` in the `npm install`
command arguments, due to
[an issue with the RethinkDB driver's release numbering][rethinkdb#4436].)

[rethinkdb#4436]: https://github.com/rethinkdb/rethinkdb/issues/4436

We will create seven files:

- server.js
- pool.js
- socket.js
- http.js
- views/index.jade
- static/layout.css
- static/client.js

The first four files are for running our app's back-end server, while the last
three files define our app's front-end client. There's nothing saying you
*have* to lay an app out like this, but it allows for a relatively neat base
of separated concerns, on which to build upon as the app's complexity grows.

## The Node server's entrypoint: server.js

Once `tunnel-and-serve.js` has instantiated our connection to RethinkDB, we
enter our app server proper at `server.js`. This script ties together
the other components of our app:

```javascript,linenums=true
var http = require('http');
var Primus = require('primus');

var pool = require('./pool.js')();
var appHttp = require('./http.js')(pool);
var appSocket= require('./socket.js')(pool);

var httpServer=require('http').createServer(appHttp);
httpServer.listen(process.env.PORT || 5000, process.env.IP || '0.0.0.0');

var socketServer = new Primus(httpServer, {transformer: 'engine.io'});
socketServer.on('connection', appSocket);
```

### Running the component constructor modules

```javascript
var pool = require('./pool.js')();
var appHttp = require('./http.js')(pool);
var appSocket= require('./socket.js')(pool);
```

Each of our `pool.js`, `http.js`, and `socket.js` modules export a function
that constructs the component of the server that they provide (the shared
resources, HTTP server, and WebSocket (or WebSocket-like) connection handler,
respectively). We call these functions to construct an instance of each of
these components: first `pool.js` to establish our pool of connections (ie.
the connection to our RethinkDB database, which could be expanded with further
development to use connections to multiple nodes), then `http.js` and
`socket.js`, using this pool of connections, to construct the callback
functions we use for our HTTP and socket servers.

### Creating and starting the server

```javascript
var httpServer=require('http').createServer(appHttp);
httpServer.listen(process.env.PORT || 5000, process.env.IP || '0.0.0.0');

var socketServer = new Primus(httpServer, {transformer: 'engine.io'});
socketServer.on('connection', appSocket);
```

This creates an HTTP server using the callback app function we returned from
the constructor function defined in `http.js`, listening on the port defined
by the `PORT` environment variable (used by Heroku), and on the interface
defined by the `IP` variable (used in conjunction with the `PORT` variable by
Cloud9's "server preview" functionality). If no `PORT` is defined, we default
to port 5000 (a commonly-used port for previewing servers in development), and
if no `IP` variable is defined, we use `0.0.0.0` to listen on all available
network interfaces (as is the default when the second argument to
`httpServer.listen` is not defined).

Then, we pass our newly-created [Node HTTP server][] to the Primus constructor,
telling it to use `engine.io` as its real-time implementation (the
"transformer", in Primus's quirky [Hasbro-centric][TFWiki] nomenclature),
rather than the default of `ws` (which breaks if the client cannot make a
WebSocket connection).

[Node HTTP server]: https://nodejs.org/api/http.html#http_class_http_server
[TFWiki]: http://tfwiki.net/wiki/Main_Page

Ideally, it would be possible to handle all of this inside our HTTP app
constructor, including engine.io/Primus's handlers as middleware, but, due to
[a shortcoming of Node's API for handling HTTP upgrade requests][node#4813], we
have to pass in our entire HTTP server to the `Primus` constructor.

[node#4813]: https://github.com/joyent/node/issues/4813

## Setting up our server-wide assets: pool.js

This script creates a `pool` object that acts as a sort of global environment
for sharing assets across the different components of our server (since we're
defining these components in the form of Node modules which are otherwise
encapsulated, not sharing a global state). It also handles the initialization
of these assets, establishing the connections and ensuring they are ready.

```javascript,linenums=true
var r = require('rethinkdb');
var endex = require('endex');

module.exports = function poolCtor() {
  var pool = {};
  var serverReportError = console.error.bind(console);

  var conn;

  function runQueryNormally(query) {
    return query.run(conn);
  }

  var connPromise = r.connect().then(function (connection) {
    return endex(r).db('chatror')
      .table('messages')
        .index('room')
        .index('sent')
      .run(connection);
  }).then(function (connection) {
    conn = connection;
    pool.runQuery = runQueryNormally;
  }).catch(serverReportError);

  pool.runQuery = function queueQueryRun(query) {
    return connPromise.then(function () {
      return query.run(conn);
    });
  };

  return pool;
};
```

### Establishing our interface / internals

```javascript
module.exports = function poolCtor() {
  var pool = {};
  var serverReportError = console.error.bind(console);

  var conn;
```

This sets up our module's exported constructor function (which gets imported as
the return value from calling `require('./pool.js')` in `server.js`) to
construct a plain object (`pool`), to which we attach the interfaces for any
resources that may be shared across the components of our server.

The `serverReportError` function is used to catch any errors that may be thrown
in a promise's asynchronous execution and log them to the server's error log,
without bringing down the whole server.

The `conn` variable is where we will internally keep our connection to the
RethinkDB server, once it is established.

In the next section of code, we will add the `pool.runQuery` function to this
pool object, which is used in the socket handler for interacting with our
database.

### Establishing /initializing the database connection and handling queries

```javascript
  function runQueryNormally(query) {
    return query.run(conn);
  }
```

This defines the simple, straightforward function that will be used to run
queries once the connection to the RethinkDB database has been established.

However, this can only be used once the connection has been established:
trying to run a query before the connection is established would result in the
query failing. To work around this, we provide a more complicated function for
chaining requests to be executed after the connection is established and its
promise resolves:

```javascript
  var connPromise = r.connect().then(function (connection) {
    return endex(r).db('chatror')
      .table('messages')
        .index('room')
        .index('sent')
      .run(connection);
  }).then(function (connection) {
    conn = connection;
    pool.runQuery = runQueryNormally;
  }).catch(serverReportError);
```

This is where we actually initialize our connection to our RethinkDB server.
Once the connection is established, we use the [endex][] module to construct a
complex branching ReQL query that will ensure that all the tables and secondary
indexes we need for our app to work exist before we begin running queries
against them, then run that query on the connection. (This also sets our
connection to use, and create if not already created, a database named
`'chatror'`.)

[endex]: https://github.com/stuartpb/endex

After the database structure is ensured, we save the connection in the `conn`
variable so that it can be used by the straightforward `runQueryNormally`
implementation of `pool.runQuery`, which then replaces the more complex
queueing implementation the pool was initialized with (defined below).

```javascript
  pool.runQuery = function queueQueryRun(query) {
    return connPromise.then(function () {
      return query.run(conn);
    });
  };

```

The initial `queueQueryRun` implementation of `pool.runQuery`, for queries made
(ie. in requests) before the connection is established and the database is
ensured, returns a promise that will resolve at the end of the
post-connection-and-initialization promise chain.

Hopefully, in
[a future version of the RethinkDB driver][rethinkdb#3771 comment],
this complex shuffle for queueing queries will not be necessary, and will be
able to be safely removed.

[rethinkdb#3771 comment]: https://github.com/rethinkdb/rethinkdb/issues/3771#issuecomment-107127070

## Handling real-time connections on the server: socket.js

This script defines the handler for incoming socket connections provided by
Primus, similarly to how an HTTP app defines a handler for HTTP requests
provided by Node's `http` module.

```javascript,linenums=true
var r = require('rethinkdb');

var BACKLOG_LIMIT = 100;

function socketAppCtor(pool) { return function socketApp(socket) {

  function reportError(err){
    socket.write({
      type: 'error',
      err: err
    });
  }

  function createMessage(message) {
    return pool.runQuery(r.table('messages').insert({
      body: message.body,
      room: message.room,
      author: message.author,
      sent: r.now()
    }));
  }

  function deliverMessage(message){
    socket.write({
      type: 'deliverMessage',
      message: message
    });
  }

  var roomCursor = null;

  function closeRoomCursor() {
    if (roomCursor) {
      roomCursor.close();
      roomCursor = null;
    }
  }

  function setRoomCursor(cursor) {
    return roomCursor = cursor;
  }

  function joinRoom(roomName) {
    closeRoomCursor();
    pool.runQuery(
      r.table('messages').orderBy({index:r.desc('sent')})
        .filter(r.row('room').eq(roomName))
        .limit(BACKLOG_LIMIT).orderBy('sent'))
      .then(setRoomCursor)
      .then(function(){
        roomCursor.each(function (err, message) {
          if (err) return reportError(err);
          deliverMessage(message);
        }, switchToChangefeed);
      })
      .catch(reportError);

    function switchToChangefeed() {
      closeRoomCursor();
      pool.runQuery(r.table('messages')
        .filter(r.row('room').eq(roomName)).changes())
        .then(setRoomCursor)
        .then(function(){
          roomCursor.each(function (err, changes) {
            if (err) return reportError(err);
            deliverMessage(changes.new_val);
          });
        })
        .catch(reportError);
    }
  }

  socket.on('data', function(data) {
    if (data.type == 'error') return console.error(data);
    else if (data.type == 'createMessage') {
      return createMessage(data.message);
    } else if (data.type == 'joinRoom') {
      return joinRoom(data.room);
    } else {
      return reportError('Unrecognized message type ' + data.type);
    }
  });
}}

module.exports = socketAppCtor;
```

### The socket callback constructor and error handler

```javascript
var BACKLOG_LIMIT = 100;

function socketAppCtor(pool) { return function socketApp(socket) {

  function reportError(err){
    socket.write({
      type: 'error',
      err: err
    });
  }
```

`BACKLOG_LIMIT` defines how many messages we will retrieve when a user connects
to a room. As an opinionated variable, we place it outside of the rest of the
code, so that it can be more easily found and adjusted without having to parse
all of the script's functionality.

This constructor function (the return value when calling
`require('./socket.js')` in `server.js`) returns a callback function that gets
attached to the Primus server's `'connection'` event (similarly to how a
callback passed to `http.createServer`
[gets attached to the server's `'request'` event][http.createServer]). This
callback receives a socket object (referred to as a "[spark][]" in Primus's
eccentric parlance), with which we can send objects to the connected client
using the `socket.write` method, and can listen for objects sent from the
client by adding a callback to the `socket.on('data')` event.

[http.createServer]: https://nodejs.org/api/http.html#http_http_createserver_requestlistener
[spark]: http://tfwiki.net/wiki/Spark

We use this `socket.write` method to report any errors that happen on the
server to the client (where they are printed to the browser console).

### The server-side message receiver

```javascript
  function createMessage(message) {
    return pool.runQuery(r.table('messages').insert({
      body: message.body,
      room: message.room,
      author: message.author,
      sent: r.now()
    }));
  }
```

This function is called any time a chat message is sent by the connected
client, inserting it into the database, and broadcasting it to any connected
clients subscribed to messages in the room (via their changefeed cursors).
The sent time is recorded by the server when it receives the document, with
`r.now()`.

### The server-side message sender

```javascript
  function deliverMessage(message){
    socket.write({
      type: 'deliverMessage',
      message: message
    });
  }
```

This function is called when a chat message is received from the database
(both when initially joining a room and when new messages come in via
changefeed for that room). It delivers the message to the client for the
client-side JS to render as appropriate.

### Getting existing messages and subscribing to new messages

```javascript
  var roomCursor = null;

  function closeRoomCursor() {
    if (roomCursor) {
      roomCursor.close();
      roomCursor = null;
    }
  }

  function setRoomCursor(cursor) {
    return roomCursor = cursor;
  }

  function joinRoom(roomName) {
    closeRoomCursor();
    pool.runQuery(
      r.table('messages').orderBy({index:r.desc('sent')})
        .filter(r.row('room').eq(roomName))
        .limit(BACKLOG_LIMIT).orderBy('sent'))
      .then(setRoomCursor)
      .then(function(){
        roomCursor.each(function (err, message) {
          if (err) return reportError(err);
          deliverMessage(message);
        }, switchToChangefeed);
      })
      .catch(reportError);

    function switchToChangefeed() {
      closeRoomCursor();
      pool.runQuery(r.table('messages')
        .filter(r.row('room').eq(roomName)).changes())
        .then(setRoomCursor)
        .then(function(){
          roomCursor.each(function (err, changes) {
            if (err) return reportError(err);
            deliverMessage(changes.new_val);
          });
        })
        .catch(reportError);
    }
  }
```

The `joinRoom` function sends a query to the RethinkDB server to get the 100
(`.limit(BACKLOG_LIMIT)` most recent (`.orderBy({index:r.desc('sent')})`)
messages for the newly-joined room (`.filter(r.row('room').eq(roomName))`),
in their original chronological order (`.orderBy('sent')`). The cursor for this
query is saved (so it can be closed if the client switches to another room
before all messages have been delivered), then each message is delivered to the
client in order (in the first-argument callback to `roomCursor.each`).

After any existing messages have been delivered,
[the second callback to `roomCursor.each` is called][each], and the
now-exhausted cursor is replaced an active [changefeed cursor][changes] for any
new messages created for the room the client is joining.

[each]: http://rethinkdb.com/api/javascript/each/
[changes]: http://rethinkdb.com/api/javascript/changes/

### The incoming object handler

```javascript
  socket.on('data', function(data) {
    if (data.type == 'error') return console.error(data);
    else if (data.type == 'createMessage') {
      return createMessage(data.message);
    } else if (data.type == 'joinRoom') {
      return joinRoom(data.room);
    } else {
      return reportError('Unrecognized message type ' + data.type);
    }
  });
```

This callback listens to objects that are sent from the client and takes the
appropriate action (using the functions described above) for the type of event
denoted by the object's type (or, if the object's type is not recognized,
responds to the client to inform them their submission was not understood).

## Providing our client to browsers: http.js

This script provides a traditional Node.JS HTTP server app, which we use to
serve mostly static assets. For extensibility's sake, the server is defined
within a callback that takes the `pool` constructed by `./pool.js` (like
the constructor in `./socket.js` does), even though we don't actually use it
anywhere in the HTTP server as written.

```javascript
var express = require('express');

function appCtor(pool) {
  var app = express();

  app.set('trust proxy', true);
  app.set('view engine', 'jade');
  app.set('views', __dirname + '/views');
  app.use(express.static(__dirname + '/static'));

  app.get('/', function(req, res) {
    return res.render('index');
  });

  return app;
}

module.exports = appCtor;
```

Most of these lines are pretty standard for Express, and can be understood
pretty straightforwardly if you've read another tutorial about using Express,
or [its API documentation][Express API]. The [`'trust proxy'`][trust proxy]
option isn't strictly necessary for anything in this server as written, but it
serves to keep its relevant values accurate in the event that we add any
routes that use them later, as serving the app on Heroku means that the app
will be behind a proxy in the form of the [Heroku router][].

[Express API]: http://expressjs.com/api.html
[trust proxy]: http://expressjs.com/api.html#trust.proxy.options.table
[Heroku router]: https://devcenter.heroku.com/articles/http-routing

### Why use Express for our HTTP server component?

There are other ways we could serve these assets, particularly if we're going
to keep them static; however, defining it as an Express server allows us to
easily expand the HTTP server to add more dynamic components to our
statically-served pages down the line, without requiring them to burden our
real-time connections for actions that are better handled through form
submission.

## Setting up our app's client HTML structure: views/index.jade

This Jade template defines the single page of our app, which is rendered and
served to visitors when they visit the app in their browser.

```jade
doctype
html
  head
    meta(charset="UTF-8")
    title Chatroom of Requirement
    link(rel="stylesheet",href="https://cdnjs.cloudflare.com/ajax/libs/normalize/3.0.3/normalize.min.css")
    link(rel="stylesheet", href="/layout.css")
  body
    header
      input#roominput(placeholder="Your room name here")
    #board
      #messages
    form#entrybox
      input#nameinput(value="Anonymous")
      input#msginput
      button(type="submit") Send
  script(src="/primus/primus.js")
  script(src="/client.js")
```

### The head

```jade
doctype
html
  head
    meta(charset="UTF-8")
    title Chatroom of Requirement
    link(rel="stylesheet",href="https://cdnjs.cloudflare.com/ajax/libs/normalize/3.0.3/normalize.min.css")
    link(rel="stylesheet", href="/layout.css")
```

After the standard [HTML5 doctype and charset declaration][MDN HTML5 intro]
(`doctype` and `meta(charset="UTF-8")` in Jade) and our page's title, we
include [normalize.css][] via [CDNJS][], so we don't have to worry about
dealing with cross-browser CSS quirks in our own CSS. Then, we include our own
stylesheet (described below) to define the page's desired layout.

[MDN HTML5 intro]: https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/HTML5/Introduction_to_HTML5
[normalize.css]: http://necolas.github.io/normalize.css/
[CDNJS]: https://blog.cloudflare.com/cdnjs-the-fastest-javascript-repo-on-the-web/

### The body structure

```jade
  body
    header
      input#roominput(placeholder="Your room name here")
    #board
      #messages
    form#entrybox
      input#nameinput(value="Anonymous")
      input#msginput
      button(type="submit") Send
```

This includes elements for all of the inputs we'll be using in our client code,
as well as a `div` (`#messages`, which translates to
`<div id="messages"></div>` in HTML) which will contain any messages we receive
in our chat client. (The message container is contained inside another `div`
so that [the contents of the message area can be scrolled][so-21515042].)

[so-21515042]: http://stackoverflow.com/questions/21515042

### The body scripts

```jade
  script(src="/primus/primus.js")
  script(src="/client.js")
```

This requests the [Primus client library][] that Primus provides when wrapping
the HTTP server, followed by our own client script which uses the Primus
client library to make our real-time connection to the server.

[Primus client library]: https://github.com/primus/primus#client-library

## Laying out our client's UI: static/layout.css

```css
html {height: 100%;}
body {
  height: 100%;
  display: flex;
  flex-flow: column;
}
header {
  flex: none;
  display: flex;
  flex-flow: row;
}
#roominput {flex: 1;}
#board {
  flex: 1;
  display: flex;
  flex-flow: column;
  justify-content: flex-end;
}
#messages {overflow-y: auto;}
#entrybox {
  flex: none;
  display: flex;
  flex-flow: row;
}
#nameinput {width: 220px;}
#msginput {flex: 1;}
.msg-author {font-weight: bold;}
```

Our style sheet lays out each element using [Flexbox][]. Containers that list
their contents from left to right use `flex-flow: row`, while containers that
list their contents from top to bottom are specified with `flex-flow: column`.
Items with `flex: 1` grow to fit the area available to them, while items with
`flex: none` will neither grow nor shrink.

[Flexbox]: https://css-tricks.com/snippets/css/a-guide-to-flexbox/

The `#board` element has a `justify-content: flex-end` rule so that the
`#messages` child element will stick to the bottom of the main chat pane, while
the `#messages` element has an `overflow-y: auto` rule to allow it to scroll
when there are more messages than can be displayed on a single screen.

What this CSS ends up giving us, when applied to the HTML we defined above, is
a full-window chat screen, with a single long text input going across the top
for the room name, and a three-item form along the bottom, with the message
composition input taking up all remaining space between the username input (of
fixed width) and the "Send" button.

Flexbox is a *wonderful* tool in front-end styling for layout: this is
undoubtedly some of the cleanest layout CSS I've ever written, with none of
the `float: right`, `clear: all`, fixed negative margin and other hacks that
I traditionally would have had to use to get this layout.

## Writing our client's code: static/client.js

```javascript
/* global Primus */

var socket = new Primus();

function joinRoom(roomName) {
  return socket.write({type:'joinRoom', room: roomName});
}

function createMessage(message) {
  return socket.write({type:'createMessage', message: message});
}

var messageArea = document.getElementById('messages');

function deliverMessage(message){
  var messageCard = document.createElement('div');
  var messageAuthor = document.createElement('span');
  messageAuthor.className = 'msg-author';
  messageAuthor.textContent = message.author;
  messageCard.appendChild(messageAuthor);
  messageCard.appendChild(document.createTextNode(' '));
  var messageBody = document.createElement('span');
  messageBody.className = 'msg-body';
  messageBody.textContent = message.body;
  messageCard.appendChild(messageBody);

  var follow = messageArea.scrollHeight ==
    messageArea.scrollTop + messageArea.clientHeight;
  messageArea.appendChild(messageCard);
  if (follow) messageArea.scrollTop = messageArea.scrollHeight;
}

var roomInput = document.getElementById('roominput');
var msgForm = document.getElementById('entrybox');
var msgInput = document.getElementById('msginput');
var nameInput = document.getElementById('nameinput');

roomInput.addEventListener('input', function setAdHocFilter() {
  var card = messageArea.lastChild;
  while (card) {
    messageArea.removeChild(card);
    card = messageArea.lastChild;
  }
  joinRoom(roomInput.value);
});

msgForm.addEventListener('submit', function sendMessage(evt){
  createMessage({
    body: msgInput.value,
    room: roomInput.value,
    author: nameInput.value
  });
  msgInput.value = '';
  return evt.preventDefault();
});

socket.on("data", function(data) {
  if (data.type == 'error') return console.error(data);
  else if (data.type == 'deliverMessage') {
    return deliverMessage(data.message);
  } else {
    console.error('Unrecognized message type', data);
  }
});

joinRoom('');
```

The Primus client establishes a connection to the server using a variety of
mechanisms upgraded as support is detected by engine.io, indubitably.

We get elements and hook them up to events, indubitably.

## Conclusion

Using the power of RethinkDB, you can go on to extend this clasic chat model to
use more complex queries: in fact, that's the notion we're working on to create
a next-generation chat solution with [DingRoll][].

[DingRoll]: https://dingroll.com/
