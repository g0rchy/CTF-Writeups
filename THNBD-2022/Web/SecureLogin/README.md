# Light Recon:
Looking at the response headers we see: `Server: Werkzeug/2.0.0 Python/3.9.5`, so we're dealing with a Flask/Django web application.

Landing on the main page, we're presented with a login prompt, after supplying some dummy credentials and hitting "submit" the web page response with "Passwords do not match.", interestingly no request has been sent during login so it's safe to assume that the credentials are checked on the client-side.

# What the hell was that line? :
Looking at the source code, we see that a function called `checkPswd()` gets invoked when we click on the "submit" button (line 67), which is also defined on line 72 inside a long obsufcated in-line Javascript code. Time to analyze what it does!
At first i've beautified and deobsfucated the code using online tools, running each block inside nodejs console while playing around with the `checkPswd()` but it seemed to be very impractical, evantually i've ended up using the debugger inside the browser's DevTools which proved to be efficient in this case.
To do so, open the DevTools, click on "sources" or "debugger" (i've used chromium) at left double click on `(index)`, a small pop up will ask if we want to "Pretty-print" the file (which is nice in our case), click on it. you should have something like this: [screen shot]

Going through it quickly there was a pretty catchy line `jQuery['ajax']` (in my case it was line 345 after beautifying the code) which takes as an argument an array that mainly contains (along with other properties) a url, data that we want to send, and the type (HTTP verb) to be used, so our guess here is that the checks are done before sending our credentials to the server.

But before that there's an if statement and right above it a line that contains the following strings: ("getElement", "pswd" and "value"), so here we can safely assume that it stores our input inside a variable, run some checks on it (most likely the credentials) then proceeds to send something to the server.

Let's set a breakpoint on that if statement by clicking on it's equivalent number, which should look like this: [breakpoint screenshot]

getting back to that login form, submit some dummy info (username:password in my case) and the browser will continue executing the javascript code until it hits our breakpoint.

hopping yet again to the debugger, we can see all the strings corresponding to each variable, first our password is stored right before the if statement, so our assumption was right, now for the interesting part, looking at the scope (either by looking at the menu or by clicking on /hovering over the array) we can see that our input is being checked with two interesting strings "th3w4aba" and "3w4b4s3cuR3p4ssw@rd": [gotpassword]

# Let me in! : 
Closing that debugger, let's login using th3w4aba:3w4b4s3cuR3p4ssw@rd, finally we're in!
[404]
but wait, where's the flag? 
looking at the source code we won't find anything, we must have missed something...
Remember that jquery ajax request that was supposed to run after that if statement check? well, let's look at the requests been made after logging in:
[requests]
nothing interesting besides the `/welcome` endpoint which we only did a `GET` request to it, where's that POST request?

Let's hop back to the debugger and set up a breakpoint on that variable right before that `jquery['ajax']` function (line 344 in my case), after examining it's content we see another string which is "register", probably another endpoint/parameter?

After stepping around (using the buttons up right) we can see that indeed a POST request has been made to /register endpoint with our username and password as parameters [screenshot]

Interestingly, the `window.location`object has been used to redirect us to `/welcome`(notice the console at the far down) [screenshot]

So, the reason why we haven't seen any request being made to `/request`is that the ajax request hasn't done yet before redirecting us to `/welcome`, thus resulting in the browser to discard it. [More Info](https://stackoverflow.com/questions/16961896/ajax-call-and-window-location) 
# I said login not register :
Since we can't capture that POST request (we might if we got lucky) let's play with it using curl (i'll proxy it to burp for easier interaction) : `curl -d 'username=th3w4aba&password=3w4b4s3cuR3p4ssw@rd' https://thnbdarija-s3cured.chals.io/register -x http://127.0.0.1:8080 -k`
[screenshot]
After sending this request, we'll notice that our input is reflected back to us, changing the username and password doesn't affect the logic behind this endpoint, this screams SSTI!
The engine used would be jinja2 (since it runs in python) :
[RCE]
Lurking around, the flag is stored under `/home/app/SSTI/flag.txt`:
[flag]

Flag: `thnb{1_gu355_3w4b4_f41l3d_4g4in_l337}`


