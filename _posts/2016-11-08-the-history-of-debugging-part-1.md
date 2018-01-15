---
layout: single
title: 'The History of Debugging: Part 1'
date: 2016-11-08 23:20:46.000000000 +01:00
type: post
tags: ["Debugging", "History"]
---

<p>Disclaimer: there might never be <a href="http://www.imdb.com/title/tt0082517/" target="_blank">a Part 2</a>.</p>
<p>Together with <a href="https://blog.maartenballiauw.be/" target="_blank">Maarten</a> (who already has <a href="https://blog.maartenballiauw.be/post/2016/10/19/making-net-code-less-allocatey-garbage-collector.html" target="_blank">written some content</a> concerning debugging and monitoring software), we're going to write some posts about the concept of debugging code. And to introduce the topic, why not start with a little bit of (mostly personal) history on the matter?</p>
<!--more-->
<h1>ENIAC</h1>
<p>It's hard to imagine nowadays, but writing software wasn't always literally writing: on the <a href="https://en.wikipedia.org/wiki/ENIAC" target="_blank">ENIAC</a> i.e., the programmers had to make physical connections between the different components, replacing burnt out vacuum tubes as they progressed. They did actually write the program on paper first though, and did their very best to run through it <em>step by step</em> before starting to program the actual computer. And even then, they went through the program again: even the ENIAC already supported <em>step-through debugging</em>!</p>
<p><a href="/assets/eniac.jpg"><img class="aligncenter size-full wp-image-565" src="{{ site.baseurl }}/assets/eniac.jpg" alt="ENIAC" width="994" height="677" /></a></p>


<h1>My First Program and QBASIC</h1>
<p>When I was in the Sixth grade, at the age of eleven, I got the chance to learn to program in QBASIC on a 386 PC, running MS-DOS 6.0, during breaks. Well, that and the occasional <a href="https://en.wikipedia.org/wiki/Out_Run" target="_blank">Sega OutRun</a> <a href="https://www.youtube.com/watch?v=Y0qFU39Y-M0" target="_blank">game</a>.<br />
But programming really caught my interest and I quickly went through the assignments my teacher had created. Meanwhile, we got a PC at home, so I could continue learning to program. In the end, I'd written several game clones like <a href="https://en.wikipedia.org/wiki/Breakout_(video_game)" target="_blank">Breakout</a>, Tetris, etc.</p>
<p>My programming skills, however, were quite limited: it took me a while to figure out that I could use subroutines (GOSUB) instead of line numbers and the now dreaded GOTO. And as far as debugging was concerned, I was using the oldest trick in the book: a lot of PRINT statements to know the value of certain variables at specific steps in the code. I'm not kidding when I say that I only recently found out that QBASIC supported <em>step-through debugging</em> and setting <em>breakpoints</em> in code: nobody told me these things when I started programming and I didn't know English well enough to figure this out myself.</p>
<p>Moral of the story so far: you will learn to appreciate any help the debugger tools provide once you had to resort to printing variable values.</p>
<h1>Windows 3.11 and Visual Basic 4.0</h1>
<p>As I got older, I stumbled upon VB4 which allowed me to create software with Forms. My own Windows in Windows! And because I started getting English class at school, I also understood more about the options I had to create software. Finally I discovered things like <em>breakpoints</em>, <em>watches</em>, <em>step-through debugging</em> and Visual Basics way of <em>error handling</em>. Aside from games, I could also start writing more useful software for family and friends.<br />
The summum? Writing a new UI which emulated the Windows 95 look and feel to convince my dad to upgrade the PC (it needed 8 megs of RAM instead of 4) and to buy that shiny Windows 95 upgrade.</p>
<h1>VB6, C/C++, PHP, ...</h1>
<p class="">I also upgraded to Visual Basic 97, then 6.0, started to dabble in C/C++ and wrote my first web applications in PHP. That was a step back in time really. Sure, you could run a <a href="https://en.wikipedia.org/wiki/LAMP_(software_bundle)#WAMP" target="_blank">WAMP</a> server locally to test your code, but you couldn't <em>attach a debugger</em> to that server to debug the code as you went through the application (I'm talking about PHP3/4 here, way before the Zend era). So instead, I had to rely on echoing stuff back into my web pages or logging it and looking at the log files. Or even worse, stop the code from processing at all using <code>die()</code> statements, which typically happened when you couldn't connect to the MySQL database.</p>
<p class="">Maybe we are spoiled today, with the capabilities offered by the different debugger tools available. But then again, are we using them efficiently? Find out in my next post in this series!</p>
