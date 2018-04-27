---
layout: post
title: Multiple content-types in AWS Lambda
---

I've been developing a new project that uses [AWS Lambda](https://aws.amazon.com/lambda/), a service from Amazon Web Services that lets you run code without having to set up servers etc.  AWS Lambda with [Chalice](https://github.com/aws/chalice), the AWS serverless architecture for Python, makes it easy to set up things like REST APIs.

Here, for example, is an application that simply echoes back a name that is sent to it using POST and x-www-form-urlencoded (i.e., typical form data).

```
from chalice import Chalice

app = Chalice(app_name='chalice_finsym')
app.debug = True

@app.route('/echoname',methods=["POST"],content_types=['application/x-www-form-urlencoded'],cors=True)
def echonm():
    rdata = parse_qs(app.current_request.raw_body.decode())
    name = rdata.get('nm')[0]
    resp = {}
    resp["name"] = name
    return resp
```

If you're writing a REST API, you can allow different output formats to be chosen, e.g., JSON or XML.  You might also be nice enough to allow different input content types - not just form data (using the x-www-form-urlencoded content-type), but maybe JSON as well.  Chalice allows you to specify a list of content-types in the route specification, but what is not clear (or not commonly explaind on the web), is how to detect which content-type has been used on a particular request.  You need to know this because it affects where you find the data in the request object.

So here is my solution.

```
@app.route('/echoname',methods=["POST"],content_types=['application/x-www-form-urlencoded','application/json'],cors=True)
def echonm():
    name = ''
    if ('content-type' in app.current_request.headers) :
        contenthdr = app.current_request.headers['content-type']
        if contenthdr == 'application/json' :
            rdata = app.current_request.json_body
            name = rdata.get('name')
        else :
            rdata = parse_qs(app.current_request.raw_body.decode())
            name = rdata.get('name')[0]
    else :
        # default content-type
        rdata = parse_qs(app.current_request.raw_body.decode())
        name = rdata.get('name')[0]
    resp = {}
    resp["name"] = name
    return resp
```

The key is to look for the content-type header in the headers dictionary of the request object, adn then check its value.  Beware, though, that the header may not be there - in which case, this code defaults to assuming its x-www-form-urlencoded. 