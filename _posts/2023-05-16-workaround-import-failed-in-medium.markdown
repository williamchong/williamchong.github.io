---
layout: post
title: "How to workaround “Import failed” error in Medium with debugger"
date: 2023-05-16 00:00:00 +0800
categories: debug
image: /assets/images/2023-05-16-workaround-import-failed-in-medium/1.webp
tags: medium javascript debugger debug
---
### Hacking the “Import your story” in debugger for proper backdate

Originally [posted in LikeCoin medium publication](https://medium.com/likecoin/how-i-workaround-import-failed-error-in-medium-552eb63b25ec).

---

## Import story tool

Medium has a powerful feature that allows you to import websites you own into stories. All you need to do is to provide an URL then press Import. [Check it out if you haven’t tried it before](https://medium.com/p/import).

![Import story tool](/assets/images/2023-05-16-workaround-import-failed-in-medium/1.webp)

You might ask “Why should we use import though?”. The Medium editor is so easy to use and powerful (kudos to the editor developers) that it can handle pasting from external sources very easily. It often takes a simple copy-and-paste to post any articles from my WordPress site to Medium.

However one significant difference is the “import story” feature [parses the published date and canonical link of the original website](https://help.medium.com/hc/en-us/articles/214550207-Importing-a-post-to-Medium), then sets them accordingly in the Medium story. On the other hand, you cannot change the published date of any manually pasted story. The published date would be set as the time you post the pasted story in Medium.

Backdating the published date is an important issue when you are trying to sync articles in batches from [existing websites](https://blog.like.co) to a [Medium publication](https://medium.com/likecoin). You don’t want to bomb subscribers with notifications of stories that are months old, or flood the publication page with post from 2022.

---

## Import fail!

So when I see this error when importing posts from our [WordPress site](https://blog.like.co), I know I am screwed.

![Import error with no useful message](/assets/images/2023-05-16-workaround-import-failed-in-medium/2.webp)

There is no useful error message as in why the import failed. Medium document simply states that [if the service fails, it fails](https://help.medium.com/hc/en-us/articles/360033931713), and tells you to paste the content instead. As mentioned earlier, it is not feasible for me to manually post all stories without backdating. A simple Google search reveals that [it is possible to force a backdate](https://liaogg.medium.com/backdating-your-medium-posts-2022-c9532e9993a3), but it requires using a placeholder page containing the published date metadata. In the case of articles, this can be accomplished by pushing the placeholder page to a Github repository.

As a developer, I am too lazy to host a page and manually set the dates for each post I need to import. Lets try digging into the Medium import error page and see if we can find out the actual cause of the failed import. We will be using Chrome’s developer tool.

![Chrome developer console of a failed import page](/assets/images/2023-05-16-workaround-import-failed-in-medium/3.webp)

First thing one would look at would be the console, any JavaScript or API call errors should be shown here. However as seen in the image, only some boring message about CSP is shown, so no luck in the console.

Second thing to look for would be in the Network tab. Since Medium uses a third part service for parsing and importing external website, one would expect some external API is called. We can filter the network request to “Fetch/XHR” to only show API calls, and see if anything interesting shows up. However, there is no failed HTTP requests. Most of the request are analytics events. By inspecting the payload and response one by one though, a particular API call seems interesting.

---

## oh-noes

![oh-noes](/assets/images/2023-05-16-workaround-import-failed-in-medium/4.webp)

The endpoint is called `/oh-noes` and the request payload looks exactly like a Javascript error stack. This seems to be some home brew error logging API (we all know Sentry is expensive). The `stack` value inside the payload points to an “`Import Error`” throw by the Javascript file `/main-posters.bundle.${hash}.js` , the exactly location of the file and the line number is also shown. By digging into the source of this file, we can trace the origin of the import error that is troubling us. To view the source of js in Chrome, go to `Sources` tab of the developer tools, then find the js file according to the path shown in the `stack` payload.

---

## Debugger

![JavaScript source](/assets/images/2023-05-16-workaround-import-failed-in-medium/5.webp)

It is a obfuscated JavaScript file, as expected in most modern web application. All the variable names are shortened, functions names and structure are messed up for improved size and performance. Let us just skip straight to the interesting part by searching the function name `QRa` and line number mentioned in above error payload.

![The interesting lines](/assets/images/2023-05-16-workaround-import-failed-in-medium/6.webp)

Finally something promising show up. We see words like “`postHTML`”, also a “`errorCode`” that is set to 400, which probably hints it is a HTTP error code. Going a few more lines below allow us to see how Medium show different error messages for some error codes it encounters.

![Error cases](/assets/images/2023-05-16-workaround-import-failed-in-medium/7.webp)

As we can see above, there are three kinds of import error. One for 400/404 error, one for 403/500/504 error, and one that catches all error. We can assume `errCode` is the HTTP error code encountered by the importer when crawling the target URL. Unfortunately, the error message we see in our import error page is the catch-all case. To understand the actual `errCode` for our case, we would want to know about the stat of `a.Ph` variable during our import. To achieve this, we can set a breakpoint on `QRa` .

![State of the program when it is paused](/assets/images/2023-05-16-workaround-import-failed-in-medium/8.webp)

After setting a breakpoint, retry the import flow by refreshing the browser. The page execution will pause when it reaches the breakpoint we set. On the left we can see where the JavaScript execution was paused; on the right we can see the content(state) of all the variables when the JavaScript reaches the breakpoint. The variable we are interested in is `a.Ph` (note that it is case sensitive).

![Content of a.Ph variable](/assets/images/2023-05-16-workaround-import-failed-in-medium/9.webp)

Unfortunately we can see the `errorCode` is 0, which means we don’t know why the import fail. However a very interesting observation is that all the fields except `postHTML` is properly filled. As we can see in the source, having `a.Ph.postHTML` empty would throw us into an error case. What if we actually fill in some random text for `postHTML` here? Actually we can do that!

---

## Hacking values in debugger

![Edit the value of postHTML in debugger](/assets/images/2023-05-16-workaround-import-failed-in-medium/10.webp)

Double click on the `postHTML` key, it would become editable. Type in a random text enclosed with `""` for it to be a proper string. Continue the page JavaScript execution by pressing continue in the Chrome developer tool popup.

![Succeed! (or not?)](/assets/images/2023-05-16-workaround-import-failed-in-medium/11.webp)

---

## Its' working, or not?

Voila! The error is gone! But what do we actually get as the result?

![postHTML value is just post content](/assets/images/2023-05-16-workaround-import-failed-in-medium/12.webp)

The text we just entered in `postHTML` would show up as the story content!

Well thats not ideal for import, but remember that:

The importer actually successfully crawled all the metadata required for import, except `postHTML` . That can be seen in debugger.
Medium editor is very powerful at handling pasted content
Lets try copy and pasting the content from the original site…

![Pasted result](/assets/images/2023-05-16-workaround-import-failed-in-medium/13.webp)

Perfect. The best part of this is the publication date and canonical link would automatically be set according to the target URL. No pushing and editing needed!

---

## Result

The result of importing multiple article from 2022 is shown below. Welcome to [browse our publication](https://medium.com/likecoin) or our [more updated site](https://blog.like.co).

![All the imports with proper publish date](/assets/images/2023-05-16-workaround-import-failed-in-medium/14.webp)

Shoutout to Medium for the powerful and friendly story editor. But it would be nicer **if the import story features can show more useful error message**!

## TL;DR

if your import fails with unknown error, but you really need those backdates:

- Import URL and fail once
- Check the JS source of Import Error by viewing `/oh-noes` payload
- Breakpoint on the origin of the Import Error
- Set a random non-empty `postHTML` to bypass the error screen
- Paste back the actual content
- Publish!
