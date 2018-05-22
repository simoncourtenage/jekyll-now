---
layout: post
title: Problems with AWS API Gateway and API Keys
---

I'm writing this post as a kind of aide memoire more to my future self, but I hope it will be helpful to others as well.  It concerns a really annoying problem with Aamazon Web Services' API Gateway, and its use of API keys.

I won't bore you with useful introductions to REST APIs and API keys.  You can google those.  But let's define our environment first.

I'm playing around with using Python and the Chalice framework to write a REST API in API Gateway.  Let's call the API the UsefulAPI, and part of it is defined in Python/Chalice as:

    @app.route('/doit',methods=["POST"],cors=True,api_key_required=True)
    def dosomething() :
        response = {}
        response['status'] = 200
        response['message'] = "Did it!"
        return response

So this API is deployed to AWS API Gateway, and the endpoint URL is something like

    https://ab01c2defg.execute-api.us-east-1.amazonaws.com/api/

This URL has two interesting parts to it - the API ID `ab01c2defg`, and the stage `api`.  Those bits of data will come in handy later. (Note - I've changed key bits of data in this post to use dummy values.)

So here is the problem.  I have a website, call it www.usefulwebsite.com, with a web page that has this javascript/jquery code in it:

    $.ajax({
        url : 'https://ab01c2defg.execute-api.us-east-1.amazonaws.com/api/doit',
        method : "POST",
        beforeSend : function (request) {
            request.setRequestHeader("X-Api-Key", 'ArandomStringOfNumbersAndDigits')
        }
    }).done(function (data) {
        alert(data.message)
    })

The jQuery code uses ajax to call the REST API, with the `doit` route.  It uses the POST request method (which matches the route specification) and adds, before the request is sent, an API key using the `X-Api-Key` header (also required by the route specification).

So, what should happen, when the jquery code is run, is that the browser should make a XmlHttpRequest call to the UsefulAPI, which should run the `dosomething()` Python function that matches the route.  The response (in JSON) should be returned to the browser, which pops up an alert box displaying "Did It!".

Does this happen?  No.  What we see is nothing.  What we get on the browser console is a HTTP 403 status code.

This is where it gets slightly confusing.  Browsers like Firefox always think that HTTP 403 status code means a [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) (Cross-Origin Resource Sharing) failure.  (Read the linked article to understand what this means.)  But servers routinely use 403 to mean forbidden in general.  For example, API Gateway uses 403 to mean AccessDenied (see [here](https://docs.aws.amazon.com/apigateway/api-reference/handling-errors/)).  So the browser console tells you one meaning, when the server means another.

After lots of testing, experimenting, reading stackoverflow, I realised this important point.  What the browser was telling me wasn't what was happening.  But what was happening?

I wish I could claim credit for how I solved this problem.  But what really happened was I stumbled across this [post](https://forums.aws.amazon.com/thread.jspa?messageID=836109&#836109) on the AWS API Gateway user forum.  In it, you discover two important things:

1. The AWS documentation on creating API keys for API Gateway is incomplete.  It doesn't (as far as I can see) tell you that once you create an API key, you need to associate it with an API ID and a stage.
2. The AWS API Gateway web console has a bug in it - the part that associate a key with an API and a stage doesn't work.

For example, on point 2: after reading the first part of the post, I used the web console to associate a key with my API and its stage (using the two bits of info from earlier).  Then, on my command line, I used this command to get info on the key:

    aws apigateway get-api-key --api-key 4x1yz28ab4

(Another frustrating point - it took me ages to realise that the value of `--api-key` is NOT the key, but the ID of the key!)

This was the output:

    {
        "id": "4x1yz28ab4",
        "name": "dummy.email.address@gmail.com",
        "description": "API key for the UsefulAPI",
        "enabled": true,
        "createdDate": 1526909964,
        "lastUpdatedDate": 1526909964,
        "stageKeys": []
    }

The `stageKeys` field should be filled out - but it's empty!  This is where the second part of the forum post applies.  Once I did

    aws apigateway update-api-key --api-key 4x1yz28ab4 --patch-operations op=add,path=/stages,value=ab01c2defg/api

and re-ran the get-api-key command, the output was

    {
        "id": "4x1yz28ab4",
        "name": "dummy.email.address@gmail.com",
        "description": "API key for the UsefulAPI",
        "enabled": true,
        "createdDate": 1526909964,
        "lastUpdatedDate": 1526909964,
        "stageKeys": [
            "ab01c2defg/api"
        ]
    }

And, when I re-ran my jquery, the HTTP status code was 200 and I saw the alert box in the browser displaying "Did it!"

What I have to work out now is how to execute the update-api-key command programmatically, in the Python code that creates API keys using the boto3 library.  But helpful note to Amazon - (i) please update your docs, (ii) fix the bug in the API Gateway web console, and (iii) consider using a different status code for AccessDenied!