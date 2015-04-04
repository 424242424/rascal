# Rascal

Rascal is a config driven wrapper around amqplib with [mostly safe](#caveats) defaults

## tl;dr

```javascript
var rascal = require('rascal')
var definitions = require('./definitions.json')

var config = rascal.withDefaultConfig(definitions)

rascal.createBroker(config, function(err, broker) {
  if (err) console.error(err.message) & process.exit(1)
  broker.subscribe('s1', function(err, message, content, next) {
    console.log(content)
    next()
  })
  setInterval(function() {
    broker.publish('p1', 'This is a test message')
  }, 100).unref()
})
```

definitions.json
```json
{
  "vhosts": {
    "/": {
      "exchanges": {
        "e1": {}
      },
      "queues": {
        "q1": {}
      },
      "bindings": {
        "b1": {
          "source": "e1",
          "destination": "q1"
        }
      }
    }
  },
  "publications": {
    "p1": {
      "exchange": "e1"
    }
  },
  "subscriptions": {
    "s1": {
      "queue": "q1"
    }
  }
}
```
## About
Rascal is a wrapper for the excellent [amqplib](https://www.npmjs.com/package/amqplib). One of the best things about amqplib is that it doesn't make assumptions about how you use it. Another is that it doesn't attempt to abstract away [AMQP Concepts](https://www.rabbitmq.com/tutorials/amqp-concepts.html). As a result the library offers a great deal of control and flexibility, but the onus is on you adopt appropriate patterns and configuration. You need to be aware that:

* Messages are not persistent by default and will be lost if your broker restarts
* Messages that crash your app will be infinitely retried
* Without prefetch a sudden flood of messages may bust your event loop
* Dropped connections and borked channels will not be automatically recovered
* Any connection or channel errors are emitted as "error" events. Unless you handle them or use [domains](https://nodejs.org/api/domain.html) these will cause your application to crash

Rascal seeks to solve these problems.

## Caveats
* Rascal currently implements only a small subset of the [amqplib api](http://www.squaremobius.net/amqp.node/doc/channel_api.html). It was written with a strong bias towards moderate volume pub/sub systems for a project with some quite agressive timescales. If you need one of the missing api calls, then your best approach is to submit a [PR](https://github.com/guidesmiths/rascal/pulls).

* Rascal deliberately uses a new channel per publish operation. This is because any time a channel operation encounters an error, the channel becomes unusable and must be replaced. In an asynchronous environment such as node you are likely to have passed the channel reference to multiple callbacks, meaning that for every channel error, multiple publish operations will fail. The negative of the new channel per publish operation, is a little extra overhead and the chance of busting the maxium number of channels (the default is 65K). We urge you to test Rascal with realistic peak production loads to ensure this isn't the case.

* Rascal has plenty of automated tests, but is by no means battle hardened (yet).


## Installation
```bash
npm install rascal
```

## Configuration

Rascal provides what we consider to be sensible defaults (optimised for reliability rather than speed) for production and test environments.

```js
var rascal = require('rascal')
var definitions = require('./your-config.json')
var config = rascal.withDefaultConfig(definitions)
```
or
```js
var rascal = require('rascal')
var definitions = require('./your-test-config.json')
var config = rascal.withTestConfig(definitions)
```
We advise you to review these defaults before using them in an environment you care about.

### Vhosts

#### namespace
Running automated tests against shared queues and exchanges is problematic. Messages left over from a previous test run can cause assertions to fail. Rascal has several strategies which help you cope with this problem, one of which is to namespace your queues and exchange.
```json
{
  "vhosts": {
      "v1": {
          "namespace": true
      }
  }
}
```
If you specify ```"namespace" :true``` Rascal will prefix the queues and exchanges it creates with a uuid. Alternatively you can specify your own namespace, ```"namespace": "foo"```. Namespaces are also if you want to use a single vhost locally but multiple vhosts in other environments.

#### connection
The simplest way to specify a connection is with a url
```json
{
  "vhosts": {
      "v1": {
          "connection": {
              "url":  "amqp://guest:guest@example.com:5672/v1?heartbeat=10"
          }
      }
  }
}
```
Alternatively you can specify the individual connection details
```json
{
  "vhosts": {
      "v1": {
          "connection": {
              "slashes": true,
              "protocol": "amqp",
              "hostname": "localhost",
              "user": "guest",
              "password": "guest",
              "port": 5672,
              "vhost": "v1",
              "options": {
                  "heartbeat": 5
              }
          }
      }
  }
}
```
Any attributes you add to the "options" sub document will be converted to query parameters. Providing you merge your configuration with the default configuration ```rascal.withDefaultConfig(config)``` you need only specify the attributes you want to override
```json
{
  "vhosts": {
      "v1": {
          "connection": {
              "hostname": "example.com",
              "user": "bob",
              "password": "secret"
          }
      }
  }
}
```
Rascal also supports automatic connection retries. It's enabled in the default config, or you want enable it specifically as follows.
```json
{
  "vhosts": {
      "v1": {
          "connection": {
              "retry": {
                  "delay": 1000
              }
          }
      }
  }
}
```
**If you decide against using Rascal's default confirmation and omit the retry configuration, amqplib will emit an error event on connection errors which will crash your application**. Because Rascal obscures access to the amqplib connection you will have no way to handle this error. A future version of Rascal will either expose the connection or handle and re-emit the errors so your application has the option of handling them

#### Exchanges

##### assert
Setting assert to true will cause Rascal to create the exchange on initialisation. If the exchange already exists and has the same configuration (type, durability, etc) everything will be fine, however if the existing exchange has a different configuration an error will be returned. Assert is enabled in the default configuration.

##### check
If you don't want to create exchanges on initialisation, but still want to validate that they exist set assert to false and check to true
```json
{
  "vhosts": {
      "v1": {
          "exchanges": {
              "e1": {
                  "assert": false,
                  "check": true
              }
          }
      }
  }
}
```

##### type
Declares the exchange type. Must be one of direct, topic, headers or fanout. The default configuration sets the exchange type to "topic" unless overriden.

##### options
Define any further configuration in an options block
```json
{
  "vhosts": {
      "v1": {
          "exchanges": {
              "e1": {
                  "type": "fanout",
                  "options": {
                      "durable": false
                  }
              }
          }
      }
  }
}
```
Refer to the [amqplib](http://www.squaremobius.net/amqp.node/doc/channel_api.html) documentation for further exchange options.

#### Queues

##### assert
Setting assert to true will cause Rascal to create the queue on initialisation. If the queue already exists and has the same configuration (durability, etc) everything will be fine, however if the existing queue has a different configuration an error will be returned. Assert is enabled in the default configuration.

##### check
If you don't want to create queues on initialisation, but still want to validate that they exist set assert to false and check to true
```json
{
  "vhosts": {
      "v1": {
          "queues": {
              "q1": {
                  "assert": false,
                  "check": true
              }
          }
      }
  }
}
```

##### purge
Enable to purge the queue during initialisation. Useful when running automated tests
```json
{
  "vhosts": {
      "v1": {
          "queues": {
              "q1": {
                  "purge": true
              }
          }
      }
  }
}
```

##### options
Define any further configuration in an options block
```json
{
  "queues": {
      "q1": {
          "options": {
              "durable": false,
              "exclusive": true
          }
      }
  }
}
```
Refer to the [amqplib](http://www.squaremobius.net/amqp.node/doc/channel_api.html) documentation for further queue options.

#### bindings
You can bind exchanges to exchanges, or exchanges to queues.
```json
{
  "vhosts": {
    "v1": {
      "exchanges": {
        "e1": {
        }
      },
      "queues": {
        "q1": {
        }
      },
      "bindings": {
        "b1": {
          "source": "e1",
          "destination": "q1",
          "destinationType": "queue",
          "bindingKey": "foo"
        }
      }
    }
  }
}
```
When using Rascals defaults, destinationType will default to "queue" and "bindingKey" will default to "#" (although this is only applicable for topics anyway)

### Publications
Now that you've bound your queues and exchanges, you need to start sending them messages. This is where publications come in.
```json
{
  "publications": {
    "p1": {
      "exchange": "e1",
      "vhost": "v1",
      "routingKey": "foo"
    }
  }
}
```
```javascript
broker.publish("p1", "some message")
```
If you prefer to send messages to a queue
```json
{
  "publications": {
    "p1": {
      "exchange": "e1",
      "vhost": "v1"
    }
  }
}
```
Rascal supports text, buffers and anything it can JSON.stringify. When publish a message Rascal sets the contentType message property to "text/plain", "application/json" (it uses this when reading the message too). The ```broker.publish``` method is heavily overloaded. Other variants are

```javascript
broker.publish("p1", "some message", callback)
broker.publish("p1", "some message", "foo")
broker.publish("p1", "some message", "foo", callback)
broker.publish("p1", "some message", { routingKey: "foo", options: { "expiration": 5000 } })
broker.publish("p1", "some message", { routingKey: "foo", options: { "expiration": 5000 } }, callback)
```
The callback parameters are err and the message id ```function(err, messageId) {}```. Rascal generates this message id unless you included it in the options object. Another option you should be aware of is the "persistent" option. Unless persistent is true, your messages will be discarded when you restart Rabbit. Despite having an impact on performance Rascal sets this in it's default configuration.

Refer to the [amqplib](http://www.squaremobius.net/amqp.node/doc/channel_api.html) documentation for further exchange options.

**It's important to realise that even though the ```broker.publish``` method can take a callback, this offers no guarantee that the message has been sent UNLESS you use a confirm channel**.
```json
{
  "publications": {
    "p1": {
      "exchange": "e1",
      "vhost": "v1",
      "confirm": true
    }
  }
}
```
Now each publish message will be ack'd or nack'd (in which case an err argument will be passed to the callback) by the server.

### Subscriptions
The real fun begins with subscriptions
```json
{
  "subscriptions": {
    "s1": {
      "queue": "e1",
      "vhost": "v1"
    }
  }
}
```
```javascript
broker.subscribe("s1", handler)
```
Rascal supports text, buffers and anything it can JSON.parse, providing the contentType message property is set correctly. Text messages should be set to "text/plain" and JSON messages to "application/json". Other content types will be returned as a Buffer. If the publisher doesn't set the contentType or you want to override it you can do so in the subscriber configuration.
```json
{
  "subscriptions": {
    "s1": {
      "queue": "e1",
      "vhost": "v1",
      "contentType": "application/json"
    }
  }
}
```
The ```broker.subscribe``` method is heavily overloaded. Other variants are
```javascript
broker.subscribe("s1", handler, callback)
broker.subscribe("s1", handler, { prefetch: 10 })
broker.subscribe("s1", handler, { prefetch: 10 }, callback)
```
The arguments to the handler are ```function(err, message, content, ackOrNack)```, where err is an error object (perhaps indicating a JSON error), the raw message, the content (a buffer, text, or object) and an ackOrNack callback. This callback should only be used for messages which were not ```{ "options": { "noAck": true } }``` by the subscription configuration or the options passed to ```broker.subscribe```.

For messages which are not auto-acknowledged (the default) calling ```ackOrNack()``` with no error argument will acknowledge it. Calling ```ackOrNack(err)``` will nack the message causing it to be discarded (or potentially sent to a dead letter exchange). If you want to requeue the message call ```ackOrNack(err, { requeue: true })```. You can also delay the requeue by specifying a defer argument, ```ackOrNack(err, { requeue: true, defer: 1000 })```

##### prefetch
Prefetch limits the number of unacknowledged messages your application can have outstanding. It's a great way to ensure that you don't overload your event loop or a downstream service. Rascal's default configuration sets the prefetch to 10 which may seem low, but we've managed to knock out firewalls, breach AWS thresholds and all sorts of other things by setting it to higher values.

### Defaults
Configuring each vhost, exchange, queue, binding, publication and subscription explicitly wouldn't be much fun. Not only does Rascal ship with default production and test configuration files, but you can also specify your own defaults in your configuration files by adding a "defaults" sub document.

```json
{
  "defaults": {
      "vhosts": {
          "exchanges": {
              "assert": true,
              "type": "topic"
          },
          "queues": {
              "assert": true
          },
          "bindings": {
              "destinationType": "queue",
              "bindingKey": "#"
          }
      },
      "publications": {
          "vhost": "/",
          "routingKey": "",
          "options": {
              "persistent": true
          }
      },
      "subscriptions": {
          "vhost": "/",
          "prefetch": 10,
          "retry": {
              "delay": 1000
          }
      }
  }
}
```

## Bonus Features

### Nuke
In a test environment it's useful to be able to nuke your setup between tests. The specifics will vary based on your test runner, but assuming you were using [Mocha](http://mochajs.org/)...
```javascript
afterEach(function(done) {
    broker ? broker.nuke(done) : done()
})
```

### Bounce (experimental)
Bounce disconnects and reinistialises the broker. We're hoping to use it for some automated reconnection tests

### Running the tests
```bash
npm test
```
You'll need a RabbitMQ server running locally with default configuration. If that's too much trouble try installing [docker](https://www.docker.com/) and running the following
```
docker run -d -p 5672:5672 -p 15672:15672 dockerfile/rabbitmq
```

