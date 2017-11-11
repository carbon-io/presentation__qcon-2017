# QCon, 2017.11.13, San Francisco CA

## (0*) Hello

* I'm Will Shulman, CEO and Co-founder of mLab.
* Database-as-a-Service for MongoDB that runs on AWS, Azure, and Google Cloud.
* We host more than half a million MongoDB deployments.
* Our infrastructure is large and complex, and is made up of dozens of types of microservices.
* We built Carbon.io to allow us to quickly build high-quality microservices.

## (1*) What is Carbon.io?

* A Node.js framework for building microservices, and APIs.
* It is a framework, built on a set of core libraries.
* It is opinionated *(but also friendly)*.

## (2*) Design goals

* To assist in the *coding*, *testing*, and *documenting* of micro-services.
* To come with *"batteries included"* with a *"the way"* to do common things such as structuring your application code, parameter parsing, logging, authentication and access control, etc...
* To simplify concurrency (a.k.a. avoid callback hell) and make it easier to debug code by bringing back stack traces.
* Database CRUD should be trivial.
* Communicating with other services should be trivial (so you can easily build distributed systems).

## (3*) The Basics

### (3.1) Services and Endpoints

In carbon.io the top-level application is called a *Service*.

* A Service is a tree of *Endpoints*
* Endpoints are a set of *Operations* (GET, PUT, POST, DELETE, etc...)

#### (3.1.1) Hello world!

```node
var carbon = require('carbon-io')
var __     = carbon.fibers.__(module)
var o      = carbon.atom.o(module).main

__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    port: 8888,
    endpoints : {
      hello: o({
        _type: carbon.carbond.Endpoint,
        get: function(req, res) {
          return { msg: "Hello world!" }
        }
      })
    }
  })
})
```

* [Source code][10]

### (3.2) Operations - defining parameters and responses

Operations can be decorated with structure that allows the system to automatically handle certain aspects of managing
inputs and outputs, and makes the API self-describing.

* Operations can formally define parameters (*query*, *header*, *path*, and *body* parameters).
* Operations can formally define responses by HTTP status code.
* Parameter and response definitions can define JSON Schemas that are automatically enforced.
* This is useful for:
  * Automatic validation of input parameters and response output
  * Generating API documentation

#### (3.2.1) Hello world! again

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    port: 8888,
    endpoints : {
      hello: o({
        _type: carbon.carbond.Endpoint,

        get: {
          parameters: {
            who: {
              location: 'query', // one of 'path', 'query', 'header', or 'body'
              required: false,
              default: 'world',
              schema: { type: 'string' } // drives parsing and validation (which can also help prevent injection attacks)
            }
          },
          responses: [
            {
              statusCode: 200,
              description: "Success",
              schema: {
                type: 'object',
                properties: {
                  msg: { type: 'string' }
                },
                required: [ 'msg' ],
                additionalProperties: false
              }
            }
          ],

          service: function(req, res) {
            return { msg: `Hello ${req.parameters.who}!` }
          }
        }
      })
    }
  })
})
```

* Let's take a look at how it is all packaged: [Hello world (parameter parsing) example][1].

### (3.2) An aside on ```__```, ```o```, and ```_o```

Carbon.io has three *magical* functions that are used throughout Carbon.io apps, ```__```, ```o```, and ```_o```.

You will often see Carbon.io modules follow this general pattern:

```node
var carbon = require('carbon-io')
var __     = carbon.fibers.__(module)
var _o     = carbon.bond._o(module)
var o      = carbon.atom.o(module).main  // The .main variant is used for top-level modules that define 'main' entry points

__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    .
    .
    .
  })
})
```

* ```__``` creates new *Fibers* (which will be covered in detail in section (7) below). Fibers are like light-weight threads in Node.js that allow you to call async function asynchronously (similar to but more powerful that ```async```/```await```).

* ```o``` is Carbon.io's universal object factory, used for instantiating classes and injecting property values. 

* ```_o``` is Carbon.io's univesal name resolver. Can be used to resolve Node.js modules, environment variables, web addresses, etc...


## (4*) Database CRUD

While one can use Carbon.io with any database technology, Carbon.io makes it particularly easy to work with MongoDB.

### (4.1) Accessing a MongoDB database

Included with Carbon.io is a MongoDB driver-wrapper called Leafnode.

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    port: 8888,
    dbUri: "mongodb://localhost:27017/mydb",
    endpoints: {
      messages: o({
        _type: carbon.carbond.Endpoint,
        post: function(req) {
          return this.getService().db.getCollection('messages').insert(req.body)
        },
        get: function(req) {
          return this.getService().db.getCollection('messages').find().toArray()
        }
      })
    }
  })
})
```

### (4.2) Collections

Collections are an abstraction on top of ```Endpoint```s that provide a higher-level interface for implementing
access to a collection of resources.

Common use case:
```
GET     /users       // Get all Users
POST    /users       // Add a User
GET     /users/123   // Get User with _id of 123
PUT     /users/123   // Modify User with _id of 123
DELETE  /users/123   // Remove User with _id of 123
```

#### (4.2.1) The Collection interface

When implementing a ```Collection```, instead of implementing the low-level HTTP methods
(```get```, ```post```, ```delete```, etc...), you implement the following higher-level interface:

* ```insert(obj, reqCtx)```
* ```find(query, reqCtx)```
* ```update(query, update, reqCtx)```
* ```remove(query, reqCtx)```
* ```saveObject(obj, reqCtx)```
* ```findObject(id, reqCtx)```
* ```updateObject(id, update, reqCtx)```
* ```removeObject(id, reqCtx)```

Which results in the following tree of ```Endpoint```s and ```Operation```s:

* ```/<collection>/```
   * ```POST``` which maps to ```insert```
   * ```GET``` which maps to ```find```
   * ```PATCH``` which maps to ```update```
   * ```DELETE``` which maps to ```remove```
* ```/<collection>/:_id```
   * ```PUT``` which maps to ```saveObject```
   * ```GET``` which maps to ```findObject```
   * ```PATCH``` which maps to ```updateObject```
   * ```DELETE``` which maps to ```removeObject```

#### (4.2.2) MongoDBCollection

The ```MongoDBCollection``` class is a ```Collection``` that is backed by a MongoDB database collection and exposes that database collection via REST.  

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    port: 8888,
    dbUri: 'mongodb://localhost:27017/mydb',
    endpoints: {
      messages: o({
        _type: carbon.carbond.mongodb.MongoDBCollection,
        collection: 'messages'
      })
    }
  })
})
```

Let's look at a more elaborate example:
* [Zipcode service][4]

#### (4.2.3) Custom Collections

You can create custom collections that implement the ```Collection``` interface however you like:

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    port: 8888,
    dbUri: "mongodb://localhost:27017/mydb",
    endpoints: {
      feedback: o({
        _type: carbon.carbond.collections.Collection,
        // POST /feedback
        insert: function(obj) {
          obj.ts = new Date()
          this.getService().db.getCollection("feedback").insert(obj)
        },
      })
    }
  })
})
```

## (5*) Working with other services

In a microservices architecture it is critical it is easy for microservices to communicate with each other. Carbon.io makes it very easy to write services that talk to other services.  

* Let's  take a look: [Hello world (chaining)][2]

## (6*) Authentication and access control

* Authentication is about determining who the user is.
* Access control is about controlling what the user has permission to do.

### (6.1) Authentication

#### (6.1.1) The Authenticator class

To implement an Authenticator you create an instance of the ```Authenticator``` class:

```node
o({
  _type: carbon.carbond.security.Authenticator,
  authenticate: function(req) { // also supports cb if desired
    var user = figureOutWhoUserIs()
    return user
  }
})
```

When configured on a Service the authenticator will authenticate all requests and expose the
authenticated user via ```req.user``` in all endpoint operations.

#### (6.1.2) Built-in Authenticators

Carbon.io comes with several built-in Authenticators:

* ```HttpBasicAuthenticator```: Base class for implementing HTTP basic authentication.
  * ```MongoDBHttpBasicAuthenticator```: An ```HttpBasicAuthenticator``` backed by MongoDB.
* ```ApiKeyAuthenticator```: Base class for implementing API-key based authentication.
  * ```MongoDBApiKeyAuthenticator```: An ApiKeyAuthenticator backed by MongoDB.
* ```OauthAuthenticator```: *coming soon*

Example

```node
o({
  _type: carbon.carbond.Service,
  port: 8888,
  authenticator: o({
    _type: carbon.carbond.security.MongoDBApiKeyAuthenticator,
    apiKeyParameterName: "api_key",
    apiKeyLocation: "header",   // can be "header" or "query"
    userCollection: "users",    // mongodb collection where user objects are stored
    apiKeyUserField: "apiKey"   // field that contains user api keys
  }),
  endpoints: {
    hello: o({
      _type: carbon.carbond.Endpoint,
      get: function(req) {
        return { msg: `Hello ${req.user.email}!`}
      }
    })
  }
})
```

### (6.2) Access control

Endpoints can configure an Access Control List (ACL) to govern which users can perform each HTTP
operation (e.g. GET, PUT, POST, etc.) on that Endpoint.

```node
o({
  _type: carbon.carbond.Endpoint,
  acl: o({
    _type: carbon.carbond.security.EndpointAcl
      groupDefinitions: { // This ACL defines two groups, 'role' and 'title'.
        role: 'role'      // We define a group called 'role' based on the user property named 'role'.
        title: function(user) { return user.title }
      }
      entries: [
        {
          user: { role: "Admin" },
          permissions: {
            "*": true // "*" grants all permissions
          }
        },
        {
          user: { title: "CFO" },
          permissions: { // We could have used "*" here but are being explicit.
            get: true,
            post: true
          }
        },
        {
          user: "12345", // User with _id "12345"
          permissions: {
            get: false,
            post: true
          }
        },
        {
          user: "*" // All other users
          permissions: {
            get: true,
            post: false,
          }
        }
      ]
    }),
    get: function(req) {
      return { msg: "Hello World!" }
    },
    post: function(req) {
      return { msg: `Hello ${req.body}!` }
    }
  })
})
```

## (7*) Concurrency

### (7.1) Fibers (the ```__``` operator)

Carbon.io uses [Node Fibers](https://github.com/laverdet/node-fibers) under the hood to manage the complexity
of Node.js concurrency. *Have you noticed any callbacks in the example code so far?*

Fibers allow you to write code that is *logically* synchronous (similar to async/await). Consider the following code snippet:

```node
fs.readFile("foo.txt", function(err, data) {
  if (err) {
    console.log(err)
  } else {
    console.log(data)
  }
})
```

With Fibers you can do this:

```node
__(function() {
  try {
    var data = fs.readFile.sync("foo.txt") // control flow is yielded
    console.log(data)
  } catch (err) {
    console.log(err)
  }
})
```

* The ```readFile``` function blocks the fiber (yields) until data is returned.
* If an error occurs it is thown as an exception (with a useful stacktrace). This is huge.

### (7.2) ```__```

The ```__``` operator is what spawns new fibers.

```node
__(function() {
  // Code that runs in fiber
  // Code in the context of the fiber may use .sync()
})
```

you can also optionally supply a callback:

```node
__(function() {
  // Code that runs in fiber
}, function(err, result) {
  // Code called when fiber returns
})
```

**Behavior**
* Code inside of a spwaned fiber runs asynchronous to the code that spawned the fiber
* If a callback is supplied, the return value from the function (or exception if thrown) is passed to the callback.
* From within the fiber ```.sync``` can be called to synchronously call functions (without *actually* blocking).

It should also be noted that you must use fibers in all top-level messages in the event loop. Examples:
* The main program
* Asynchronous functions (e.g. ```process.nextTick(function() { ... })```)
* In HTTP middleware to wrap the processing of an HTTP Request

### (7.3) ```.sync```

The ```.sync``` method can be called to synchronously call an asynchronous function as long as that function takes the standard
errback function as its last argument.

* Call by omiting the last errback argument
* The value will be returned by function
* An exception will be thrown if there was an err

There are two forms of ```.sync```:

* ```OBJ.sync.METHOD(ARGS)```

Example
```node
fs.sync.readFile("foo.txt")
```

* ```OBJ.METHOD.sync(ARGS)```

Example
```node
fs.readFile.sync("foo.txt")
```

Best practice: The first form should be used if there is a receiver, and the second on plain functions.

### (7.4) Creating synchronous wrappers

You can use ```.sync``` to make user-friendly synchronous wrappers:

```node
function readFile(path) {
  return fs.readfile.sync(path)
}
```

### (7.5) ```__.ensure``` vs ```__.spawn```

There are two variants of the ```__``` operator, ```ensure``` and ```spawn```.

* ```__.ensure```: Only spawns a new fiber if not already executing within a fiber (**default**).
* ```__.spawn```: Always spawns a new fiber.

Using ```__.ensure``` is particularly useful as top-level wrappers for applications that you also want to be able
to use as components / libraries.

A great example of this are unit tests that you might want to both be part of a larger test suite as well as runnable
standalone.

```node
var carbon = require('carbon-io')
var __     = carbon.fibers.__(module) // default behavior is that of __.ensure
var o      = carbon.atom.o(module).main

__(function() {
  module.exports = o({
    _type: carbon.testtube.Test,
    doTest: function() {
      // test code here
    }
    tests: [
      _o('./SubTest1'),
      _o('./SubTest2'),
    ]
  })
})
```

### (7.7) Revisiting our examples

* [Hello world (mongodb)][6] XXX is this valid?
* [Hello world (chaining)][7]

### (7.8) Advantages and disadvantages

Advantages
* Very natural control flow model.
* Exceptions and stack traces.
* Makes Node.js more functional. *Wait what? Not passing functions as callbacks is more functional?*
  * A pillar of functional programming is that expressions evaluate to values.

Disadvantages
* Can't be used in the browser.
* While usually obvious, it is not generally clear when control flow will yield under the hood (*beware of shared mutable state*).

### (7.9) Comparison to async / await
* Fibers works with callbacks or promise-based functions.
* Fibers and coroutines support deep continuations. This means that you can yield at any depth in the call stack and resume there later.
* async / await are based on generators which only support single-frame continuations. Yielding only saves 1 stack frame, and you must use async / await at every level of the call stack. This can be cumbersome, although more explicit about when yielding can occur.
* Fibers usually result in better stack traces / error handling. 

### (7.10) Future work (*no pun intended*)
* Better integration with Promises, and async/await (slated Carbon v0.8).

## (8) Testing with Test-tube

Carbon.io comes with a testing library called Test-tube.

### (8.1) Basic test structure

All tests have the following structure:

```node
var carbon = require('carbon-io')
var __ = carbon.fibers.__(module)
var o = carbon.atom.o(module).main
var testtube = carbon.testtube
var assert = require('assert')

__(function() {
  module.exports = o({
    _type: testtube.Test,
    name: 'ExampleTest',
    description: 'A simple test example.',
    setup: function(context, done) {},    // optional
    teardown: function(context, done) {}, // optional
    tests: [],                            // sub-tests (optional)
    doTest: function(context, done) {     // optional as well - usually you implement this or have sub-tests
       // test implementation
    }
  })
})
```

Test implementations (as well as ```setup``` and ```teardown```) can be synchronous or asynchronous.

Synchronous
```node
__(function() {
  module.exports = o({
    _type: testtube.Test,
    name: 'ExampleTest',
    description: 'A simple test example.',
    doTest: function(context) {
       assert(1 == 1)
    }
  })
})
```

Asynchronous
```node
__(function() {
  module.exports = o({
    _type: testtube.Test,
    name: 'ExampleTest',
    description: 'A simple test example.',
    doTest: function(context, done) {
      someAsyncFunction(function(err, value) {
        if (err) {
          done(e)
        } else {
          try {
            assert(value == 0)
          } catch (e) {
            done(e)
          }
          done()
        }
      })
    })
  })
})
```

### (8.2) Test suites

Since all ```Test``` objects can have an array of child / sub-tests, Tests are trees. This makes
it easy to manage large test suites.

```node
var __ = require('@carbon-io/carbon-core').fibers.__(module)
var o = require('@carbon-io/carbon-core').atom.o(module).main // Since this can run as main
var _o = require('@carbon-io/carbon-core').bond._o(module)
var testtube = require('@carbon-io/carbon-core').testtube

__(function() {
  module.exports = o({
    _type: testtube.Test,
    name: "Math test suite",
    tests: [
      _o('./AdditionTests'),
      _o('./SubtractionTests'),
      _o('./MultiplicationTests'),
      _o('./DivisionTests')
    ],
  })
})
```

* Test trees can be arbitrarily deep.
* Any test node in the tree can be run individually.

### (8.3) HttpTests

Test-tube makes it particularly easy to write HTTP-based tests.

```node
__(function() {
  module.exports = o({
    _type: testtube.HttpTest,
    name: "HttpTests",
    description: "Http tests",
    baseUrl: "http://localhost:8888",
    tests: [
      {
        reqSpec: {
          url: "/hello",
          method: 'GET'
        },
        resSpec: {
          statusCode: 200,
          body: "Hello world!"
        }
      },
    ]
  })
})
```

### (8.4) ServiceTests

The ```ServiceTest``` class is an extension of ```HttpTest``` that makes it easy to have the test start and stop your ```Service```
as part of the test process.

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.test.ServiceTest,
    name: "HttpTests",
    description: "Http tests",
    service: _o('./HelloService'),
    tests: [
      {
        reqSpec: {
          url: "/hello",
          method: 'GET'
        },
        resSpec: {
          statusCode: 200,
          body: "Hello world!"
        }
      },
    ]
  })
})
```

This test will:
1. Start ```HelloService```.
1. Run the HTTP tests defined in ```tests```.
1. Stop ```HelloService```.

### (8.5) Running tests

```shell
$ node test/HelloServiceTest
```

## (9) Generating API documentation for your Services

Each ```Service``` is capable of generating its own docs.

Flavors:
* Github Flavored Markdown
* Static HTML [using aglio](https://github.com/danielgtaylor/aglio)
* *We will be adding plug-in model*

```shell
$ node lib/HelloService.js gen-static-docs -h

Usage: /usr/local/bin/node HelloService.js gen-static-docs [options]

Options:
   -v VERBOSITY, --verbosity VERBOSITY   verbosity level (trace | debug | info | warn | error | fatal)
   --flavor FLAVOR                       choose your flavor (github-flavored-markdown | aglio)  [github-flavored-markdown]
   --out PATH                            path to write static docs to (directory for multiple pages (default: api-docs) and file for single page (default: README.md))
   -o OPTION, --option OPTION            set generator specific options (format is: option[:value](,option[:value])*, can be specified multiple times)
   --show-options                        show generator specific options

generate docs for the api
Environment variables:
  <none>

```

Example
```shell
$ node lib/HelloService.js gen-static-docs --flavor aglio --out api.html
```

* [Example output][9]

## (10) Should I use carbon.io in production?

While we at mLab do, we do not suggest using Carbon.io for production until the 1.0 release. 
*This presentation and examples are based on Carbon.io v0.6.*

## (11) Questions?


## (12) Additional Resources

* https://github.com/carbon-io/presentation__qcon-2017 (this talk)
* https://docs.carbon.io
* https://www.npmjs.com/package/carbon-io
* https://github.com/carbon-io/carbon-io
* https://github.com/carbon-io-examples
* will@mlab.com


[1]: https://github.com/carbon-io-examples/example__hello-world-service-parameter-parsing/tree/carbon-0.6
[2]: https://github.com/carbon-io-examples/example__hello-world-service-chaining/tree/carbon-0.6
[3]: https://github.com/carbon-io-examples/example__hello-world-service-aac/tree/carbon-0.6
[4]: https://github.com/carbon-io-examples/example__zipcode-service/tree/carbon-0.6
[5]: https://github.com/carbon-io-examples/example__contact-service/tree/carbon-0.6
[6]: https://github.com/carbon-io-examples/example__hello-world-service-mongodb/tree/carbon-0.6/lib/HelloEndpoint.js#L50
[7]: https://github.com/carbon-io-examples/example__hello-world-service-chaining/tree/carbon-0.6/lib/PublicHelloService.js#L58
[8]: https://github.com/carbon-io-examples/example__test-suites/tree/carbon-0.6
[9]: http://htmlpreview.github.io/?https://raw.githubusercontent.com/carbon-io-examples/contacts-service-advanced/blob/carbon-0.6/docs/index.html
[10]: https://github.com/carbon-io-examples/example__hello-world-service/tree/carbon-0.6

