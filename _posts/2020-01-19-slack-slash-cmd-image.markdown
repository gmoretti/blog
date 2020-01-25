---
layout: post
title:  "Image manipulation Slack slash command with AWS"
description: An slash command in Slack + AWS that responds with an image with text sended as parameter.
date:   2020-01-25 14:01:20 +0200
---

After bein able to successfully send back messages back to Slack, another chance to itengrate Slack with AWS came up.
The idea was simple, I had an image to which I wanted to add arbitrary text from Slack. I envisioned this as creating a call from Slack with [Slack Slash Commands](https://api.slack.com/interactivity/slash-commands) and simply answer it with [AWS Lambda](https://aws.amazon.com/es/lambda/features/)... with a twist.

## The idea

To be able to achieve this I ended up having to use quite a few pieces from the AWS stack:

1. Setup Slash command to send a call with the desired text as payload
2. **Lambda Function** to deal with the request
3. **Amazon SNS** messaging system to publish the requests
4. A **lambda function** that processes the image and sends back te response to Slack

Let's tackle these steps one by one.

## Slack slash command

This is the easy part. All we have to do is setup an Slack App and then setup an slack command.

In the configuration of the Slack command a URL has to be defined, which is were the Slack command would point at. This call, as specified in [Slack Slash Commands](https://api.slack.com/interactivity/slash-commands) documentation sends a `application/x-www-form-urlencoded` content-type header and a body with, among other things, the slash command's payload which is text we will be printing in out image. The header is relevant as we will see when setting up our Lambda function.

There's another important constraint regarding slash commands. This is, **the answer to the request made by slack cannot be more than 3000ms**. We will later see that the **image processing took longer sometimes** times. For these cases, Slack sends a parameter `response_url` where the final response should be sended up to 30 minutes later.

## AWS Services
### API Gateway + Lambda + SNS + S3

To be able to offer a response and then took our time to process the image and later respond with it, we will need an **API Gateway endpoint pointing to a first LAMBDA**, we will call this lambda *proxyLambda*. 

To handle long request asynchronously we will make use of [AWS SNS](https://docs.aws.amazon.com/sns/latest/dg/sns-lambda-as-subscriber.html), this is a messaging platform from Amazon and it will let us use the first *proxyLambda* to publish a message into AWS SNS and have antoher lambda subcribed to SNS for reading and processing the requests, we will call this Lambda *subscriberLambda*.

Now, once our *subscriberLambda* has finish it needs to store the new image in an **S3 bucket**, since Slack is expecting just a text message with a link to an image to desplay.

With all this the flow should look something like this:

1. API Gateway receives request from Slack and proxies it to *proxyLambda*
2. *proxyLambda* publishes a message to SNS with the request data and answers to Slack ASAP
3. *subscriberLambda* gets trigger on the new message and processes the message, generating the image

#### API Gateway configuration

The idea for the Gateway configuration is to get Slack request proxied straight to the lambda. The reason behind this is what we previusoly said about the Slack slash command request type. The standard configuration to attach an AWS API Gateway to a lambda funtion is via a method (say GET or POST), the gateway expects a payload to be in json, if not it crashes and the request never gets to the lambda. I was able to solve this issue by usin the proxy configuration while configuring a new resource.

![proxy resource 1]({{ site.baseurl }}/assets/images/slash-comm-1.PNG "proxy resource 1")

The configuration to sned the request to the lambda it's also pretty straightforward.

![proxy resource 2]{{ site.baseurl }}/(assets/images/slash-comm-2.PNG "proxy resource 2")

#### Proxy Lambda function

Now once we have our requests proxied to our lambda, we will use it to handle it and push it to our SNS channel. 
Doing it like this we can respond back to Slack very fast and handle the actual request later. 
The lambda function would look like this.

```javascript
console.log("Loading function");
var AWS = require("aws-sdk");

exports.handler = function(event, context, callback) {
    var eventText = JSON.stringify(event.body, null, 2);
    console.log("Received event:", eventText);
    var sns = new AWS.SNS();
    var params = {
        Message: eventText, 
        Subject: "Test SNS From Lambda",
        TopicArn: "arn:aws:sns:us-east-1:333333:your-SNS-topic"
    };
    sns.publish(params, context.done);

    callback(null, {
        statusCode: 200,
        body: "temporal response to Slack...",
        headers: {'Content-Type': 'application/json'}
    });
};
```
This is actually all we need to forward the request to SNS. The important part here is the Topic ARN, this gets created
when we create our Messaging channel.

#### Subscriber Lambda

Here is the lambda that would do the actual work, this functions would be listening for messages in the SNS channel
and will act accordingly when a new one comes. Lets see how the main part looks like. Bear in mind, there's a lot happening
in here: Reading SNS message and parsing the request parameters, new modified image generation, storing the new image in an
S3 bucket and finally sending the link and response back to Slack.

```javascript
const engine = require('./ImageGenerator'); // This is a separate file with the image processing code
var uuid = require("uuid"); // Id generator for the new images
const fs = require('fs');
const AWS = require('aws-sdk');
var request = require('request');
var s3 = new AWS.S3();

exports.handler = async function(event, context, callback) {
    
    // This section reads the SNS message and parses the request parameters
    var message = event.Records[0].Sns.Message;
    let buff = Buffer.from(message, 'base64');
    let text = buff.toString('ascii');
    let rawParams = text.split('&');
    let params = new Object();

    for(var param of rawParams) {
        let paramSplitted = param.split('=');
        params[paramSplitted[0]] = paramSplitted[1];
    }

    // Now we will use some methods from an imported file to process and add the requested text to the image.
    // For some reason the text from the request puts '+' instead of spaces so we get rid of those
    console.log(params);
    let newImage = await engine.generate(decodeURIComponent(params.text).replace(/\+/g,' '));


    // Now we send the request to S3 to save the newly creted image; now here we wait for the response to 
    // be able to generate the slack message with the new url for the file. 
    var paramsS3 = {
        "Body": newImage,
        "Bucket": "YOU-BUCKET-NAME",
        "Key": uuid.v4() + '.jpg',
        "ACL": "public-read",
        "ContentType": "image/jpg" 
    };
    s3.upload(paramsS3, async function(err, data){
        if(err) {
            callback(err, null);
        } else {
            console.log(params.response_url);
            await sendSlack(data.Location, decodeURIComponent(params.response_url));
        };
    });

    // Lambda response is not really relevant, since our objective of sending the slack message has been previously achieved
    callback(null, null);
};

async function sendSlack(data, url) {
    const options = {
        url: url,
        method: 'POST',
        headers: { 
            'Content-Type': 'application/json',
            'Authorization': "Bearer xoxp-r46y54byeby6erny6erny6erny6ern"
        },
        body: JSON.stringify({
            'replace_original': true,
            'response_type': 'in_channel',
            'text': '',
            "attachments": [{
                'text': '',
                'image_url': data
            }],
        })
    };

    request(options, function(err, res, body) {
        let json = JSON.parse(err);
        console.log(json);
    });
}
```
**The full lambda with dependencies** can be found in this [GitHub Repository](https://github.com/gmoretti/slack-slash-cmd-image-process-lambda)
Now take in to account that your code can get a little bit to big and labmda user interface **will not let you handle it from there**
so you will need something like this to deploy the files from you local environment.

```sh
aws lambda update-function-code --function-name your-lambda-name --zip-file fileb://the-zip-with-the-code.zip --region us-east-1
```
This AWS-CLI command requires you first **zip your code**, so a tiny script does the both could become handy at this point, and also,
if the intention is to automate the whole deploy for example.

#### Amazon SNS configuration

Ok, Now lets configure the magic of connecting these to Lambdas, a.k.a. The message broker configuration.
This is very easy when done from the user interface. We go to **Amazon SNS** configuration in the **AWS console** and
crete a new Topic. **Just fill the Name and Description and leave the rest for now.**

Now the important part, once the Topic is created we need to create a **New Subscription** for this topic, and select our
subscriber lambda here.

![proxy resource 3]({{ site.baseurl }}/assets/images/slash-comm-3.PNG "proxy resource 3")

This is how we will get our lambda executed every time there's a new request in the queue.

## Final thoughts

Altought being a cool approach to test all theese technologies, i'm pretty sure i'll would have gone with a different
one when realising my image processing was taking so long for Slack, 3 secs should be enough in my opinion.
In any case I have learned a lot, I hope you do too.


Any thoughts?, feel free to contact me, **thank you!**
