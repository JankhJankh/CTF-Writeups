# A Stitch In Time

For the "A stich in time" challenge I was presented with a website with a login.   
In CTFs this generally makes me think I either need to get the administrator password or get an administrator session. I immediately noticed that this website requires a One-Time-Password as well as credentials to log in, and it looks like it was using Google authenticator for it. As Google Authenticator uses Time-based-One-Time Passwords (TOTP), it was a reasonable assumption that this website used TOTP.   
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time0.png)
I did a quick review of the source code and found links to the /logs directory.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time1.png)
Because directory listing is on, this showed me the location of the error.log file.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time2.png)
Browsing to this directory, I saw that it contained the logs for authentication, including the administrator username and password, as well as the times they have logged in.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time3.png)
This gave me two out of three things required for login, but I still needed the TOTP.    
So how did I get the TOTP?    
First, let’s take a step back and think about what a TOTP is.    
A Time-Based-One-Time password is the result of a one-way function that takes the current time, and a shared secret. The goal being that the secret is shared once, and from that point on, you only use the one-time passwords, as they cannot be used to retrieve the secret.    
Time is measured via UNIX standard time, also known as epoch. Epoch is essentially a big counter which started on 00:00:00 Thursday, 1 January 1970 and has been counting the seconds ever since. This is a standardised time format so that no matter what time zone you are in, the time is still the same.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time4.png)
So how it works in the real world is that the first time you log into a server using TOTP, you will only need your username and password. Then, when you enable TOTP it will give you a secret key –  usually a QR code to scan. Your phone then takes this secret key and, using the current time (with some rounding), creates a TOTP.    
Upon sending this TOTP to the server, the server will link your account to this secret, and whenever you log in it will require you to create a new TOTP based on the current time.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time5.png)
So, to generate this one-time password I needed to find the secret. But there should be no way to find it as once it is shared it is never shown again.
Looking back at the website, there was a button for generating a QR code for a TOTP. I clicked the button and it generated a QR code. I then scanned it and put it into Google authenticator, which generated a TOTP, and I put it in to the login information along with the username and password. 
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time6.png)
However, this login attempt still returned a “login failed” error.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time7.png)
Once again, looking back at the functionality we have seen so far within this CTF, I had the functionality to generate TOTPs, but without knowing what this functionality was doing, there were too many factors at play. For example, I still did not have any evidence that this generator was using the shared secret key for the admin account.    
To get a better idea of what the generate QR code button is doing, I looked at the code on the generate QR code button and saw that it was calling a function called *generateQRCode()*.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time8.png)
As this was a JavaScript function, I looked at the attached JavaScript files. When I looked in the login.js script, I saw the *generateQRCode()* function.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time9.png)
To verify the process of generating a QR code, I stepped through this code to see if it was correctly implemented. I found it was correctly taking the current epoch, and observed that for the secret, it was taking the users password. I also noticed that I could create seemingly valid TOTPs with this JavaScript and didn’t need to use Google Authenticator at all, so to be safe I rescanned the QR code and verified that the javascript gives the same code.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time10.png)
Once again, I tried this knowing  that I had:
*	The right time, because I trust the *Date().getTime()* function as a method to get the current epoch.
*	The secret, because I found the password in the logs.
*	The correct authenticator app, because I have used Google authenticator before.
As such, I should have a valid TOTP, so I tried the login again – and once again, it did not work.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time11.png)
At this point, I had verified every step of the TOTP generation, and I trusted the username and password found in the logs. So my hypothesis was that something was going wrong in the way the server was handling our data. 
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time12.png)
I decided to re-check the server logs to see if these valid requests were gettting a different error. However, due to the number of people doing the CTF, it was impossible to see what attempts were mine, so I tried it again, and directly after, put in a request that was unique so I could look for it in the logs.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time13.png)
These logins were successfully logged, and it was at this point I noticed what was wrong. I will let you try to figure it out for yourself, or you can scroll past the next photo for a spoiler.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time14.png)
The login time wasn’t the time I had seen when I generated the QR code. And it was not off by a few seconds, which would be understandable, but it was off by around 20000 seconds.   
The time I had generated the TOTP was 1555319075 but the time I saw in the logs afterwards was 1555344271.
Because the server was checking the TOTP passwords server side, but I was generating them client side, I had completely missed the fact that they are using different times. So, although I had been creating valid TOTPs, the server didn’t want my valid TOTPs. The server wanted TOTPs that matched its time.   
With this new information, I took the time from the most recent log in the error file, put the sever time into our generator, plus 30 seconds for a bit of room for error. I then generated the QR code off that, and got the TOTP that the server should think is the right time.   
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time15.png)
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time16.png)
Upon sending it this new TOTP, the server successfully logged me in and returned the flag.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time17.png)
To recap this challenge, bypassing authentication required a username, a password, the server time, and a TOTP secret key.
The username, password, and server time were leaked through the error logs, and the TOTP secret was leaked through client-side JavaScript.
![Image](https://raw.githubusercontent.com/JankhJankh/CTF-Writeups/master/BSides%20Canberra%202019/images/time18.png)
While this challenge wasn’t the hardest, it showed very good fundamentals of CTFing, and made sure that you really understood the concepts at play so you couldn’t just blindly stumble across the flag.  
Thanks to Cybears for organising this CTF.
