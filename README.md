Add voice and SMS messaging capabilities to your Node.JS applications with node-twilio-api!

# node-twilio-api

A high-level Twilio helper library to make Twilio API requests, handle incoming requests,
and generate TwiML.

Also ships with Connect/Express middleware to handle incoming Twilio requests.

**IMPORTANT**: You will need a Twilio account to get started (it's not free). [Click here to sign up for 
an account](https://www.twilio.com/try-twilio)

## Install

This project is in an **alpha** stage. Placing and receiving calls should now be working, but these
features are an their infantcy. **Do NOT use in production environments yet!
Use at your own risk!**

`npm install twilio-api`

## Features and Library Overview

Only voice calls are supported at this time, but I plan to implement the entire Twilio API over
the next few months.

 - [Create Twilio client](#createClient)
 - [Manage accounts and subaccounts](#manageAccts)
 - [List available local and toll-free numbers](#listNumbers)
 - [Manage Twilio applications](#applications)
 - List calls and modify live calls
 - [Place calls](#placingCalls)
 - [Receive calls](#incomingCallEvent)
 - [Generate TwiML responses](#generatingTwiML) without writing any XML (I don't like XML).
 - [Built-in pagination with ListIterator Object](#listIterator)

## Todo

 - List and manage valid outgoing phone numbers
 - List and provision incoming phone numbers
 - Support for Twilio Connect Applications
 - List and manage conferences, conference details, and participants
 - Send/receive SMS messages
	 - List SMS short codes and details
 - Access recordings, transcriptions, and notifications
 - Respond to fallback URLs
 - Better scalability with multiple Node instances
	- An idea for this is to intercept incoming Twilio requests only if the message is for
	that specific instance. Perhaps use URL namespacing or cookies for this?

## Usage

1. Create a Client using your Account SID and Auth Token.
2. Select your main account or a subaccount.
3. Do stuff...
	- Call API functions against that account (i.e. place a call)
	- Write logic to generate TwiML when Twilio sends a request to your
		application (i.e. when an incoming call rings)

```javascript
var express = require('express'),
    app = express.createServer();
var twilioAPI = require('twilio-api'),
	cli = new twilioAPI.Client(ACCOUNT_SID, AUTH_TOKEN);
app.use(cli.middleware() );
app.listen(PORT_NUMBER);
//Get a Twilio application and register it
cli.account.getApplication(ApplicationSid, function(err, app) {
	if(err) throw err;
	app.register();
	app.on('incomingCall', function(...) {
		//... functionality coming soon...
	});
	app.makeCall(...);
});
//... more sample code coming soon...
//For now, check the /tests folder
```

## API

The detailed documentation for twilio-api follows.

#### <a name="createClient"></a>Create Twilio client

Easy enough...

```javascript
var twilioAPI = require('twilio-api');
var cli = new twilioAPI.Client(AccountSid, AuthToken);
```

#### <a name="middleware"></a>Create Express middleware

- `Client.middleware()` - Returns Connect/Express middleware that handles any request for 
*registered applications*. A registered application will then handle the request accordingly if
the method (GET/POST) and URL path of the request matches the application's VoiceUrl,
StatusCallback, SmsUrl, or SmsStatusCallback.

Could this be much easier?

```javascript
var express = require('express'),
    app = express.createServer();
var twilioAPI = require('twilio-api'),
	cli = new twilioAPI.Client(AccountSid, AuthToken);
//OK... good so far. Now tell twilio-api to intercept incoming HTTP requests.
app.use(cli.middleware() );
//OK... now we need to register a Twilio application
cli.account.getApplication(ApplicationSid, function(err, app) {
	if(err) throw err; //Maybe do something else with the error instead of throwing?
	
	/* The following line tells Twilio to look at the URL path of incoming HTTP requests
	and pass those requests to the application if it matches the application's VoiceUrl/VoiceMethod,
	SmsURL/SmsMethod, etc. As of right now, you need to create a Twilio application to use the
	Express middleware. */
	app.register();
});
```

Oh, yes.  The middleware also uses your Twilio AuthToken to validate incoming requests,
[as described here](http://www.twilio.com/docs/security#validating-requests).

#### <a name="manageAccts"></a>Manage accounts and subaccounts

- `Client.account` - the main Account Object
- `Client.getAccount(Sid, cb)` - Get an Account by Sid. The Account Object is passed to the callback
	`cb(err, account)`
- `Client.createSubAccount([FriendlyName,] cb)` Create a subaccount, where callback is `cb(err, account)`
- `Client.listAccounts([filters,] cb)` - List accounts and subaccounts using the specified `filters`,
	where callback is `cb(err, li)` and `li` is a ListIterator Object.
- `Account.load([cb])` - Load the Account details from Twilio, where callback is `cb(err, account)`
- `Account.save([cb])` - Save the Account details to Twilio, where callback is `cb(err, account)`
- `Account.closeAccount([cb])` - Permanently close this account, where callback is `cb(err, account)`
- `Account.suspendAccount([cb])` - Suspend this account, where callback is `cb(err, account)`
- `Account.activateAccount([cb])` - Re-activate a suspended account, where callback is `cb(err, account)`

#### <a name="listNumbers"></a>List available local and toll-free numbers

- `Account.listAvailableLocalNumbers(countryCode, [filters,] cb)` - List available local telephone
numbers in your `countryCode` available for provisioning using the provided `filters` Object.
See [Twilio's documentation](http://www.twilio.com/docs/api/rest/available-phone-numbers#local)
for what filters you can apply. `cb(err, li)` where `li` is a ListIterator.
- `Account.listAvailableTollFreeNumbers(countryCode, [filters,] cb)` - List available toll-free
numbers in your `countryCode` available for provision using the provided `filters` Object.
See [Twilio's documentation](http://www.twilio.com/docs/api/rest/available-phone-numbers#toll-free)
for what filters you can apply. `cb(err, li)` where `li` is a ListIterator.

#### <a name="applications"></a>Applications

- `Account.getApplication(Sid, cb)` - Get an Application by Sid. The Application Object is passed to
	the callback `cb(err, app)`
- `Account.createApplication(voiceUrl, voiceMethod, statusCb, statusCbMethod,
	smsURL, smsMethod, SmsStatusCb, [friendlyName], cb)` - Creates an Application with
		`friendlyName`, where callback is `cb(err, app)`
		The `VoiceUrl`, `voiceMethod` and other required arguments are used to intercept incoming
		requests from Twilio using the provided Connect middleware. These URLs should point to the same
		server instance as the one running, and
		you should ensure that they do not interfere with the namespace of your web application.
- `Account.listApplications([filters,] cb)`
- `Application.load([cb])`
- `Application.save([cb])`
- `Application.remove([cb])` - Permanently deletes this Application from Twilio, where callback
	is `cb(err, success)` and `success` is a boolean.
- `Application.register()` - Registers this application to intercept the appropriate HTTP requests
	using the [Connect/Express middleware](#middleware).
- `Application.unregister()` - Unregisters this application. This happens automatically if the
	application is deleted.

A valid application must have a VoiceUrl, VoiceMethod, StatusCallback, StatusCallbackMethod,
SmsUrl, SmsMethod, and SmsStatusCallback.  Fallback URLs are ignored at this time.

#### <a name="placingCalls"></a>Placing Calls

- `app.makeCall(from, to, options[, onConnectCallback])` - Place a call and call the callback once the
	party answers. **The callbacks will only be called if `app` is a registered application!**
	If your application is registered, but your VoiceUrl is not set to the same server,
	the callee will receive an error message and a debug error will be logged on your account.
	For example, if your server is running at www.example.com, please ensure that your VoiceUrl is
	something like: http://www.example.com/twilio/voice
	Also, be sure that your VoiceUrl protocol matches your protocol (HTTP vs. HTTPS).
	
`from` is the phone number or client identifier to use as the caller id. If using a phone number,
	it must be a Twilio number or a verified outgoing caller id for your account.

`to` is the phone number or client identifier to call.

`options` is an object containing any of these additional properties:
	
- sendDigits - A string of keys to dial after connecting to the number. Valid digits in the string
	include: any digit (0-9), '#', '*' and 'w' (to insert a half second pause).
- ifMachine - Tell Twilio to try and determine if a machine (like voicemail) or a human has answered
	the call. Possible values are 'Continue', 'Hangup', and null (the default).
- timeout - The integer number of seconds that Twilio should allow the phone to ring before assuming
	there is no answer. Default is 60 seconds, the maximum is 999 seconds.

`cb` - Callback of the form `cb(err, call)`. This is called as soon as the call is placed.
	You can being building your TwiML response in the context of this callback, or you can listen for
	the various events on the Call Object.

Phone numbers should be formatted with a '+' and country code e.g., +16175551212 (E.164 format).

#### <a name="generatingTwiML"></a>Generating TwiML

Generating TwiML is as simple as calling methods on the Call Object.  Let's look at an example of
placing and handling an outbound call:

```javascript
/* Make sure that you have already setup your Twilio client, loaded an application, and started
	your server. Twilio must be able to contact your server for this to work. Ensure that your
	server is running, proper ports are open on your firewall, etc.
*/
app.makeCall("+16145555555", "+16145558888", function(err, call) {
	if(err) throw err;
	//Now we can use the call Object to generate TwiML and listen for Call events
	call.on('connected', function(status) {
		/* This is called as soon as the call is connected to 16145558888 (when they answer)
			Note: status is probably 'in-progress' at this point.
			We can generate our TwiML here, as well.
		*/
	});
	//Now we generate TwiML for this call... which will served up to Twilio when the call is connected
	call.say("Hello. This is a test of the Twilio API.");
	call.pause();
	var input = call.gather(function(call, digits) {
		call.say("You pressed " + digits + ".");
		var str = "Congratulations! You just used Node Twilio API to place an outgoing call.";
		call.say(str, {'voice': 'man', 'language': 'en'});
		call.pause();
		call.say(str, {'voice': 'man', 'language': 'en-gb'});
		call.pause();
		call.say(str, {'voice': 'woman', 'language': 'en'});
		call.pause();
		call.say(str, {'voice': 'woman', 'language': 'en-gb'});
		call.pause();
		call.say("Goodbye!");
	}, {
		'timeout': 10,
		'numDigits': 1
	});
	input.say("Please press any key to continue. You may press 1, 2, 3, 4, 5, 6, 7, 8, 9, or 0.");
	call.say("I'm sorry. I did not hear your response. Goodbye!");
	call.on('ended', function(status, duration) {
		/* This is called when the call ends.
			Note: status is probably 'completed' at this point. */
	});
});
```

Here are all of the TwiML commands you can use:

 - `Call.say(text[, options])` - Reads `text` to the caller using text to speech. Options include:
	- `voice` - 'man' or 'woman' (default: 'man')
	- `language` - allows you pick a voice with a specific language's accent and pronunciations.
		Allowed values: 'en', 'en-gb', 'es', 'fs', 'de' (default: 'en')
	- `loop` - specifies how many times you'd like the text repeated. The default is once.
		Specifying '0' will cause the &lt;Say&gt; verb to loop until the call is hung up.
 - `Call.play(audioURL[, options])` - plays an audio file back to the caller. Twilio retrieves the
	file from a URL (`audioURL`) that you provide. Options include:
	- `loop` - specifies how many times the audio file is played. The default behavior is to play the
		audio once. Specifying '0' will cause the the &lt;Play&gt; verb to loop until the call is hung up.
 - `Call.pause([duration])`	- waits silently for a `duration` seconds. If &lt;Pause&gt; is the first
	verb in a TwiML document, Twilio will wait the specified number of seconds before picking up the call.
 - `Call.gather(cbIfInput, options, cbIfNoInput)` - Gathers input from the telephone user's keypad.
	Calls `cbIfInput` once the user provides input, passing the Call object as the first argument and
	the input provided as the second argument. If the user does not provide valid input in a timely
	manner, `cbIfNoInput` is called if it was provided; otherwise, the next TwiML instruction will
	be executed.  Options include:
	- `timeout` - The limit in seconds that Twilio will wait for the caller to press another digit
		before moving on. Twilio waits until completing the execution of all nested verbs before
		beginning the timeout period. (default: 5 seconds)
	- `finishOnKey` - When this key is pressed, Twilio assumes that input gathering is complete. For
		example, if you set 'finishOnKey' to '#' and the user enters '1234#', Twilio will
		immediately stop waiting for more input when the '#' is received and will call
		`cbIfInput`, passing the call Object as the first argument and the string "1234" as the second.
		The allowed values are the digits 0-9, '#' , '*' and the empty string (set 'finishOnKey' to '').
		If the empty string is used, &lt;Gather&gt; captures all input and no key will end the
		&lt;Gather&gt; when	pressed. (default: #)
	- `numDigits` - the number of digits you are expecting, and calls `cbIfInput` once the caller
		enters that number of digits.
	The `Call.gather()` function returns a Gather Object with methods: `say()`, `play()`, and `pause()`,
	allowing you to nest those verbs within the &lt;Gather&gt; verb.
 - `Call.record(...)` - Not yet implemented
 - `Call.sms(...)` - Not yet implemented
 - `Call.dial(...)` - Do not use. Not tested.
 - `Call.hangup()` - Ends a call. If used as the first verb in a TwiML response it does not prevent
	Twilio from answering the call and billing your account.
 - `Call.redirect(url, method)` - Transfers control of a call to the TwiML at a different URL.
	All verbs after &lt;Redirect&gt; are unreachable and ignored. Twilio will request a new TwiML document
	from `url` using the HTTP `method` provided.
 - `Call.reject(reason)` - rejects an incoming call to your Twilio number without billing you. This
	is very useful for blocking unwanted calls. If the first verb in a TwiML document is &lt;Reject&gt;,
	Twilio will not pick up the call. The call ends with a status of 'busy' or 'no-answer',
	depending on the `reason` provided. Any verbs after &lt;Reject&gt; are unreachable and ignored.
	Possible `reason`s include: 'rejected', 'busy' (default: 'rejected')
 - `Call.cb(cb)` - When reached, Twilio will use the &lt;Redirect&gt; verb to re-route control to the
	specified callback function, `cb`. The `cb` will be passed the Call object. This is useful if you
	want to loop like this:

	```javascript
	(function getInput() {
		call.gather(function(call, digits) {
			//Input received
		}).say("Please press 1, 2, or 3.");
		call.cb(getInput); //Loop until we get input
	})();
	```

#### <a name="callEvents"></a>Call Events

- Event: 'connected' `function(status) {}` - Emitted when the call connects. See the
	[Twilio Documentation]
	(http://www.twilio.com/docs/api/twiml/twilio_request#request-parameters-call-status)
	for possible call status values.
- Event: 'ended' `function(status, callDuration) {}` - Emitted when the call ends. See the
	[Twilio Documentation]
	(http://www.twilio.com/docs/api/twiml/twilio_request#request-parameters-call-status)
	for possible call status values.

#### <a name="appEvents"></a>Application Events

- Event: 'outgoingCall' `function(call) {}` - Triggered when Twilio connects an outgoing call
	placed with `makeCall`.
- <a name="incomingCallEvent"></a>Event: 'incomingCall' `function(call) {}` - Triggered when the
	Twilio middleware receives a voice request from Twilio. Once you have the Call Object, you
	can [generate a TwiML response](#generatingTwiML).

#### <a name="listIterator"></a>ListIterator

A ListIterator Object is returned when Twilio reponses may be large. For example, if one were to list
all subaccounts, the list might be relatively lengthy.  For these responses, Twilio returns 20 or so
items in the list and allows us to access the rest of the list with another API call.  To simplify this
process, any API call that would normally return a list returns a ListIterator Object instead.

The ListIterator Object has several properties and methods:

- `Page` - A property of the ListIterator that tells you which page is loaded at this time
- `NumPages` - The number of pages in the resultset
- `PageSize` - The number of results per page (this can be changed and the default is 20)
- `Results` - The array of results. If results are a list of accounts, this will be an array of Account
	Objects, if it's a list of applications, this will be an array of Application Objects, etc.
- `nextPage([cb])` - Requests that the next page of results be loaded. Callback is of the form
	`cb(err, li)`
- `prevPage([cb])` - Requests that the previous page of results be loaded.

## Testing

twilio-api uses nodeunit right now for testing. To test the package, run `npm test`
in the root directory of the repository.

**BEWARE:** Running the test suite *may* actually place calls and cause you to incur fees on your Twilio
account. Please look through the test suite before running it.

## Disclaimer

Blake Miner is not affliated with Twilio, Inc. in any way.
Use this software AT YOUR OWN RISK. See LICENSE for more details.