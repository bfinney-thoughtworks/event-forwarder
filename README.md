# Rabblerouser Event Forwarder

[![Build Status](https://travis-ci.org/rabblerouser/event-forwarder.svg?branch=master)](https://travis-ci.org/rabblerouser/event-forwarder)

A lambda function that receives all events from a kinesis stream and forwards them to an API over HTTP

## Background.
AWS Kinesis consumers need to be constantly polling the stream for new events (with a dedicated thread for it).
NodeJS will struggle with this as it is single-threaded.

This is a lambda function that can do that dedicated job, and forward events to a consumer via HTTP.

## Install and run the tests
```sh
yarn
yarn test
```

## Run it locally
You can either run it natively:

```sh
KINESIS_ENDPOINT'http://localhost:4567' STREAM_NAME='rabblerouser_stream' yarn start
```

Or you can build the docker image like this...
```sh
docker build -t rabblerouser/event-forwarder .
```

... and then run it with docker-compose. E.g. see [core](https://github.com/rabblerouser/core).

## Deployment
There is a build pipeline for this project, it publishes the zipped code to an s3 bucket.

Actual deployment of this lambda function requires a few moving parts, such as IAM roles, a kinesis stream to subscribe
to, and a consumer endpoint where events should be forwarded to.

With that in mind, the easiest way to deploy this right now is as part of a whole Rabble Rouser stack. See
[infra](https://github.com/rabblerouser/infra) for how to do that.

## API Reference

### Input

This lambda function receives an event object from kinesis that looks like this:

```js
{
  Records: [
    {
      kinesis: {
        kinesisSchemaVersion: "1.0",
        partitionKey: "<kinesis partition key>",
        sequenceNumber: "<sequence number of the event>",
        data: "<base64-encoded JSON string>", // This is where the real payload data is
        approximateArrivalTimestamp: 123456.78
      },
      eventSource: "aws:kinesis",
      eventVersion: "1.0",
      eventID: "<shardID of the source shard>:<sequence number of the event>",
      eventName: "aws:kinesis:record",
      invokeIdentityArn: "<ARN of the IAM role the lambda is running as>",
      awsRegion: "<region of the source kinesis stream>",
      eventSourceARN: "<ARN of the source kinesis stream>"
    },
  ]
}
```

### Output

This lambda function sends an HTTP POST request to a configurable endpoint, with a JSON body that looks like this:

```json
{
  "kinesisSchemaVersion": "1.0",
  "partitionKey": "<kinesis partition key>",
  "sequenceNumber": "<sequence number of the event>",
  "data": "<base64-encoded JSON string>",
  "approximateArrivalTimestamp": 123456.78
}
```

## Error handling
Lambda functions always notify the caller as to whether the function finished successfully or not. When the event source
is a kinesis stream, success means that the next batch of events can be sent, whereas failure means that the batch needs
to be retried.

Success/failure for this event forwarder lambda is based on the HTTP status code of the service that it POSTs events to.
This means that if the service is down, or failing for some reason, the events will be retried until the service
responds with a success code.

The other thing to remember is that the lambda can receive a batch multiple of events in a single call. This is
problematic if some of the events in the batch were processed successfully and others failed, especially if you
need to be sure that each event is only processed once. For that reason, it's recommended that this lambda function is
deployed with a maximum batch size of 1 in its configuration. This ensures that each individual event can succeed or
fail atomically.

Processing one event at a time is a bit of an anti-pattern, however. It would be good if we had a way for a batch of
events to partially succeed/fail, so that only the failed ones are retried.
