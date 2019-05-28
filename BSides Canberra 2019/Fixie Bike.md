# Fixie Bike

Like most challenges, this one started with a URL link. This link takes us to a website with an input field, a button that says ‘submit banner’, and three links relating to login functionality.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie0.png)            
I first checked for any information in the source code, such as passwords, comments leaking data, and information about what the server is.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie1.png)            
When I saw a /flag directory, I immediately browsed to it, to find a “Seems unlikely buddy.” message. This gave me a strong indication that my goal was to obtain an administrator session and browse to this location.      
After this I started assessing the login functionality. When I clicked the log in button, it made the page say logged in, but it doesn’t change anything about the website. I did notice that a request was sent to the /auth directory, so I navigated to it manually and found that it returned your user level.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie2.png)            
After sending a few requests through a proxy my team noticed that we could log in with whatever name we wanted. We also noted a header showing that the server was running Python 3.7 and aiohttp 3.1.3
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie3.png)            
Exploring more of the website functionality, I also noticed that I could add a custom banner.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie4.png)            
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie5.png)            
As the banner is reflected to the user, I immediately checked for Cross-Site Scripting (or XSS). The idea being that if we can enter user data and make the browser treat our data as part of the website, we could potentially attack other users of the website – such as an administrator. 
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie6.png)            
Unfortunately, XSS is sanitised here, as shown by the *\<h1\>hi\</h1\>* being rendered literally instead of as HTML (in which case, it would just say “hi” and be in a different font).     
I ended up trying a few more strings, the last of which was *“\<h1\>aaaa\</h1\>”*, and then looked at the website via a proxy again, this time aiming to see how it is sanitised. 
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie7.png)            
However, upon viewing this code I saw that the banner string I had sent to the webserver had been put into a server header. Headers are extra bits of information hidden from users that give the browser extra information about the website. such as how much data there is, what type of data it is, and any cookies that the server has generated for the user.      
Now that I knew that I could put user data into a header I attempted header injection. Header injection is an exploit where if user data is controlled in a header, and some form of newline (‘\r\n’, ‘\n’, ‘0x0a’, ‘0x0d’) can be put into the header, you could create any new headers you want. If there is no restriction on space, you could potentially control all data returned to the user (after the injection point, that is). A classic example is adding XSS to the website and then commenting out the rest of it. The example code for this is shown below.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie8.png)            
This successfully worked and popped up an alert box.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie9.png)            
To give a better understanding of why this works, see the below code.  
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie10.png)           
By adding two new lines, we are telling the browser that the headers are finished and that the website content has started. Then, we add our malicious code to pop an alert box, and use a html comment “<!—” To hide the rest of the web page.     
Now that we had functioning XSS, I tried to steal the administrator’s session cookie, which I believed to be the ILOVEMYFIXIEBIKE cookie. As I had verified that this cookie didn’t have HTTP-only enabled (a protection to stop this kind of attack), I created a payload and sent it to the webserver. This payload sends the cookies of anyone visiting the site to 127.0.0.1. I tested this locally first as there are less things that can go wrong, and where possible you should always test exploits locally first.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie11.png)           
I checked that it was rendering correctly, and then browsed to the webpage. 
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie12.png)           
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie13.png)           
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie14.png)           
It successfully redirected me to 127.0.0.1 and sent the cookies to my webserver. Below is a diagram showing this attack chain.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie15.png)           
Now that I knew that this worked, I tried it on an external web server by swapping out the 127.0.0.1 with the public IP address of a cloud instance I spun up. However, nothing came through. To save time I asked the organisers if there would be any firewalling restrictions in place, and they told me that there are no firewalls blocking things.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie16.png)           
This leads to the problem. Before I continued my research down this path, I reviewed the information about the challenge and read the hints that a developer is accessing the website, and that they use a non-traditional browser. With this new information I drew up a graph to demonstrate my knowledge and assumptions at this point. We had verified an attack chain that worked on our machine given our browser, user interaction, and firewall settings. From the context of the challenge and my answer from one of the organisers we knew that their machine uses a non-traditional browser, an admin is regularly visiting the site, and there are no firewall settings on their machine.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie17.png)           
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie18.png)           
As we can see from this, If we put trust in the challenge context and the firewall info given to us, it is clear to see that the browser is they key problem with our attack chain.    
This was not enough information to solve the problem. There could be XSS but requiring a more complex payload, or an entirely different attack vector. For example, XSS cannot be exploited through the admin browser if they are not executing the code retrieved. Curl and Wget are two such examples.    
After doing some googling I discovered the webserver has a matching aiohttp client for use with the server. So, I downloaded this software and tested how it interacted with the server.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie19.png)           
From my testing with aiohttp, it looked as though it didn’t execute anything, and just returned the data. So, my new goal became to find what data the client does interact with.     
I injected random strings into the webpage and requested the site through aiohttp to see how it would respond, and found that giving it broken characters returned an error:
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie20.png)           
So, from this I was confident that the data was interacting with server headers. Another way you can interact with a browser via headers is with the Location header. This header tells the browser where to redirect to. My logic was that if I could make the admin redirect to the flag page, then perhaps there was a way for me to retrieve the information from it.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie21.png)           
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie22.png)           
This unfortunately did not work as aiohttp correctly ignores the Location header on 200 responses. This is understandable behaviour because the specification says to only pay attention to Location headers if it is on a 301 (Redirect), and not a 200 (OK) response. Doing more research into possible ways to exploit the aiohttp client, I came across a CVE from 2018.     
*https://github.com/aio-libs/aiohttp-session/issues/272*
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie23.png)           
This CVE is for a session fixation attack, which is a vulnerability in which sessions are not correctly invalidated. In this case if a cookie is not tied to a session, but a user currently has a cookie, the server will use that cookie for its session.      
This attack made a lot of sense for this scenario as web clients need to interact with Set-Cookie headers, as without them these web clients would be unable to maintain sessions. So, I followed this attack chain by injecting the Set-Cookie header into the server so that any user who accesses the server will have their session cookie overwritten with mine.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie24.png)           
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie25.png)           
tested this new header injection in my browser and it correctly sets this new cookie – if the user already has a cookie it becomes overwritten, and if they do not have a cookie it is added (at least, this was the functionality I believe I was seeing).
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie26.png)           
Assuming the admin was also viewing the page, their session should have been overwritten with our cookie as well. By logging out, we invalidate their session, but let the admin keep their cookie. When they log back in, the cookie should have their privileges associated with it.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie27.png)           
We can now refresh the page, and our cookie now has admin privileges
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie28.png)           
Once I had been logged back in by my header injection, I checked the auth page to verify that the session fixation attack had been successful.   
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie29.png)            
As it said I was an admin, I browsed to the /flag directory found earlier and was presented with a flag.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie30.png)           
Below is a timeline graphic to better show how this was exploited:
raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/fixie31.png)          
In review, to access the flag we required a valid admin session. This session was hijacked though a session fixation attack by forcing an administrator to use our malicious cookie. The administrator was forced to use our malicious cookie via a header injection attack, enabling us to later log in as the administrator.      
Once again, thanks for Cybears for running this CTF. I really enjoyed the layered aspects of this challenge.
