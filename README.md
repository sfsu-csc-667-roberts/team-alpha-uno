# New app setup

### Setting up our application’s directory structure

Create the directory that will hold all of our code for our project, and initialize git and npm in that directory:

```bash
mkdir team-alpha-uno
cd team-alpha-uno
git init .
npm init -y
```

(Feel free to update the `description`, `author`, and `license` fields in the resulting `package.json` file.

Next, we will create directories to distinguish between our front end javascript (the code that will be delivered to the client to be able to implement things like chat functionality, and the ability to dynamically update a page) and our back end javascript (the code that will be used to respond to client requests).

```bash
mkdir frontend
mkdir backend
```

Finally, create the entry point for our web server (this is just an empty file for now; we will add some code shortly):

```bash
touch server.js
```

Finally, we want to ensure that we do not commit certain files into our repository (`node_modules/` because they can be installed using `npm install`, and `.env` because it will hold strings that we do not want to make public, and that will be used only for local development):

```bash
touch .gitignore
```

In the `.gitignore` file, add the following:

```
node_modules/*
.env
```

### Create a basic Express server

We will be using [https://expressjs.com/](https://expressjs.com/) to implement the backend code for our term project. Express provides an API that allows us to write javascript code to respond to HTTP requests, as well as providing the ability to host static files, and to dynamically generate HTML responses to send to the client. We will be discussing _how_ to do this as we continue to work on our projects, but for now, we need to include the `express` dependency in our project:

```bash
npm install express
```

Now, we can create a _minimal_ server by adding the following code to `server.js`:

```jsx
const express = require("express");
const app = express();
const PORT = process.env.PORT || 3000;

app.get("/", (request, response) => {
  response.send("Hello World!");
});

app.listen(PORT, () => {
  console.log(`Server started on port ${PORT}`);
});
```

This code requires (another way of saying includes or imports) the `express` function (that is exported from the `express` package) into `server.js`, then uses the function to create an instance of an express application. (See [https://expressjs.com/en/5x/api.html#app](https://expressjs.com/en/5x/api.html#app) for the express application API.) We then set up a “route” - this defines an HTTP verb (in this case, `get`), and a URL (in this case, the root of our site: `"/"`) that the express server will monitor for requests. When an HTTP request is received that matches this verb and URL, the express application will invoke the “handler” function we define, passing in `request` and `response` objects that we can use to create a response. In this case, we use the `response` object’s `send` method to respond with the text, “Hello World!”.

We can run this using the `node` javascript runtime:

```bash
node server.js
```

We can verify it is working by visiting [http://localhost:3000](http://localhost:3000), where we should see the text that we used the `response` object to send to the client (”Hello World!”).

### Organizing the server code

Eventually, we will be adding quite a few additional routes to the server, and we want to avoid creating a single, massive, hard to main server file. One tool that express provides us with is the `Router` ”middleware” (more on this later) that allows us to create individual routes in modules (separate files), and then “mount” those routes in our main application instance.

Create a directory where our route logic will go, and an initial route file for our root routes:

```bash
mkdir backend/routes
touch backend/routes/root.js
```

As we add additional functionality like authentication and authorization, game logic, chat logic, etc., we can break up the routes into easy to understand and maintain files in this new directory.

Add the following code to import the express `Route` middleware, create the route we previously defined in `server.js`, and then export that route so that another module can import (require) it:

```jsx
const express = require("express");
const router = express.Router();

router.get("/", (request, response) => {
  response.send("Hello world from within a route!");
});

module.exports = router;
```

Now, we can refactor the server to include the following code:

```jsx
const express = require("express");
const app = express();
const PORT = process.env.PORT || 3000;

const rootRoutes = require("./backend/routes/root");

app.use("/", rootRoutes);

app.listen(PORT, () => {
  console.log(`Server started on port ${PORT}`);
});
```

This imports the routes that were exported from the `root.js` file, and then “mounts” all of the routes defined in that `Router` under the `"/"` URL (appends the URLs from the route to the root URL).

If you left the server running from the previous section and refresh the [http://localhost:3000](http://localhost:3000) page, you won’t see a change! In order for node to load the change, it needs to reload the file. To do this stop the server (`Ctrl-C` on a \*nix system), and start it again using the `node server.js` command. (This is tedious, and we will work on a way to automate this shortly.)

Visiting [http://localhost:3000](http://localhost:3000) should now show us the updated content.

### Simplify startup

We can use the `package.json` file to help simplify start up by adding a “script” entry that runs the startup command for us. (We will be adding additional scripts here as we continue development.) In `package.json`, in the `scripts` section, add the following entry:

```json
{
  "scripts": {
    "start": "node ./server.js"
  }
}
```

Now we can use the following command to start the server:

```bash
npm run start
```

This runs the `node server.js` command, starting the server as we have seen in previous steps.

Now, we can add another `scripts` entry to help reload the server as we make changes, so that we do not have to manually stop and start the server whenever a file is updated in order to see that change reflected in our site. To do this, we will use another library, `nodemon`, that will automatically restart the node process whenever a change is detected in a directory.

First, install `nodemon`:

```bash
npm install nodemon --save-dev
```

Note that we will _only_ be using `nodemon` in our development environments - we do not want the server to reload in production if a file changes! For this reason, we use the `--save-dev` flag when installing `nodemon` to tell npm that the module should only be installed in a development environment.

Now, we can add a new script to `package.json`:

```json
{
  "scripts": {
    "start:dev": "nodemon --watch backend ./server.js"
  }
}
```

We create this separate script so that we can add developer tools when we are programming, and so that we can omit those tools in the production environment (that’s why we keep the `start` script).

Now start the server:

```bash
npm run start:dev
```

We don’t see anything different since we haven’t made any changes, but now we can make a change to our route file (for example, updating the text), and just press the refresh button in the browser to see those changes reflected (without having to stop and start the server). If you look at the shell, you will see that `nodemon` causes the server to reload when it detected the file change.

### Adding error handling to the server

With the current implementation, if we try to visit a URL (route) that the express application does not understand, it responds with a simply text message. For example, with your server running, browse to [http://localhost:3000/nothing](http://localhost:3000/nothing). You will see the message:

```
Cannot GET /nothing
```

In order to create a better and more consistent user experience when an error is encountered, we will use the `http-errors` library:

```bash
npm install http-errors
```

Add this into the `server.js` file by importing the library at the top of the file, and then adding a middleware function _at the bottom of the file_. Adding this function at the bottom of the file is important - express attempts to match routes in the order they are defined, so if a valid route is defined, we want express to use that route logic instead of executing this function. Placing the function at the bottom of the file means it should only be reached (and therefore executed) if the requested URL does not get matched to any route.

```jsx
const createError = require("http-errors");

/** Existing server.js content **/

app.use((request, response, next) => {
  next(createError(404));
});
```

Visiting the page, we now see a very noisy (and informative) error message being displayed.

### So what is middleware?

Middleware is a fancy term for “a function that will always be executed by the express application” (following the order rules discussed above). These functions have access to the `request` and `response` objects, as well as a special function named `next` that tells express to execute the next middleware.

The routes that we’ve seen to this point are just a special type of middleware - functions that have access to the `request` and `response` objects, and that only executes for a _specific path (or URL)_. The routes we have written also ends the request-response cycle, so they have not needed to use the `next` function to continue executing middleware; after we respond to a client request, there’s nothing else we want to do.

Since we will likely be writing additional middleware to support our project, lets create a `middleware` directory in our `backend` folder, and create a simple middleware to exercise our understanding:

```bash
mkdir backend/middleware
touch backend/middleware/request-time.js
```

In `request-time.js`, add the code:

```jsx
const requestTime = (request, response, next) => {
  console.log(`Request received at ${Date.now()}: ${request.method}`);
  next();
};

module.exports = requestTime;
```

Now, tell the express application to use this middleware in `server.js` (remember that the placement of this is important!):

```jsx
const createError = require("http-errors");
const requestTime = require("./backend/middleware/request-time");

const express = require("express");
const app = express();
app.use(requestTime);

const PORT = process.env.PORT || 3000;

/* Rest of server.js */
```

Since this middleware was placed before all other middleware (including routes and the error handler), it always gets executed, printing out a timestamp and the HTTP verb used in the HTTP request sent to the express application.

Now that we know a little more about middleware, we can remove this from `server.js`.

### Serving static files

Sometimes, we want to serve a static file - one that is not created by javascript logic in a route (images, stylesheets, etc.). To configure our express application to do this, create a directory to hold our static assets:

```bash
mkdir backend/static
```

Now, set up this directory so that express knows to serve static files from it. In `server.js`, add the following:

```jsx
const path = require("path");
const createError = require("http-errors");

const express = require("express");
const app = express();

const PORT = process.env.PORT || 3000;

app.use(express.static(path.join(__dirname, "backend", "static")));

const rootRoutes = require("./backend/routes/root");

/* Rest of server.js */
```

To test this functionality, I added a `favicon.ico` file in the `static` directory, and used chrome (which annoyingly always asks for a favicon file, but its useful in this instance). When I refresh the root URL of my application, I can see in the network tab that the favicon is being returned, and that the route is still loading.

### Creating content

We can use `response.send()` to send html content to our client, for example:

```jsx
router.get("/", (request, response) => {
  const name = "person";

  response.send(
    `<html><head><title>Hello</title><body><p>Hello ${name} html!</p></html>`
  );
});
```

As you can imagine, it could get really tedious generating these html strings by hand, especially when we want to insert dynamic information into the html (like the variable `name`, above). Express allows us to integrate a “template engine” that removes the tedium from dynamically generating html. A templating engine allows us to specify a template, provide it with values we want to plug into that template (sometimes called “locals”), and then uses the template to generate the HTML.

There are a few different template engines supported by express. This branch uses `ejs`. Feel free to research different template engines that are supported by express if `ejs` isn’t your cup of tea: [https://expressjs.com/en/resources/template-engines.html](https://expressjs.com/en/resources/template-engines.html).

First, install the template engine library:

```bash
npm install ejs
```

In `server.js`, tell the express application to use the template engine (doing this after immediately after the application instance is created so that the engine is available for us in any route), and _where_ to find the templates:

```jsx
const app = express();

app.set("views", path.join(__dirname, "backend", "views"));
app.set("view engine", "ejs");
app.use(express.static(path.join(__dirname, "backend", "static")));
```

Create the directory to store our templates (also called views), and our first template file:

```bash
mkdir backend/views
touch backend/views/site-header.ejs
touch backend/views/home.ejs
touch backend/views/site-footer.ejs
```

The `site-header.ejs` and `site-footer.ejs` files are used to create the content we want for _every page_, and allows us to reuse common html around page-specific content (in this case defined in `home.ejs`).

First, in `site-header.ejs`, add the following:

```html
<html>
  <head>
    <title><%= title %></title>
    <link rel="stylesheet" href="/stylesheets/home.css" />
  </head>
  <body></body>
</html>
```

In `site-footer.ejs`, add the following:

```html
  </body>
</html>
```

In `home.ejs`, add:

```html
<%- include("site-header", { title: title }) %>
<h1><%= title %></h1>

<p><%= message %></p>
<%- include("site-footer") %>
```

Return to our root route, and change `response.send` to:

```jsx
response.render("home", {
  title: "Hi World!",
  message: "Our first template.",
});
```

In `site-header.ejs` and `site-footer.ejs`, the html skeleton is defined for our html page. `home.ejs` _includes_ these layout files to set up the html skeleton, and defines content in between the `include`s. The `<%= %>` syntax means to output the value of a local (variable passed in to the template), and the `<%- %>` syntax means to output the included content _unescaped_ (not removing special characters that could be interpreted as HTML, since we want this to be interpreted as HTML). Notice that in the `site-header.ejs` file, I added a stylesheet. We will discuss stylesheets and CSS later in the semester; for now, create a directory `stylesheets` in our `static` directory:

```bash
mkdir backend/static/stylesheets
```

add a file named `home.css`, and add:

```css
body {
  background-color: rgba(0, 0, 255, 0.1);
}
```

Switching `response.send` to `response.render` tells the express application to find a template, in this case `home`, and to provide the “locals” `title` and `message` to be used to fill out the template. Refresh the root page of our app, and you should see an html page with the dynamic content that was specified. Updating the variable values in the route will update the information displayed in the html.

### Setting up the frontend

Eventually, we will be adding code specifically to handle the front end logic for our game (this will go in the `frontend` directory). In our front end code, we want to be able to utilize modern javascript, and to be able to organize our code into discrete modules (individual files) to make it easier to maintain. Unfortunately, the browser is not able to find this code unless we manually insert `<link>` and `<script>` tags for all of the different files.

To avoid this manual process, we will use a modern build tool, `webpack` in order to aggregate all of our files into a single file, and place that file in the `public/` folder we created earlier. This way, we only have to add one `<script>` tag to our site, and that will include all of the front end code we have written.

We need a few libraries to do this:

```bash
 npm install --save-dev webpack webpack-cli babel-loader
```

Webpack can be run manually, passing in configuration flags, or we can specify configuration in a special file named `webpack.config.js` at the root of our project:

```bash
touch webpack.config.js
```

In this file, we will define the configuration that the `webpack` tool will use to determine how to build our javascript, where to find the javascript we want built, and where to place the final build product. Add this code to the `webpack.config.js` file:

```jsx
const path = require("path");

module.exports = {
  entry: "./frontend/index.js",
  output: {
    path: path.join(__dirname, "backend", "static", "scripts"),
    publicPath: "/backend/static/scripts",
    filename: "bundle.js",
  },
  mode: "production",
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: { loader: "babel-loader" },
      },
    ],
  },
};
```

Here, we’re telling `webpack` to start at `./frontend/index.js`, and to create a bundle of all of the javascript referenced from that file and all files it references (i.e. traversing the dependency tree of `import` statements, and ensuring that all code that is `import`ed is added to resulting build artifact). The `module` `rules` section tells `webpack` to handle any file that ends with the extension `.js`. (We can use additional loaders with different extensions; for example if we wanted to use typescript - a strongly typed superset of javascript - we could test for the `.ts` extension, and use an appropriate loader to convert that into javascript that the browser could understand.) Finally, we tell `webpack` to output the resulting file in our `backend/static/scripts` directory, naming it `bundle.js`.

With this configuration done, we will add another script into the `scripts` section of our `package.json` file in order to automate this build process. (The `build` script will be used to ensure a new package is created for production every time we deploy, and the `build:dev` script will be used while we are developing, to watch for changes, and re-run `webpack` when a change is detected.)

```json
{
  "scripts": {
    "build": "webpack",
    "build:dev": "webpack --watch"
  }
}
```

Let’s add the entry file into the `frontend` directory:

```bash
touch frontend/index.js
```

And add some basic code so we can check that this works as expected:

```jsx
console.log("Hello from a bundled asset.");
```

At the command line, use the `build` script:

```bash
npm run build
```

A new folder named `scripts` will be created in `/backend/static` containing the file `bundle.js`.

We will include this in the `site-header.ejs` file we created earlier by adding the following `script` tag underneath the `link` tag:

```html
<script src="/scripts/bungle.js"></script>
```

Make sure your server is running (`npm run start:dev`), and point your browser at [http://localhost:3000](http://localhost:3000) . Check the console in the browser, and you should now see the `console.log` statement we added in the `frontend/index.js` file.

### Development is hard

With our current scripts, we would have to run both `npm run start:dev` and `npm run build:dev` in order to have our front end code changes automatically bundled, and to have our back end changes cause the server to reload. In addition, when either of these changes happen, we have to manually reload the browser. Lets make it easier to manage all of this in our development environment with a few more changes to our project.

First, lets add some libraries to help:

```bash
npm install --save-dev livereload connect-livereload
```

We need to integrate these with our server, but we only want this to happen while we’re in the development environment. Change the `start:dev` script to add an environment variable named `NODE_ENV`:

```json
{
  "scripts": {
    "start:dev": "NODE_ENV=development nodemon --watch backend ./server.js"
  }
}
```

We will use this environment variable in our code to only run development code if this variable’s value is `development` (we will set the value to `production` when we look at how to deploy our application). To use this environment variable, update `server.js` to include the following, right after we create the `app` instance:

```jsx
if (process.env.NODE_ENV === "development") {
  const livereload = require("livereload");
  const connectLiveReload = require("connect-livereload");

  const liveReloadServer = livereload.createServer();
  liveReloadServer.watch(path.join(__dirname, "backend", "static"));
  liveReloadServer.server.once("connection", () => {
    setTimeout(() => {
      liveReloadServer.refresh("/");
    }, 100);
  });

  app.use(connectLiveReload());
}
```

We also want to reload the server whenever we make a change to an `.hbs` file, and since `livereload` is monitoring the `static` folder, we can omit that from `nodemon`'s set of files. Create a configuration file for `nodemon`:

```bash
touch nodemon.json
```

In this file, add the following:

```json
{
  "ext": "js,ejs",
  "ignore": ["backend/static"],
  "watch": "backend"
}
```

Now we want to run _both_ the `nodemon` process and the `webpack` process concurrently:

```bash
npm install --save-dev concurrently
```

We will change the `start:dev` script to use `concurrently`, and move the existing `start:dev` script to `server:dev` in `package.json`, removing the configuration flags from the `nodemon` command:

```json
{
  "scripts": {
    "start:dev": "concurrently \"npm:server:dev\" \"npm:build:dev\"",
    "server:dev": "NODE_ENV=development nodemon ./server.js"
  }
}
```

### Final touches

To facilitate development (and eventually debugging), we will add a logging library called `morgan` to our application. Install the dependency:

```bash
npm install morgan
```

And tell the express application about it in `server.js` (immediately after the application object is created):

```jsx
const express = require("express");
const app = express();

app.use(morgan("dev"));
```

Additional information will now be displayed in the shell as requests are made:

```
[nodemon] starting `node ./server.js`
Server started on port 3000
GET / 304 175.073 ms - -
GET /stylesheets/home.css 304 1.138 ms - -
GET /favicon.ico 304 0.500 ms - -
```

We want to add some additional utility to our server, namely the ability to support url encoded request bodies, json request bodies, and cookies. Supporting cookies requires a library, `cookie-parser`:

```bash
npm install cookie-parser
```

JSON and url encoded bodies are supported by express, with some minor setup. To setup all three of these, I made some additions to `server.json`, with the final version of my file looking like this:

```jsx
const path = require("path");
const createError = require("http-errors");
const cookieParser = require("cookie-parser");
const morgan = require("morgan");

const express = require("express");
const app = express();

app.use(morgan("dev"));
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());
app.set("views", path.join(__dirname, "backend", "views"));
app.set("view engine", "ejs");
app.use(express.static(path.join(__dirname, "backend", "static")));

const rootRoutes = require("./backend/routes/root");
const { urlencoded } = require("express");
app.use("/", rootRoutes);

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server started on port ${PORT}`);
});

app.use((request, response, next) => {
  next(createError(404));
});
```

### Keeping code consistent

It can be difficult to keep code consistent when working on a team of multiple developers. For that reason, we are going to automate code formatting, and use some tools to apply the code formatting every time we commit a change. First install the libraries that will manage this for us (since we only use these during development, we are using the `--save-dev` flag):

```bash
npm install --save-dev husky lint-staged
npm pkg set scripts.prepare="husky install"
npm run prepare
npx husky add ./husky/pre-commit "npx lint-staged"
# Make sure the hook is included in your repo
git add .husky/pre-commit
```

Then, add the following to `package.json` (the outer braces in this snippet are the outer braces in `package.json`):

```json
{
  "lint-staged": {
    "**/*": "prettier --write --ignore-unknown"
  }
}
```

Now that we have some automation happening on every commit, we can also use it to ensure that our frontend code gets built for the production environment. One way to do this is to just have `webpacke` run on every commit. We can update the `lint-staged` entry in `package.json` to do this for us:

```json
{
  "lint-staged": {
    "**/*": "prettier --write --ignore-unknown",
    "frontend/**/*.js": "webpack"
  }
}
```
