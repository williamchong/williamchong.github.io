---
layout: post
title: "Convert OpenAI API stream to HTTP streamed response"
date: 2023-10-28 04:00:00 +0800
categories: code
image: /assets/images/2023-10-28-openai-api-to-http-streamed-response/poe.png
tags: javascript nodejs openai chatgpt
---

![Asking the AI how to stream its API](/assets/images/2023-10-28-openai-api-to-http-streamed-response/poe.png)

Previously, we covered the implementation of HTTP streamed response for [Google's Text-to-Speech API]({% post_url 2023-10-13-convert-google-text-to-speech-to-nodejs-stream %}) and [Azure Text-to-Speech API]({% post_url 2023-10-15-convert-azure-text-to-speech-to-nodejs-stream %}). In this post, we will explore the same technique for another hot topic: calling the [OpenAI API](https://platform.openai.com/) and streaming ChatGPT responses word by word using HTTP streamed (chunked) responses.

## Background

Why is a streamed response useful in this case? Waiting for ChatGPT to complete its answer is time-consuming. In fact, requesting the `gpt-3.5-16k` model to translate a piece of an article with approximately 400 tokens often requires more than 30 seconds to receive a complete response. If this is implemented in a web app, users will be staring at a loading screen for more than 30 seconds with nothing else to do.

Fortunately, ChatGPT (and most other mainstream generative models) generate responses word by word. They predict the most probable word sequence as an answer based on the input text and existing words. To enhance the user experience, it would be better to display this word-by-word generation process live to the user, allowing them to start reading immediately and feel engaged. This is the standard user experience for most AI programs nowadays.

## Possible Solutions and Why HTTP?

To enable the UI or frontend to show the generative process in real-time, we need to stream the response from the OpenAI API from our backend server to the frontend. OpenAI API and its SDKs [use Server-Sent Events](https://platform.openai.com/docs/api-reference/chat/create) as an approach, while [websockets](https://developer.mozilla.org/docs/Web/API/WebSocke) are another popular technique in this context. However, implementing these techniques requires additional knowledge and libraries.

Is there a simpler way? If we can accomplish this task using only HTTP and XHR, without relying on extra knowledge or libraries, it would be ideal. Fortunately, we can achieve this by utilizing HTTP chunked responses as a long-lived streaming connection. If you are familiar with older techniques, you might recognize this as a form of "long-polling."

## Backend, all about transformation

The OpenAI library supports a stream mode specifically for this purpose. However, [in the sample code](https://github.com/openai/openai-node#streaming-responses), only a blocking "`for await...of`" loop is used to send the output to stdout.

{% highlight javascript %}
import OpenAI from 'openai';

const openai = new OpenAI();

async function main() {
const stream = await openai.chat.completions.create({
  model: 'gpt-4',
    messages: [{ role: 'user', content: 'Say this is a test' }],
    stream: true,
  });
  for await (const part of stream) {
    process.stdout.write(part.choices[0]?.delta?.content || '');
  }
}

main();
{% endhighlight %}

To make our API non-blocking and truly streaming, we need to pipe the stream to an HTTP response object instead of using a "`for await...of`" loop. However, in the provided code, we cannot easily achieve that since the output `part` from the stream is not directly useful. What we actually need is `part.choices[0]?.delta?.content || ''`. In previous articles, we used `PassThrough` to connect streamed input and output. However, in this case, we need to use [`Transform`](https://nodejs.org/api/stream.html#class-streamtransform). In fact, PassThrough is just [a `Transform` that does nothing](https://nodejs.org/api/stream.html#class-streampassthrough). Here, we want to transform any output (JSON object) from the stream into words (`part.choices[0]?.delta?.content || ''`) to be displayed to the users.

To use `Transform`, we define the `transform` function parameter and perform the transformation inside this function. Note that since the input piped into `Transform` is an object, we must also set `objectMode: true`. Otherwise, an  `The "chunk" argument must be of type string or an instance of Buffer` error will be thrown when an object is received, as Node.js [expects the input to be string or buffer](https://nodejs.org/api/stream.html#object-mode).

The resulting code is as follows.
{% highlight javascript %}
const response = await openai.chat.completions.create({
  messages: [
    {
      role: 'system',
      content: 'Translate the given blog post to Chinese please'
    },
    { role: 'user', content: text }
  ],
  model: 'gpt-3.5-turbo-16k',
  stream: true
})
const stream = Readable.from(response)
const bufferStream = new Transform({
  objectMode: true,
  transform (chunk, \_, callback) {
    const data = chunk as ChatCompletionChunk
    callback(null, data.choices[0]?.delta?.content || '')
  }
})
stream.pipe(bufferStream)
return sendStream(event, bufferStream)

{% endhighlight %}

## Frontend, wth?

The above code covers the API part. However, to handle a chunked response, we also need special handling on the frontend. Conceptually, it's simple: keep receiving the chunked response and display it as text on the DOM. However, things get more complicated in practice.

{% highlight javascript %}
const { data, error } = await useFetch('/api/translate', {
  method: 'POST',
  body: {
    text: t,
    language: translateLocale.value,
    type
  },
  responseType: 'stream'
})
if (error.value) { throw error.value }
const stream = data.value as ReadableStream
{% endhighlight %}

In my case, I'm using Nuxt 3 to build my web app. The official way to make XHR calls in Nuxt 3 is by using [`useFetch`](https://nuxt.com/docs/api/composables/use-fetch). Setting `responseType: 'stream'` allows the response to be in stream mode, which is straightforward. However, the `data` returned in this case is a [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) object. And here's where the trouble begins.

Reading a `ReadableStream` in the browser environment is much more challenging compared to using a Node.js stream. Fortunately, Mozilla understands this pain and provides a [specific document page](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_streams#consuming_a_fetch_using_asynchronous_iteration) with hints, especially regarding using a simple `for await...of` loop to save us the trouble of reading the stream.

In an ideal world, the code would look like this:
{% highlight javascript %}
// in an ideal world
const stream = data.value as ReadableStream

for await (const chunk of stream) {
  // translateOutput.value is binded to a textarea as UI output
  translateOutput.value += value
}
{% endhighlight %}

However, this throws a "`stream is not async iterable`" error on Chrome. What's going on? It turns out to be [a bug in chrome](https://bugs.chromium.org/p/chromium/issues/detail?id=929585). It's frustrating, but at least the bug report provides a workaround.

One last piece of the puzzle is that the received chunk is in bytes, represented as a `Uint8Array` in modern browsers. We need to convert it back to a UTF-8 string. In Node.js, this is trivial with just a `toString('utf8')` call. However, in the browser environment, we need some help from the [`TextDecoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder) class, which handles byte decoding. If you're targeting older browsers, you might need a polyfill for `TextDecoder`.

Here's our final frontend code:

{% highlight javascript %}
const { data, error } = await useFetch('/api/translate', {
  method: 'POST',
  body: {
    text: t,
    language: translateLocale.value,
    type
  },
  responseType: 'stream'
})
if (error.value) { throw error.value }
const stream = data.value as ReadableStream
// should use await-for-of if not for https://bugs.chromium.org/p/chromium/issues/detail?id=929585
const reader = stream.getReader()
let done = false
while (!done) {
  const chunk = await reader.read()
  done = chunk.done
  const value = chunk.value as Uint8Array
  // translateOutput.value is binded to a textarea as UI output
  translateOutput.value += new TextDecoder().decode(value)
}
{% endhighlight %}

## Done... or not yet? NGINX and load balancers
The result works well, but then I encountered an additional infrastructure-related issue. The code works fine on my local machine, but on the production deployment, the frontend doesn't receive anything from the stream. The text only shows up once the whole HTTP request is complete without chunking. This defeats the purpose of my work.

Fortunately, I quickly identified the issue. It turns out our NGINX proxy is [buffering chunked response by default](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering). Simply adding `proxy_buffering off;` fixed the issue. It's also good to know that neither Cloudflare nor the Google Cloud load balancer does this by default. Otherwise, I would have had a major headache.

Another issue arose when we had to [adjust the timeout for our Google Cloud HTTP load balancer]((https://cloud.google.com/load-balancing/docs/https#websocket_support)) running as our Kubernetes ingress, from 60 seconds to 1800 seconds. This timeout affects all HTTP connections in the cluster, regardless of whether they are HTTP chunked responses, server-sent events, or websockets.

## Done (Real)

This task turned out to be more of a hassle than I initially thought. However, we now have a properly streaming chatbot using only HTTP protocols, without the need for extra libraries.
