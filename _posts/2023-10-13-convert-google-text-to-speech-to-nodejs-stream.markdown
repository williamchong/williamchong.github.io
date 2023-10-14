---
layout: post
title:  "Convert Google text to speech API result to streamed HTTP response"
date:   2023-10-13 10:02:00 +0800
categories: code
image: /assets/images/2023-10-13-convert-google-text-to-speech-to-nodejs-stream/cover.png
tags: javascript nodejs google cloud
---

![Google cloud text to speech API](/assets/images/2023-10-13-convert-google-text-to-speech-to-nodejs-stream/cover.png)

When using Google Cloud's Text-to-Speech API, the default behavior of the [synthesizeSpeech()](https://googleapis.dev/nodejs/text-to-speech/latest/google.cloud.texttospeech.v1.TextToSpeech.html#synthesizeSpeech2) method is to return the [audioContent](https://googleapis.dev/nodejs/text-to-speech/latest/google.cloud.texttospeech.v1.ISynthesizeSpeechResponse.html) as a complete buffer.

However, if you want to enable streaming playback for long audios, you can convert the buffer to a file streaming response. This can be achieved by converting the buffer to a Node.js `stream` object using the [PassThrough](https://nodejs.org/api/stream.html#class-streampassthrough) class from the Node.js Stream API.

The sample bug below is a snippet from a Nuxt 3 project.

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
