<!-- 
.. title: Simple Flask Webhook
.. slug: simple-flask-webhook
.. date: 2016-05-24 01:52:11 UTC+01:00
.. tags: web, programming, python, flask
.. category: programming
.. link: 
.. description: A simple webhook implementation using Flask
.. type: text
-->

# Introduction
Since this is my first technical blog post, I though I'd start with something relatively simple. Also, please bare with me as I get to grips with the formatting of this confounded contraption!  
If you have any feedback, then please feel free to send me a mail at [james.innes@ogma-dev.com](mailto:james.innes@ogma-dev.com).  

In this post, I'm going to give a quick guide on how to get a fairly simple, yet incredibly useful, webhook setup with [Flask](http://flask.pocoo.org/).  

## Source Code
You can find the complete source code here: [https://github.com/Ogma-Dev/Simple-Flask-Webhook](https://github.com/Ogma-Dev/Simple-Flask-Webhook)  

## What is a Webhook?
Very basically; a webhook is a HTTP web service that listens on a server and will respond and/or perform a task when they are called by some event. [Read more](https://en.wikipedia.org/wiki/Webhook).  

## What is Flask?  
Flask is a micro [web framework](https://en.wikipedia.org/wiki/Web_framework) written in Python. [Read more](http://flask.pocoo.org/).  

## Prerequisites
Since you're here, I'm going to assume you have Python installed. Python 2 is soon(ish...) to be depreciated, so I'm going to be using Python 3 here (and in the future).  

### Virtual Environments
In my opinion it's always good practice to use virtual environments. I currently use Ubuntu, so the following examples to setup will be with aptitude, but I'm sure it's mostly translatable to whichever package manager your linux distribution uses.  

If you don't have virtual environments setup on your system, you should install the `virtualenv` package with:  
```
$ sudo apt-get install virtualenv
```
_We'll get into making the virtual environment in a little bit... For now, you just need it installed._

# Let's Get Started
Ok, with the introduction out of the way, let's get started...  

## Setup
I like to start all new applications in a clean folder of their own... So, that's where we'll start. Make a new directory and let's call is `simple_flask_webhook`, then change directory into that folder;
```
$ mkdir simple_flask_webhook
$ cd simple_flask_webhook
```
Once in here, we will setup our virtual environment for our webhook;
```
$ virtualenv --python=python3 venv
```

Activate the virtual environment (i.e. set it as your working installation of Python);  
```
$ source venv/bin/activate
```
You should now have a `(venv)` infront of your bash prompt (if you don't have any shell fanciness!).

## Python Packages
With the virtual environment created and activated we need to install Flask;
```
(venv)$ pip install flask
```
If all went well and flask installed, your output of `pip freeze` should look something like this:
```
(venv)$ pip freeze
Flask==0.10.1
itsdangerous==0.24
Jinja2==2.8
MarkupSafe==0.23
pkg-resources==0.0.0
Werkzeug==0.11.9
```
_Note: Don't worry about the version numbers, by the time you read this it might be a bit old!_  

## The Webhook
Right, now let's get to the good stuff!  

Create a new file, `webhook.py`;
```
(venv)$ touch webhook.py
```

Open it up with your favourite editor, and we'll start writing our app...  

```
from flask import Flask, request, abort

app = Flask(__name__)


@app.route('/webhook', methods=['POST'])
def webhook():
	if request.method == 'POST':
		print(request.json)
		return '', 200
	else:
		abort(400)


if __name__ == '__main__':
	app.run()
```

Hah! I told you it was simple! The webhook will print out any json you send to it. You can test it out with `curl`.  
i.e. `curl -H "Content-Type: application/json" -X POST -d '{"data": "This is some test data"}' 127.0.0.1:5000/webhook`  

### Adding a Little Validation
Well, that's ok; but perhaps a little more security wouldn't go a miss...
Let's implement a simple token validation;  

```
import os
from datetime import datetime, timedelta
from flask import Flask, request, abort, jsonify

def temp_token():
	import binascii
	temp_token = binascii.hexlify(os.urandom(24))
	return temp_token.decode('utf-8')

WEBHOOK_VERIFY_TOKEN = os.getenv('WEBHOOK_VERIFY_TOKEN')
CLIENT_AUTH_TIMEOUT = 24 # in Hours

app = Flask(__name__)

authorised_clients = {}


@app.route('/webhook', methods=['GET', 'POST'])
def webhook():
	if request.method == 'GET':
		verify_token = request.args.get('verify_token')
		if verify_token == WEBHOOK_VERIFY_TOKEN:
			authorised_clients[request.remote_addr] = datetime.now()
			return jsonify({'status':'success'}), 200
		else:
			return jsonify({'status':'bad token'}), 401

	elif request.method == 'POST':
		client = request.remote_addr
		if client in authorised_clients:
			if datetime.now() - authorised_clients.get(client) > timedelta(hours=CLIENT_AUTH_TIMEOUT):
				authorised_clients.pop(client)
				return jsonify({'status':'authorisation timeout'}), 401
			else:
				print(request.json)
				return jsonify({'status':'success'}), 200
		else:
			return jsonify({'status':'not authorised'}), 401

	else:
		abort(400)

if __name__ == '__main__':
	if WEBHOOK_VERIFY_TOKEN is None:
		print('WEBHOOK_VERIFY_TOKEN has not been set in the environment.\nGenerating random token...')
		token = temp_token()
		print('Token: %s' % token)
		WEBHOOK_VERIFY_TOKEN = token
	app.run()
```
Wooft, well; that's a bit meatier. So, what's going on here...  
When you start the app, it will check for an Environment Variable, called `WEBHOOK_VERIFY_TOKEN`, if there isn't one set it will automatically generate a token for you.  
We initialise a dictionary of `authorised_clients`, which stores IP addresses (key) and the time they validated (value).  
In order for an IP to become validated, they need to sent a GET request with a parameter `validate_key` equal to our webhooks token.  
i.e. `curl 127.0.0.1:5000/webhook?verify_token=YOUR_TOKEN_HERE`  
Once an IP is registered the webhook will accept POSTs from it, the same as previously.
It's not exactly Fort Knox, but it'll suffice for now.  

### Further Improvements
This post is only supposed to give a brief introduction to writing webhooks and there are many improvements you could make which are probably outwith the scope of this post. However, here are a few suggestions to get you started!  

#### Persistent Authorised Clients
Since we only store the autorised IP addresses in a dictionary, they're lost whenever the application closes. You could store these into a file format, or even a database. [Flask-SQLAlchemy](http://flask-sqlalchemy.pocoo.org/) makes simplifies using databases with Flask!

#### Added Authentication
If you want to add further authentication, I'd recommend checking out [Flask-HTTPAuth](https://github.com/miguelgrinberg/flask-httpauth).  

#### HTTPS
If you want to enable HTTPS (without using a full blown http service like nginx), pass your SSL certificates to Flask by modifying the run line to; `app.run(ssl_context=(ssl_cert_file, ssl_key_file))` where the variables are paths to your certificates.  
[LetsEncrypt](https://letsencrypt.org/getting-started/) is a really quick and easy way to get setup with some certs!

## End Notes
I hope you find this useful, and if you have any comments or have built on the examples and would like to share please feel free to [drop me a line](mailto:james.innes@ogma-dev.com).  
