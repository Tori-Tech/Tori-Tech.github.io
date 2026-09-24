---
title: "I built an Arduino TOTP Token Generator"
layout: post
date: 2026-09-23 12:00:00 +0000
categories: [Hardware, hacking_gadgets, tools, Cybersecurity]
tags: [arduino, totp, 2fa, gadgets, hacking_gadgets, tools, hardware-hacking]
---

I became curious about Time-based One Time Passwords after I failed to log into my GitHub account because my system clock had drifted about 5 minutes behind "real" time. I sat at my desk for about fifteen minutes, re-entering codes from the open-source password manager that I use, thinking something was wrong with my account. Well, after frantically Googling issues and hurrying into settings menus, I managed to fix the issue and force my uncooperative laptop to automatically set the time.

But that did not change the insatiable curiosity that had then crossed my mind. How do TOTPs work? Can I build one? Are they present in our lives in more places than just GitHub logins? Is every code I get from a service asking to verify my identity some form of TOTP? Better yet, could I build my own TOTP generator?

I could have just built the TOTP in a single Python file, but I had recently been gifted an Arduino Uno starter kit and I'd fallen in love with the little projects in the included book, especially since a few of them involved an LCD screen. I wondered if I could make a TOTP generator that would send a code to my Arduino, which would then display that code on the LCD screen. 

I immediately started Googling around. I found a few tutorials for a similar project, but they weren't built for my specific hardware, and some were created with an RTC module, which I don't have, and wasn't keen on buying. Also, I realized that I didn't want to build a fully independent hardware token yet, I just wanted a way to display codes on this external device. In practical applications, it could be a way to reduce clutter on my laptop's screen because with this device, I wouldn't need a terminal open to see my one-time passwords. 

So, I changed gears and did some research. Would it be possible to construct the TOTP generator itself in Python, on my local machine, then somehow get it to send the data to the Arduino so that all the little microcontroller had to do was display some numbers? Yes, it was possible, and that's exactly what I did. 

First, I had to navigate the maze of wires that was involved in setting up the LCD screen. In case you're wondering what that looks like, here's a reconstructed diagram from Tinkercad.

![Arduino Uno and LCD screen wiring schematic from Tinkercad](../assets/img/TOTP_Diagram.png)

That round dial in the diagram is a potentiometer, and it's just there to allow you to control the LCD screen's contrast. 


The code for this project was simple. Once plugged in via USB cable, the Arduino begins communicating with my Python script, which is possible thanks to the ``serial`` library. A secret key is shared, which is the seed for the one-time password generation. The Arduino will then send a request to a Python server running on my machine, which triggers the Python script to generate a 6 digit token using the ``totp.now()`` function included in the ``pyotp`` library. The Python script will generate the code, send it to the Arduino, and the Arduino's code will take the response and print it to the LCD screen. A new one will generate every 30 seconds.

To complement the project, I built a simple CLI-based login utility. You enter in a username, then the currently active TOTP. The script verifies the one-time password's integrity by checking a simple .txt file created by the TOTP generator script. The active code is inside this file; the login script just compares it to whatever was entered into the input field, and displays a welcome message if the two fields match. While this method is quite insecure in a production environment, it is but a stepping stone for more complex projects. I'm not quite sure how I will improve it, or what I will use it for, but I had a lot of fun building it and connecting a hardware project to a software project, which was honestly something I've never done before, since I've been creating software-only projects up until now.

If you're interested in the source code or you want to make one of these for yourself, the GitHub repository is [here](https://github.com/Tori-Tech/Arduino-TOTP-Generator).





