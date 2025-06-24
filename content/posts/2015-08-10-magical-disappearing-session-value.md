---
categories:
- programming
date: "2015-08-10T00:00:00Z"
tags:
- IE
title: Magical disappearing session value
pin: true
---

Fascinating bug: Intermittent, only reproducible on some computers and not others, only a problem in a certain version of Internet Explorer on those computers, affecting server-side behavior, and faux-consistent. (In other words, the behavior could be consistent, but the consistency was due to a confounding variable rather than the obvious cause.)

I work on a web application that uses SES-friendly URLs for nearly everything. This works fine, even though it's implemented at the application framework level and is therefore kind of a hack that browsers might or might not understand. We've run into a few instances of relative links not working correctly in some browsers, but those were easily fixed.

For searching our resources, on the results page we put all of the search criteria into the URL so people can share the URL with others. Since we use the same search functionality for our topic pages, the topic page URLs usually look something like this, running a search in the background by topic and subtopic ID:

```
https://www.example.com/resource/list/topic/1111/subtopic/2222
```

On this topic page, you can also choose to get the presented results in different formats. We started getting bug reports that certain subtopics were presenting the correct resources on the page, but if you clicked to get certain other formats, it would ignore the subtopic and just give you all of the resources under the topic. This behavior would happen on certain subtopics, but not others, under the same topic.

My hypothesis, from experience with the setup, was that our legacy of storing search variables in the session for certain purposes was probably biting us. But how to prove it, and why only one session variable?

I started working with it, and I found out I couldn't reproduce the behavior on my development machine, even with the exact same install of IE11 with the same general settings. After trying the usual "turn it off and on again" type of phone troubleshooting, I headed over to the client's office to borrow a computer.

It turned out that it happened pretty consistently with the borrowed computer. Of course, I was testing against production, and my development environment was on my local machine. I didn't have admin rights to the borrowed computer, so setting up the environment there wasn't going to happen, and getting the two machines to talk would require one more Ethernet connection than I had. Luckily, we've got a decent test server setup that we normally use for acceptance testing, so as long as I could get the same behavior to happen there I could remote into the test server and do the necessary logging there.

I finally found a place where the problem happened consistently, so, reloading the page on the borrowed computer while remoted into the test server, I started logging the session variables in question, working my way back from the endpoint in question. Yep, the session variable for the subtopic wasn't disappearing, but it was null. Checking at the start of the process, session.subtopic was 2222, like it should have been.

Logging all the way through, I discovered something troubling. The session.subtopic value was disappearing during or before an event that had nothing to do with session variables. The usual debugging method of staring at the code while muttering didn't reveal anything, so I went back to the network tab in IE's Developer Tools and reloaded the Ajax call again and again. Still nothing.

We do have two mutually exclusive JavaScript files that could potentially load into that page. Maybe we somehow loaded both of them, occasionally creating a [heisenbug](https://en.wikipedia.org/wiki/Heisenbug) due to precedence? Let's reload the one page where everything worked fine and look for those two JS files. Nope, only one.

Wait, what's that? Way at the end of the list of files going over the network is something that starts with `https://www.example.com/resource/list`.... That's not right.

Full URL:

```
https://www.example.com/resource/list/topic/1111/subtopic/resources/images/icons/favicon.ico
```

Tried it on my computer, and that load never happens. Try it on the client's computer, and it happens pretty regularly, but not all the time.

Searching through the code base, I found a place where I had added the site's favicon months before:

```html
<link rel="shortcut icon" href="resources/images/icons/favicon.ico" type="image/vnd.microsoft.icon" />
```

Not sure where I found it originally, but I'm sure I copied and pasted from somewhere. Unfortunately, my search for the proper code didn't lead me to [rel="shortcut icon" considered harmful](https://mathiasbynens.be/notes/rel-shortcut-icon).

See any problems? Yep, it's a relative link, so it requires the browser to make a guess at the proper place in the URL from which it is relative. Plus, I'd also dropped favicon.ico in the normal root directory default location, so it shouldn't have been necessary at all.

Chrome and Firefox just ignored the whole thing and pulled the favicon.ico from the root. IE followed it, but only sometimes. I'm not sure, and probably never really will be sure, why there was a difference between the client IE and mine, but I suspect mine had cached the file at some point while the client's IE requested it sometimes. The *sometimes* was just consistent enough to appear to happen on certain subtopic values and not others.

So, the effect, many times, was that the page would load with the proper URL and proper information, that it would store in the session variables. The page would then reload into the browser using a URL without a valid subtopic value and ending in a filename. IE would pull all of the data into that presumed file, and then discard it as not being a legitimate .ico file, never actually reloading the visible page. That second run through the process overwrote the session variables with all of the same information, except for that one variable at the end that had an invalid value. Validation would quietly set it to null, since usually the page results and URL would clue you in on what was wrong.

I removed the offending line, and we were all good. Even the client's IE was fine with the root version of the favicon.ico.

I suspect that, if we had a trailing slash after the subtopic value, or anything else less important was in that last position, we would have never noticed it until questioning our server usage.

Conclusion:

* We were often running searches twice for months. I can't imagine how many server resources we were using, even with caching the results.
* I really need to rearchitect the search process to simplify it and remove some of our old behavior. We have better tools now, and there's little excuse for a simple process that uses very different code for the same results.
* Examining the actual traffic more often would have brought this bug to light.
* 90% of the users are using a slightly different system than I am. We need one of those for testing.
* No matter how innocuous it seems, copying and pasting deserves extra testing attention.
* Validation really needs to send a message, even if it seems like it could just be quiet.
