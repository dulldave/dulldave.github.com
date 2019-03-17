---
title: "AWS SQS Dead Letter Replay Lambda"
description: "Ever needed to replay items in your dead letter queue?"
date: "2019-03-17T10:00:00Z"
---

I came across an incident recently where data was going to a service via SQS, upon hitting that service it would then talk to a bunch of other services to gather more data for the request. At the same time, an attack had taken down the connection to those other services which was causing our requests to fail and build upon a dead letter queue. Now, these requests weren’t broken per se, they were just victim to a one-off transient error. The data within the request was perfectly valid, however, the dead letter queue was used more for logging the very occasional permanent error that had occurred and being able to replay those items was never needed.

Because of that incident, I wanted to build something that could be turned on at a moments notice and be used to replay items in the dead letter queue, albeit temporarily.

Lambda seemed to be a good fit for this. We could specify a queue to trigger the Lambda against and a queue to replay those items too. Which would mean if we got notified via our CloudWatch alarms that the DLQ was filling up we could diagnose the issue and at the right time consume those items in the DLQ.

The other benefit of using a Lambda is that we can have all of the code sitting, with the triggers disabled, and only when we need to enable them to start the process.

![AWS Lambda console showing an enabled trigger](./enabled-trigger.png)

The code itself is very simple. When the Lambda is triggered via SQS, it loops over the records that come in and simply forwards that message onto the output queue defined in the environment variables for the Lambda. Which then means the services listening to the output queue can start reprocessing the messages that had failed. 🎉🎉🎉

The full source code can be found here: [dulldave/aws-sqs-dead-letter-replay-lambda](https://github.com/dulldave/aws-sqs-dead-letter-replay-lambda)
