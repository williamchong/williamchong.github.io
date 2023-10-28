---
layout: post
title:  "Convert Google text to speech API result to HTTP streamed response"
date:   2023-10-13 10:02:00 +0800
categories: code
image: /assets/images/2023-10-13-convert-google-text-to-speech-to-nodejs-stream/cover.png
tags: javascript nodejs google cloud
---

![Google cloud text to speech API](/assets/images/2023-10-13-convert-google-text-to-speech-to-nodejs-stream/cover.png)

When using the Google Cloud Text-to-Speech API, the default behavior of the [`synthesizeSpeech()`](https://googleapis.dev/nodejs/text-to-speech/latest/google.cloud.texttospeech.v1.TextToSpeech.html#synthesizeSpeech2) method, as described in the `synthesizeSpeech()` documentation, is to return the [`audioContent`](https://googleapis.dev/nodejs/text-to-speech/latest/google.cloud.texttospeech.v1.ISynthesizeSpeechResponse.html) as a complete buffer.

However, if you want to enable streaming playback for long audios, you can convert the buffer to a file streaming response. To achieve this, you can utilize the [PassThrough](https://nodejs.org/api/stream.html#class-streampassthrough) class from the Node.js Stream API, as outlined in the `PassThrough` documentation.

Here is a sample snippet from a Nuxt 3 project that demonstrates this:

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
