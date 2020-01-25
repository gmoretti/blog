---
layout: post
title:  "Scheduled REST call with AWS Lambda and Slack feedback"
description: Use Amazon Lambda to create scripts and execute them automatically.
date:   2019-08-04 20:00:00 +0200
---

The reason behind this short post came for two reasons. First, one of our recurrent tasks at work was making a call to a REST service to create a web-hook to it. Second, I've been playing around with Amazon Web Services for a while and thought that Lambdas were a perfect service for running simple scripts. I did some research and of course I was not the only one. Once this process is done we usually inform that it's ready via slack message, so I also thought it could be cool to get that done too.

# AWS Lambda
Amazon Lambda lets you choose the language for the functions you need, I choose Python just to test it, and because it has cool and well tested modules, as the Requests one, that we'll be using.

Setting up a lambda function is very simple following the [official documentation](https://docs.aws.amazon.com/lambda/latest/dg/getting-started-create-function.html) everything to code and test is available straight from the amazon console. So let's see what we need to do next.

# Breakdown
These are the three things we want to accomplish

1. Call out web service.
2. Call response to Slack.
3. Set a trigger for once-a-week execution

## Service Callout
Assuming you have already have a python script on a lambda in front of you, for calling to these services we will be using the [Requests module](https://2.python-requests.org//es/latest/) this library will make easy to deal with all the REST operations, and it's a good choice since it can be taken from the libraries that are provided inside a lambda by default. This rises a very important point, which is **any dependencies needed, have to be manually added to the lambda**, but there are libraries that are [already available inside a lambdas function](https://gist.github.com/gene1wood/4a052f39490fae00e0c3).

The following code imports the requests module from the available modules and does the needed call to the service. In my case in needed to be a POST call and it had to include a bearer token for authentication, so I created a couple of extra method.

```python
import json
import datetime
from botocore.vendored import requests

api_url = 'https://someservices.com/getSomeStuff'
myToken = 'someBearerToken'

def getHeader():
    return {
        'Content-Type': 'application/json', 'user-agent': 'my-app/0.0.1',
        'Authorization': 'Bearer {0}'.format(myToken)),
        'user-agent': 'my-app/0.0.1'
    }

def informPayload(region):
return {
    'payload1': 'value1',
    'payload2': 'value2',
    'payload3': 'value3'
}

def makeCall():
    headers = getHeader()
    payload = informPayload()    
    response = requests.post(api_url, headers=headers, json=payload)

    #I'm also interested in giving feedback when it fails (code 422) so i also added it as a returned response
    if (response.status_code == 201 or response.status_code == 200 or response.status_code == 422):
        return json.loads(response.content.decode('utf-8'))
    else:
        print(response.content)
        return None 

#This is the default handle for the lambda, it will be the entry poing
def lambda_handler(event, context):
    response = makeCall()

    if response is not None:
        print("Here's your response: ")
        print(response)
    
    else:
        print('[!] Request Failed')

    return {
        'statusCode': 200,
        'body': json.dumps('Avinode Hook was executed!')
    }
```
## Feedback through Slack
Now that we have our call code, we can use the same process of calling an external service to call Slack API and send a message to our team channel with the response. To get credentials to do so, you will have to go over the steps on Slack to create a new Salck APP and install it in your team's Slack workspace. You can follow the [official slack docs](https://api.slack.com/messaging/sending) for this. 
With this information we are going to modify the *lambda_handler* method and add a new sender method, here you have the new methods.

```python
#Updated
def lambda_handler(event, context):
    response = makeCall()

    if response is not None:
        slack_msg = '@here is the result of your call: \n \n'
        slack_msg += json.dumps(response)
        sendSlack(slack_msg)
        print("Here's your response: ")
        print(response)

    else:
        print('[!] Request Failed')

    return {
        'statusCode': 200,
        'body': json.dumps('Avinode Hook was executed!')
    }

#New Method
def sendSlack(message):
    slack_post_message_url = 'https://slack.com/api/chat.postMessage'
    slack_bearer_token = 'yourOwnToken'
    headers = {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer {0}'.format(slack_bearer_token)
    }
    body = {
        'channel': 'test2',
        'text': message
    }
    response = requests.post(slack_post_message_url, headers=headers, json=body)
```

## Scheduled Execution
Now, to execute your script on a regular basis Lambdas can be triggered by events, this events can be configured to be [CRON expressions](https://www.baeldung.com/cron-expressions), this is the ultimate tool to schedule tasks and is supported natively by lambdas. Following these two tutorials from official Amazon docs we can have our function scheduled in no time.

- This one will help us [set up an event](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/RunLambdaSchedule.html).

- The next one will show us [how to make the event a CRON expressions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/RunLambdaSchedule.html).

**Take into account that the event minimum time between executions is 1 minute, so the position in the CRON expression for seconds it's not used.**

# Final Thoughts
I think this is a very cheap an reliable way to have recurrent scheduled processes done, specially if you don't have the infrastructure or you don't want to waste a machine to have this done. On the other hand it will probably not be suitable for time-consuming tasks since Lambda service is charge on processing time, it could become quite expensive.
