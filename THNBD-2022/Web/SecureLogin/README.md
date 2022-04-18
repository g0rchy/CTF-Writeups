# Light Recon:
Looking at the response headers we see: `Server: Werkzeug/2.0.0 Python/3.9.5`, so we're dealing with a Flask/Django web application.

![login](https://user-images.githubusercontent.com/92731777/163890788-47cea03d-0876-4876-8c9e-16e847334f72.png)

We're facing a login prompt, after supplying some dummy credentials and hitting "submit" the web page respond with "Passwords do not match.", interestingly no request has been sent during login so it's safe to assume that the credentials are checked on the client-side.

# What the hell was that line? :

Looking at the source code [here](./index.html), we see that a function called `checkPswd()` gets invoked when we click on the "submit" button (line 67), which is also defined on line 72 inside a long obsufcated in-line Javascript code. Time to analyze what it does!
At first i've beautified and deobsfucated the code using online tools, running each block inside nodejs console while playing around with the `checkPswd()` but it seemed to be very impractical, evantually i've ended up using the debugger inside the browser's DevTools which proved to be efficient in this case.
To do so, open the DevTools, click on "sources" or "debugger" (depends on the browser, i've used chromium) then at left double click on `(index)`, a small pop up will ask if we want to "Pretty-print" the file (which is nice in our case), click on it. you should get something like this:

![image](https://user-images.githubusercontent.com/92731777/163889248-8369af48-78c8-4d15-a0a6-7b503df0f0df.png)

Going through it quickly there is catchy line `jQuery['ajax']` (in my case it was line 345 after beautifying the code) which takes as an argument an array that mainly contains (along with other properties) a url, data that we want to send, and the type (HTTP verb) to be used, so our guess here is that the checks are done before sending our credentials to the server.

But before that there's an if statement and right above it a line that contains the following strings: ("getElement", "pswd" and "value"), so here we cann assume that it stores our input inside a variable, run some checks on it (most likely for the correct credentials) then proceeds to send something to the server (again maybe our credentials).

Let's set a breakpoint on that if statement by clicking on it's equivalent number, which should look like this:

![image](https://user-images.githubusercontent.com/92731777/163889328-262923bd-ee69-4bfa-bd42-446dc4482a1e.png)

Getting back to that login form, submit some dummy info (username:password in my case) and the browser will continue executing the javascript code until it hits our breakpoint.

Hopping yet again to the debugger, we can see all the strings corresponding to each variable, first our password is stored right before the if statement, so our assumption was right, now for the interesting part, looking at the scope (either by looking at the menu at the right or by clicking on /hovering over the array) we can see that our input is being checked with two interesting strings "th3w4aba" and "3w4b4s3cuR3p4ssw@rd":

![image](https://user-images.githubusercontent.com/92731777/163889346-030ff4d2-2e1e-43e2-8df2-5b10a4b8b434.png)

# Let me in! : 
After Stoping/Closing that debugger, let's login using th3w4aba:3w4b4s3cuR3p4ssw@rd, finally we're in!

![image](https://user-images.githubusercontent.com/92731777/163890863-1650810c-a02c-4010-abf5-2558012ec519.png)

But wait, where's the flag? 
Looking at the source code we won't find anything, we must have missed something...
Remember that jquery ajax request that was supposed to run after that if statement check? well, let's look at the requests been made after logging in:

![image](https://user-images.githubusercontent.com/92731777/163889580-e353723f-efbd-4b4a-b13c-fb38fae66b3c.png)

Nothing interesting besides the `/welcome` endpoint which we only did a `GET` request to it, where's that POST request?

Let's hop back to the debugger and set up a breakpoint on that variable right before that `jquery['ajax']` function (line 344 in my case), after examining it's content we see another string which is "register", probably another endpoint/parameter?

After stepping around (using the arrows up right) we can see that indeed a POST request has been made to /register endpoint with our username and password as parameters

![post](https://user-images.githubusercontent.com/92731777/163891343-db8075be-5e98-4c70-bb3e-d053d63bf0a6.png)
![data](https://user-images.githubusercontent.com/92731777/163891356-d227faff-eac3-4c9a-99ea-1b78cd5deb0f.png)


Interestingly, the `window.location` object has been used to redirect us to `/welcome`, i've used the console at the far bottom to deobsfuscted that gibberish

![image](https://user-images.githubusercontent.com/92731777/163889653-dc745e0c-3f52-4e07-a17c-5289ecfb8a80.png)

So, the reason why we haven't seen any request being made to `/request`is that the ajax request hasn't done yet before redirecting us to `/welcome`, thus resulting in the browser to discard it. [More Info](https://stackoverflow.com/questions/16961896/ajax-call-and-window-location) 
# I said login not register :
Since we can't capture that POST request (we might if we got lucky) let's play with it using curl (i'll proxy it to burp for easier interaction) : `curl -d 'username=th3w4aba&password=3w4b4s3cuR3p4ssw@rd' https://thnbdarija-s3cured.chals.io/register -x http://127.0.0.1:8080 -k`

![image](https://user-images.githubusercontent.com/92731777/163889730-3a3ce0be-e4e6-436f-9cf9-37532ddce197.png)


After sending this request, we'll notice that our input is reflected back to us, changing the username and password doesn't affect the logic behind this endpoint, this screams SSTI!
The engine used would be jinja2 (since it runs in python) :
![image](https://user-images.githubusercontent.com/92731777/163889738-04ce5259-94a1-4d95-bfd1-6910d2ceace5.png)

After lurking around, the flag is stored under `/home/app/SSTI/flag.txt`:

![image](https://user-images.githubusercontent.com/92731777/163889761-57d8b2d6-bfae-4ce8-b839-2ecbe529da0e.png)

Flag: `thnb{1_gu355_3w4b4_f41l3d_4g4in_l337}`


