# CouchDB Queue Service

CQS is a message queue system, using Apache CouchDB. It is **exactly like** [Amazon Simple Queue Service (SQS)][sqs_api]. The API is the same. Everything is exactly the same, it just runs on CouchDB.

CQS is implented in Javascript and supports:

* NodeJS
* Google Chrome 12
* Firefox 3.5, Firefox 3.6, Firefox 4
* Safari 5
* Internet Explorer 8, Internet Explorer 9

Use CQS if you use Javascript, you know (or appreciate) Amazon SQS, and you *want the same thing on your server*.

For Node, install with NPM.

    $ npm install cqs

The test script `test/run.js` will copy itself into a Couch app which you can run from the browser.

## API

Initialize the CQS module to point to a database on your couch.

    // A normal import.
    var cqs = require('cqs');
    
    // Pre-apply my couch and db name.
    cqs = cqs.defaults({ "couch": "https://user:password@example.iriscouch.com"
                       , "db"   : "cqs_queue"
                       });

### List Queues

    cqs.ListQueues(function(error, queues) {
      console.log("Found " + queues.length + " queues:");
      queues.forEach(function(queue) {
        console.log("  * " + queue.name);
      })

      // Output:
      // Found 2 queues:
      //   * a_queue
      //   * another_queue
    })

### Create Queues

Creating queues requires **database administrator** access.

    // Just create with a name.
    cqs.CreateQueue("important_stuff", function(error, queue) {
      if(!error)
        console.log("Important stuff queue is ready");
    })

    // Create with an options object.
    var opts = { QueueName               : "unimportant_stuff"
               , DefaultVisibilityTimeout: 3600 // 1 hour
               , browser_attachments     : true // Attach browser libs and test suite
               };

    cqs.CreateQueue(opts, function(error, queue) {
      if(!error)
        console.log("Created " + queue.name + " with timeout + " queue.VisibilityTimeout);

      // Output
      // Created unimportant_stuff with timeout 3600
    })

### Send a Message

Everything is like SQS, except the message body is any JSON value.

    // The convenient object API:
    important_stuff.send(["keep these", "things", "in order"], function(error, message) {
      if(!error)
        console.log('Sent: ' + JSON.stringify(message.Body));

      // Output:
      // Sent: ["keep these","things","in order"]
    })

    cqs.SendMessage(important_stuff, "This message is important!", function(error, message) {
      if(!error)
        console.log('Sent message: ' + message.Body);

      // Output:
      // Sent message: This message is important!
    })

    // Or, just use the queue name.
    cqs.SendMessage('some_other_queue', {going_to: "the other queue"}, function(error, message) {
      if(!error)
        console.log('Message ' + message.MessageId + ' is going to ' + message.Body.going_to);

      // Output:
      // Message a9b1c48bd6ae433eb7879013332cd3cd is going to the other queue
    })

### Receive a Message

Note, like the SQS API, `ReceiveMessage` always returns a list.

    // The convenient object API:
    my_queue.receive(function(error, messages) {
      if(!error)
        console.log('Received message: ' + JSON.stringify(messages[0].Body));

      // Output:
      // Received message: <message body>
    })

    // The standard API, receiving multiple messages
    cqs.ReceiveMessage(some_queue, 5, function(er, messages) {
      if(!error)
        console.log('Received ' + messages.length + ' messages');

      // Output:
      // Received <0 through 5> messages
    })

### Delete a Message

When a message is "done", remove it from the queue.

    // The convenient object API:
    message.del(function(error) {
      // Message deletion never results in an error. If a message is successfully
      // deleted, it will simply never appear in the queue again.
      console.log('Message deleted!');
    })

    // The standard API:
    cqs.DeleteMessage(my_message, function(error) {
      console.log('Message deleted');
    })

## To Do

I wish CQS had many more features. (Pelcome watches.)

* Use one design document instead of one per queue. One design document is better at the low end, multiple design documents are better at the high end. Ideally, we can choose one or the other.
* Management
  * Every API call has a 0.1% chance of running management tasks
  * You could run a dedicated management process so the producers/consumers don't have to worry
  * Purge old messages. Instead of HTTP `DELETE`, update the document with `_deleted=true` and a timestamp. Messages older than N days get purged. The hard thing about purge is avoiding view rebuilds. I believe this is the correct procedure:
    1. Preflight checks. These are optional but can reduce the time the lock is held.
      1. Freshen the views. For each design document:
        1. Pick a deterministically random view based on `_id` and `_rev`
        1. Query the view `?reduce=false&limit=1`
        1. Query `_info`. If `purge_seq` changes during this loop, abort
      1. If DB compaction is running, abort
    1. Fetch `_local/maintenance`. Example fields in this doc:
      * `owner` : some UUID
      * `updated_at` : `"2011-09-15T14:23:16.021Z"`
      * `expires_at` : `"2011-09-15T14:53:16.021Z"`
    1. Think of a UUID
    1. Store `_local/purging` with your UUID, and perhaps a timestamp, as a lock. (Two purges in a row is disastrous.)
    1. Identify `_id`s and `_rev`s to purge. They must be deleted first.
    1. 

* Purge old messages. Set `_deleted`=true with a timestamp, and then 
[sqs_api]: http://docs.amazonwebservices.com/AWSSimpleQueueService/latest/APIReference/
