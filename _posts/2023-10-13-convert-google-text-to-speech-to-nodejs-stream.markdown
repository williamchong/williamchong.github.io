---
layout: post
title:  "Convert Google text to speech API result to streamed HTTP response"
date:   2023-10-13 10:02:00 +0800
categories: code
---

When using google cloud's text to speech API, by default [synthesizeSpeech()](https://googleapis.dev/nodejs/text-to-speech/latest/google.cloud.texttospeech.v1.TextToSpeech.html#synthesizeSpeech2) returns the [audioContent](https://googleapis.dev/nodejs/text-to-speech/latest/google.cloud.texttospeech.v1.ISynthesizeSpeechResponse.html) as a whole `buffer`. I would like to convert the buffer to a file streaming response, allowing stream playback for long audios. We can easily convert the `buffer` to node.js `stream` accepted using [PassThrough](https://nodejs.org/api/stream.html#class-streampassthrough).

The sample bug below is a snippet from a nuxt 3 project.

{% highlight javascript %}
import { TextToSpeechClient } from "@google-cloud/text-to-speech/build/src/v1";

// ...

const request = {
  input: { text },
  voice: { languageCode: language, ssmlGender: "NEUTRAL" },
  audioConfig: { audioEncoding: "OGG_OPUS" },
};
const [response] = await client.synthesizeSpeech(request);
// response.audioContent is a Buffer object
if (!response.audioContent) return { error: "No audio content" };

// PassThrough is both a write and read stream
const bufferStream = new PassThrough();
// set the whole buffer content as PassThrough input.
bufferStream.end(Buffer.from(response.audioContent));

// set mime types
setHeader(event, 'content-type', 'audio/ogg; codecs=opus');

// send the PassThrough output as HTTP streamed response
return sendStream(event, bufferStream);

{% endhighlight %}
