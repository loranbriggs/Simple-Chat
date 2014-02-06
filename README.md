# Chat Application with Node/Express, SockJS, and Angular

*Prerequisits:*

1. [Node](http://nodejs.org/) and NPM
1. Express installed globally

    npm install -g express

## Creating the project
We will be using Node with Express as our server side framework. If express is
installed globally run:

    express --ejs SimpleChat

This will run the express generator using the [EJS](http://embeddedjs.com/)
view engine which is basicaly html that you can embed javascript into. It is
the simpliest of the view engines Express is compatable with out of the box.
This will create a directory like this:

    SimpleChat
    ├── app.js
    ├── package.json
    ├── public
    │   ├── images
    │   ├── javascripts
    │   └── stylesheets
    │       └── style.css
    ├── routes
    │   ├── index.js
    │   └── user.js
    └── views
        └── index.ejs

Open the `package.json` with a text editor and add `sockjs` as a dependency.

    {
      "name": "Simple-Chat",   // change this name if desired
      "version": "0.0.1",
      "private": true,
      "scripts": {
        "start": "node app.js"
      },
      "dependencies": {
        "express": "3.4.8",
        "sockjs": "*",              // add this line
        "ejs": "*"
      }
    }

Now from withing the `SimpleChat` directory run:

    npm install

This will install all the dependencies into the `node_modules` directory. You
can run this application now:

    node app.js

Point your browser to <http://localhost:3000/> and you should see the Express
welcome page.

Our motivation for this comes from
<http://www.gilesthomas.com/2013/02/a-super-simple-chat-app-with-angularjs-sockjs-and-node-js/>,
we are going to take the html code from there and alter it slightly. To
organize our code a bit better we are going to pull out the javascript into a
seperate file. Edit `views/index.ejs` to this:

    <!DOCTYPE html>
    <html ng-app>
      <head>
      <script src="http://cdn.sockjs.org/sockjs-0.3.min.js"></script>
      <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.11/angular.min.js"></script>

      <script src="javascripts/chat.js"></script>

      </head>

      <body>

      <div ng-controller="ChatCtrl">
        <ul>
          <li ng-repeat="message in messages track by $index">{{message}}</li>
        </ul>

        <form ng-submit="sendMessage()">
          <input type="text" ng-model="messageText" placeholder="Type your message here" />
          <input type="submit" value="Send" />
        </form>
      </div>

      </body>
    </html>

And create a file `public/javascripts/chat.js` and add the following to it:

    var sock = new SockJS('http://localhost:3000/chat');
    function ChatCtrl($scope) {
      $scope.messages = [];
      $scope.sendMessage = function() {
        sock.send($scope.messageText);
        $scope.messageText = "";
      };

      sock.onmessage = function(e) {
        $scope.messages.push(e.data);
        $scope.$apply();
      };
    }

Next we need to edit our `app.js` file so everyone connected to our chat
application can listen to one another. Again we are looking at
<http://www.gilesthomas.com/2013/02/a-super-simple-chat-app-with-angularjs-sockjs-and-node-js/>
for inspiration. Edit `app.js` to the following:

    /**
    * Module dependencies.
    */

    var express = require('express');
    var routes = require('./routes');
    var user = require('./routes/user');
    var http = require('http');
    var path = require('path');

    var app = express();

    // add the next 20 lines of code
    var sockjs = require('sockjs');

    var connections = [];

    var chat = sockjs.createServer();
    chat.on('connection', function(conn) {
        connections.push(conn);
        var number = connections.length;
        conn.write("Welcome, User " + number);
        conn.on('data', function(message) {
            for (var ii=0; ii < connections.length; ii++) {
                connections[ii].write("User " + number + " says: " + message);
            }
        });
        conn.on('close', function() {
            for (var ii=0; ii < connections.length; ii++) {
                connections[ii].write("User " + number + " has disconnected");
            }
        });
    });

    // all environments
    app.set('port', process.env.PORT || 3000);
    app.set('views', path.join(__dirname, 'views'));
    app.set('view engine', 'ejs');
    app.use(express.favicon());
    app.use(express.logger('dev'));
    app.use(express.json());
    app.use(express.urlencoded());
    app.use(express.methodOverride());
    app.use(app.router);
    app.use(express.static(path.join(__dirname, 'public')));

    // development only
    if ('development' == app.get('env')) {
      app.use(express.errorHandler());
    }

    app.get('/', routes.index);
    app.get('/users', user.list);

    // add / change the follow:
    var server = http.createServer(app).listen(app.get('port'), function(){
      console.log('Express server listening on port ' + app.get('port'));
    });
    chat.installHandlers(server, {prefix:'/chat'});

Now we have our backend communicating with our Frontend. If the server is still
running from before make sure you stop it `Ctrl - C` and run it again
`node app.js` and refresh <http://localhost:3000/>.

Now open up two browser windows for example `Chrome` and `Firefox` and talk to
yourself!
