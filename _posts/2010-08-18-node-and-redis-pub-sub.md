---
title: Node.js and Redis Pub-Sub
author: James R. Bracy
layout: post
---

[Node.js](http://nodejs.org/) is a perfect platform for creating event driven
applications. [Redis](http://code.google.com/p/redis/) and [WebSockets](http://en.wikipedia.org/wiki/WebSockets)
are great companions to Node.js. The following tutorial will walk through the
steps to build a web application that streams real time flight information
using Node.js, Redis, and WebSockets.

## Dependencies

Node.js, Redis, and a WebSocket enabled browser ([Firefox 4](http://www.mozilla.com/en-US/firefox/beta/),
[Google Chrome 4](http://www.google.com/chrome), or [Safari 5](http://www.apple.com/safari/))
are required. A tutorial covering the installation of Node.js can be found
[here](http://nodeknockout.posterous.com/countdown-to-knockout-post-1-how-to-install-n).

The easiest way to get a Redis instance would be to use [Redis To Go](http://redistogo.com/).
The free plan is all that is needed for this tutorial. If you wish to install
locally run the following commands:

    $ git clone http://github.com/antirez/redis.git
    $ cd redis/src
    $ make
    $ sudo make install
    $ cd ../..
    $ rm -rf redis

Now you can start a Redis instance locally using the `redis-server` command.

## Create the Project

Create a directory for the project. We will name the project "Flight Stream".

    $ mkdir flight-stream
    $ cd flight-stream
    
The project will require the Node.js Redis client [`redis-client`](http://github.com/fictorial/redis-node-client),
the WebSocket library [`node-websocket-server`](http://github.com/miksago/node-websocket-server),
and the MIME library [node-mime](http://github.com/bentomas/node-mime). Create
a `lib` directory and copy the libraries to this directory.

    $ mkdir lib
    $ cd lib
    $ curl -O http://github.com/fictorial/redis-node-client/raw/master/lib/redis-client.js \
           -O http://github.com/bentomas/node-mime/raw/master/mime.js \
           -O http://github.com/miksago/node-websocket-server/raw/master/lib/ws.js
    $ mkdir ws
    $ cd ws
    $ curl -O http://github.com/miksago/node-websocket-server/raw/master/lib/ws/connection.js \
           -O http://github.com/miksago/node-websocket-server/raw/master/lib/ws/manager.js
    $ cd ../..

## Create the Server

Initially the Node.js server will simply server the static index.html file that
will be create. Create the `server.js` file and add the following code:
    
    require.paths.unshift(__dirname + '/lib');

    var fs = require('fs'),
        ws = require('ws'),
        sys = require('sys'),
        url = require('url'),
        http = require('http'),
        path = require('path'),
        mime = require('mime'),
        redis = require('redis-client');
    
    var httpServer = http.createServer( function(request, response) {
        var pathname = url.parse(request.url).pathname;
        if (pathname == "/") pathname = "index.html";
        var filename = path.join(process.cwd(), 'public', pathname);

        path.exists(filename, function(exists) {
            if (!exists) {
                response.writeHead(404, {"Content-Type": "text/plain"});
                response.write("404 Not Found");
                response.end();
                return;
            }

            response.writeHead(200, {'Content-Type': mime.lookup(filename)});
            fs.createReadStream(filename, {
                'flags': 'r',
                'encoding': 'binary',
                'mode': 0666,
                'bufferSize': 4 * 1024
            }).addListener("data", function(chunk) {
                response.write(chunk, 'binary');
            }).addListener("close",function() {
                response.end();
            });
        });
    });
    

    var server = ws.createServer({}, httpServer);

    server.listen(8000);

The `httpServer` serves the static files in the `public` directory. The
`server` is what will be handling the WebSocket connections.

Create the `public` directory:

    $ mkdir public

Now copy the source for the index and stylesheets from [github](http://github.com/waratuman/flight-stream).

    $ cd public
    $ curl -O http://github.com/waratuman/flight-stream/raw/master/public/application.css \
           -O http://github.com/waratuman/flight-stream/raw/master/public/reset.css \
           -O http://github.com/waratuman/flight-stream/raw/master/public/jquery.js \
           -O http://github.com/waratuman/flight-stream/raw/master/public/text.css \
           -O http://github.com/waratuman/flight-stream/raw/master/public/index.html
    $ mkdir images
    $ cd images
    $ curl -O http://github.com/waratuman/flight-stream/raw/master/public/images/background.png \
           -O http://github.com/waratuman/flight-stream/raw/master/public/images/red.png \
           -O http://github.com/waratuman/flight-stream/raw/master/public/images/orange.png \
           -O http://github.com/waratuman/flight-stream/raw/master/public/images/green.png \
           -O http://github.com/waratuman/flight-stream/raw/master/public/images/blue.png
    
At this point you should be able to run `node server.js` and see the index
page when you got to `localhost:8000`.

Now lets get `redis-client` working in the server. The following lines
establish a connection to Redis. The `connected` and `reconnected` listeners
authenticate the connection after it has been established.

    var db = redis.createClient(9281, 'goosefish.redistogo.com');
    var dbAuth = function() { db.auth('dc64f7b818f4e3ec2e3d3d033e3e5ff4'); }
    db.addListener('connected', dbAuth);
    db.addListener('reconnected', dbAuth);
    dbAuth();

<cite style="float: right;">View the full source [here](http://github.com/waratuman/flight-stream/blob/master/server.js)</cite>
<br /><br />

Now subscribe to the `flight_stream` channel on Redis.

    db.subscribeTo("flight_stream", function(channel, message, pattern) {
        try { var flight = JSON.parse(message); }
        catch (SyntaxError) { return false; }

        if ( flight.origin.iata == "BOS" || flight.destination.iata == "BOS") {
            server.broadcast(message);
        }
    });

<cite style="float: right;">View the full source [here](http://github.com/waratuman/flight-stream/blob/master/server.js)</cite>
<br /><br />

Whenever a message is published the function passed to the `subscribeTo`
method will get called. In this case we try to parse the message as JSON then
publish it to all of the clients if the flight is leaving Boston or arriving
at Boston.

# Client

Next the client will need to be coded. I previously created the HTML for this
app, so all that we need to do is set up the WebSockets.

Open `public/application.js` and insert the following code. This code will
create a WebSocket if the browser supports it and if it does, create a
connection with the server. When a message is received, another row will be
inserted on the page displaying the flight.
    
    function delay_color(mins) {
        var min = parseInt(mins);
        if (min <= -15) return 'green';
        else if (min <= 0) return 'green';
        else if (min <= 15) return 'orange';
        else return 'red';
    }

    function delay_name(mins) {
        var min = parseInt(mins);
        if (min <= 0) return 'On-Time';
        else return 'Delayed';
    }

    var ws;
    var connect = function() {
        if (!window['WebSocket']) {
            alert("No WebSocket support.");
            return;
        }

        ws = new WebSocket('ws://' + window.location.host);
        ws.onmessage = function(evt) {
            try { var flight = JSON.parse(evt.data); }
            catch (SyntaxError) { return false; }

            $('table.flight-updates > tbody').prepend(
                '<tr>' +
                    '<td>' + flight.airline.icao + ' ' + flight.number + '</td>' +
                    '<td>' + flight.origin.iata + '</td>' +
                    '<td>' + flight.destination.iata + '</td>' +
                    '<td>' +
                        '<div class="tag ' + delay_color(flight.destination.gate_delay) + '">' +
                            delay_name(flight.destination.gate_delay) +
                            '<span>' + flight.destination.gate_delay + ' min</span>' +
                        '</div>'+
                    '</td>'+
                    '<td>' + flight.destination.gate + '</td>' +
                '</tr>');
        };
    };

    $(document).ready( function() {
        connect();
    });

The functions `delay_color` and `delay_name` are helper functions for
displaying text and selecting the right class for styling.

Now you can start the server, go to [http://localhost:8000/](http://localhost:8000/)
and you should start seeing updates roll in. If you are using your own Redis,
you are going to need to publish something to the `flight_stream`. Here is a 
sample message that you can use:

    {
        "number": "482",
        "status": "Scheduled",
        "origin": {
            "gate_delay": 0,
            "icao": "KSFO",
            "name": "San Francisco International Airport",
            "iata": "SFO",
            "gate": "3-88"
        },
        "destination": {
            "gate_delay": 0,
            "icao": "KBOS",
            "name": "Denver International Airport",
            "iata": "BOS",
            "gate": "-"
        },
        "airline": {
            "icao": "UAL",
            "name": "United Airlines",
            "iata": "UA"
        }
    }
    
If you are using the Redis instance used throughout the tutorial, you will
start seeing flight come up. I will be publishing some data every few seconds
to the instance for the coming days.

## Basic Operations

Basic operations were not really covered in this tutorial, so here are some
examples to get you started.

    db.set("foo", 1)
	db.get("foo", function(err, value) { sys.puts(value); });
	db.randomkey(function(err, key) { sys.puts(key); });
	db.hset("bar", "hi", "world");
	db.hget("bar", "hi", function(err, value) { sys.puts(value); });
	db.hgetall("bar", function(err, data) { sys.puts(data["hi"]); });