# User Authentication: Backend

## Resources

- [Passport Docs](http://www.passportjs.org/docs/)
- [Example using pg-promise](https://github.com/joinpursuit/Pursuit-Core-Web-Passport-Auth) 
  - checkout the solution branch for a completed, working example. Start from the master branch to just have boilerplate.

## Introduction

User registration and authentication is fairly difficult to wrap your head around. Thankfully, **it can stay that way** for now. If you want to do Serious Backend Security Development stuff, you can go super deep into these concepts, but for most of us, we just need to have the **boilerplate available** and know how to use it.

That's not to say this won't be a test of your core JavaScript and Express skills, because it absolutely will be. We just won't be going into the nitty gritty of the precise JavaScript syntax, because there's a lot of stuff we're going to set up to configure authentication that makes more sense to our NPM packages than it does to us.

Let's take a look at the provided example above, starting with the root-level `package.json`:

## New Modules

```js
"dependencies": {
  "bcrypt": "^3.0.7", //new
  "cookie-parser": "~1.4.4",
  "debug": "~2.6.9",
  "express": "~4.16.1",
  "express-session": "^1.17.0",
  "morgan": "~1.9.1",
  "passport": "^0.4.1", // new
  "passport-local": "^1.0.0", // new 
  "pg-promise": "^10.4.0" 
},
"devDependencies": {
  "nodemon": "^2.0.2"
}
```

New ones include `bcryptjs`, `express-session`, `passport`, and `passport-local`. The first one, a JavaScript-y extension of bcrypt, provides the functionality to put user passwords through a hashing function and salt them for our database. This is important for our users' security, because we don't want to store passwords in "plaintext".

The other three are all about user authentication.

## Concepts time!

### Passport Strategies

- `passport` is a node module that creates some nice abstractions around the authentication process for us. It contains the logic for creating and processing _sessions_. You know when you leave a website, return, and are still logged in? Yep, that's because there's a _session token_ containing part of your unique user information. Passport checks this piece of information against your stored information on the backend. If everything is good, you stay logged in and can fetch your private information from protected endpoints. Passport is written in a way that allows it to be plugged into most any express application, and because of that, we have to tell it what we want it to do at certain times, which is where the `strategy` concept comes in.

- `passport-local` is the specific strategy we're applying to our Passport authentication. There are many different strategies you can employ to manage your sessions. For example, the `passport-google` package lets you authenticate to other sites using your Google login. _Local_ means that we're handling everything in-house (as opposed to using another service, like Firebase) - we're managing usernames and passwords, we're giving session tokens, and we're processing registration and authentication.

- `express-session` works with `passport` to help Express recognize and process session tokens. It's a little piece of middleware that we need in order to properly process these objects. Sessions are stored in-memory in express and "signed" cryptographically so that hackers can't send requests pretending to be someone they're not.

- `bcrypt` contains methods that perform encryption and hashing. Hashing is the process of turning an input string (like a password) into a longer, unreadable string. This process is mathematically _deterministic_, which means that given the same input string, it will produce the same result every time. It also can't be reversed - so you can't take a hashed password and un-hash it. You can only check to see if your input string matches the hash.

- **Serializing** processes a user token into plain text, which is how it can be assigned to our request header. In this case, we're putting our username onto our request header to represent that user's session.
- **Deserializing** takes a plain text request header, converts it into a JavaScript-readable format, and checks our database to make sure that user actually exists. This accomplishes two things: It lets us process a session token, and it makes sure a hacker isn't throwing together a request header without an actual user account to back it up.


## More code structure: `auth/`

In this folder, we have two files:

```
-helpers.js
-passport.js
```

- `helpers.js` is where we use `bcrypt` to create and verify users. We also create a piece of middleware here, `loginRequired`, that makes sure a session token is on our request header. We'll use this in routes that we want a user to be logged in to access.
- `passport.js` is where we set up how passport will _serialize_ and _deserialize_ users.
- `local.js` is where we define our `strategy` for passport.

> Note that this project structure isn't required. Express and passport don't care where each of these pieces live, as long as they're all there.

## Coding time

### `helpers.js`

Let's add a couple functions here that we'll use elsewhere.

```js
const bcrypt = require('bcrypt');

const hashPassword = async (plainPassword) => {
  try {
    let passwordDigest = await bcrypt.hash(plainPassword, 10);
    return passwordDigest;
  }
  catch (err) {
    throw (err)
  }
}

const comparePasswords = (plainPassword, passwordDigest) => {
  return bcrypt.compare(plainPassword, passwordDigest)
}

// ... other helper functions
```

We define two functions here - `comparePasswords`, which (as the name implies) hashes its first (plain) argument and compares it to the second (hashed) argument. We have to hash instead of decrypt because **it is impossible** to decrypt hashes.

The second function does what its name implies - creates a hash from a "plaintext" password input, then returns it. We'll use this function to create hashes that get stored in the database.

We'll also add a third function:

```js
const loginRequired = (req, res, next) => {
  if (req.user) return next(); // If the user is logged in just call the next middleware
  res.status(401).json({
    payload: {
      message: "Unauthorized - To Access this route you have to be logged in."
    },
    error: true
  })
}
```

`loginRequired` is a little bit different. It's a piece of middleware that we add to routes to make sure there's a user token in the request. We utilize it in our routing like this (this snippet is from the `users.js` file in the routes folder):

```js
// routes/users.js
router.get("/", loginRequired, db.getUsers)
router.post("/logout", loginRequired, db.logoutUser);
```

Because after all, you have to login to log out!

### `passport.js`

This file we configure Passport. It adds the logic to `serialize` and `deserialize` the user to/from the session. Here we also setup the passport strategy to use. The strategy will be run every time a user logs in. 

```js
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;
const { comparePasswords } = require('../auth/helpers');
const usersQueries = require('../db/queries/users');

passport.use(new LocalStrategy(async (username, password, done) => {
  try {
    const user = await usersQueries.getUserByUsername(username);
    if (!user) {
      return done(null, false)
    }

    const passMatch = await comparePasswords(password, user.password_digest);
    if (!passMatch) {
      return done(null, false)
    }

    delete user.password_digest; // Delete password_diggest from user object to not expose it accidentally
    done(null, user);

  } catch (err) {
    done(err)
  }
}))

passport.serializeUser((user, done) => {
  done(null, user)
})

passport.deserializeUser(async (user, done) => {
  try {
    let retrievedUser = await usersQueries.getUserByUsername(user.username)
    delete retrievedUser.password_digest;
    done(null, retrievedUser)
  } catch (err) {
    done(err, false)
  }
})

module.exports = passport;
```

When a user provides a username and password in the request body, and the passport strategy looks in our database for that user based on his/her username hence the `userQueries.getUserByUsername()` above. If the user is found in the database, it then checks to see if the password is correct. If both the username and the password are correct, it returns our user object. Otherwise, it returns `false`, which tells passport that this isn't the correct combination.

### `app.js` - Ensuring that authentication occurs

We add several bits to app.js to ensure that requests to the database are properly authenticated:

```js
const session = require("express-session");
const passport = require("passport");

// ... other routers
const authRouter = require('./routes/auth');

// ... other middleware stuff
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));
app.use(cookieParser("NOT_A_GOOD_SECRET"));

app.use(
  session({
    secret: "NOT_A_GOOD_SECRET",
    resave: false,
    saveUninitialized: true
  })
);
app.use(passport.initialize());
app.use(passport.session());

app.use('/users', usersRouter);
app.use('/auth', authRouter);
// ... other endpoints and routers setup below
```

`session` here is used in two different contexts - `session` by itself represents our instance of `express-session`, whereas `passport.session` is passport's processing of that session token.

**Note:** Our session token stores a hash, too, and hashing functions often require a seed to get started. The `secret` part is that seed, and could be anything. It's not generally best-practice to shove it right in there - when we deploy to production, we'll want to hide it in a similar way that we'd hide our API keys, using environment variables.

### `queries/userQueries.js` - Creating/Signing-Up a user

In `routes/auth.js` we have the following;

```js
const express = require('express');
const router = express.Router();
const usersQueries = require('../db/queries/users');
const passport = require('../auth/passport');

router.post('/signup', async (req, res, next) => {
  try {
    const user = req.body;
    const newUser = await usersQueries.createUser(user)
    res.send({
      payload: newUser,
      msg: "New user signup success",
      err: false
    })
  } catch (err) {
    next(err)
  }
});
```

`userQueries.createUser` comes from `db/queries/userQueries.js`, we import some auth files and use them to rewrite our create function, creating a function called `createUser`:

```js
// db/queries/userQueries.js
const db = require('../db')
const authHelpers = require("../../auth/helpers");

const createUser = async (user) => {
  const passwordDigest = await authHelpers.hashPassword(user.password);

  const insertUserQuery = `
      INSERT INTO users (username, password_digest) 
        VALUES ($/username/, $/password/)
        RETURNING *
    `

  const newUser = await db.one(insertUserQuery, {
    username: user.username,
    password: passwordDigest
  })

  delete newUser.password_digest // Do not return the password_digest and accidentally expose it
  return newUser
}

// ... other user queries
```

The order here is important:

- First, we use our `authHelpers.hashPassword` method to process and hash the user's password.
- Then, we add our user to the database.
- Finally, we return a message to indicate that we've successfully registered the user.

Once a user signs up the client can send another request to the backend to log our user in. Lets look at what the `/login` endpoint will look like:

```js
// routes/auth.js
router.post("/login", passport.authenticate("local"), (req, res) => {
  // If this ever runs it's because authentication went well
  res.json({
    payload: req.user,
    msg: "User login success",
    err: false
  })
});
```

Note here how we are handling the Passport stuff in the route as middleware _before it even gets_ to our route handler function. 

This tells passport to use the local strategy we defined in the `passport.js` file. Luckily, we have a function `logout` on the request that makes logging out a breeze, even without middleware:

```js
//  routes/auth.js
router.post('/logout', authHelpers.loginRequired, (req, res, next) => {
  req.logout();
  res.json({
    msg: "User logout success",
    err: false
  })
```
Recall our `loginRequired` in `auth/helpers.js`. Here it is used.

Finally, we want to create a method that shows all the registered users in the database. This is a simple query.

```js
// queries/users.js
// ... other user queries
const getAllUsers = async () => {
  const users = await db.any("SELECT * FROM users")
  return users;
}

// ... exports
```

And now we can protect our endpoint like so
```js
// routes/users.js

// ... other routes and handlers

router.get('/', authHelpers.loginRequired, async (req, res, next) => {
  try {
    const users = await usersQueries.getAllUsers()
    res.send({
      payload: users,
      msg: "Retrieved all users",
      err: false
    })
  } catch (err) {
    next(err)
  }
});


module.exports = router;
```

## Putting it all together

Here's the general process of what needs to happen for a user to be authenticated. We haven't touched the front-end yet but we can just use our imaginations for now.

### New User Registration

* React - load the registration form
* React - POST the form with username and password as JSON
* Express - receive the POST request and call db.createUser
* Express - hash the password, then insert the username and hashed password into psql
* Express - Send response confirmation
* React - Receive response and notify user of success

### User logging in

* React - load the login form
* React - POST the form with username and password as JSON
* Express - Receive the POST request and forward to passport middleware
* Passport - call passport.authenticate which looks up the user in psql
* Passport - if username is found
  * hash the incoming password against the stored password
    * if password matches, create session and send response
    * if password doesn't match, send response as unauthorized
* Passport - if username not found, send response as unauthorized
* React - receive response, create localstorage token
* React - send GET request to express to verify user is logged in
* Express - receive GET request and verify session exists by deserializing user
  * If session exists, serialize user & respond with username
  * If session does not exist, respond with `null`
* React - receive response
  * if username matches session token, setState with username & loggedIn: true
  * if username does not match session token, delete localstorage token & send request to de-authenticate express session

### User logging out

* React - send request to log user out 
* Express - receive request, delete session token, send response confirmation
* React - receive response, delete localstorage token
* React - send GET request to verify user is logged in
  * see above for remaining steps
