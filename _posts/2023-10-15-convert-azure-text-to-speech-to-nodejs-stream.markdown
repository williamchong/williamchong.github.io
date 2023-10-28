---
layout: post
title:  "Convert Azure text to speech API result to HTTP streamed response"
date:   2023-10-15 02:40:00 +0800
categories: code
image: /assets/images/2023-10-15-convert-azure-text-to-speech-to-nodejs-stream/azure.png
tags: javascript nodejs azure
---
![Azure text to speech API](/assets/images/2023-10-15-convert-azure-text-to-speech-to-nodejs-stream/azure.png)

Previously, we discussed the usage of 「Google's Text-to-Speech API]({% post_url 2023-10-13-convert-google-text-to-speech-to-nodejs-stream %})I in a Node.js stream. Similarly, when using the [Azure version of the API](https://azure.microsoft.com/en-us/products/ai-services/text-to-speech), it might be preferable to receive a streamed response over a buffer.

Unlike Google's API, which only supports the entire buffer as the response format, the Azure API allows us to set different output options using the [audioConfig](https://learn.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/audioconfig?view=azure-node-latest) class, as described in the `audioConfig` documentation. These options include `fromAudioFileOutput`, `fromDefaultSpeakerOutput`, and `fromStreamOutput`. It's important to note that `audioConfig` is also used for input configuration in other scenarios, but we won't cover that here.

The official Azure API guide provides [a sample code snippet](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/get-started-text-to-speech?tabs=macos%2Cterminal&pivots=programming-language-javascript), which demonstrates the usage of fromAudioFileOutput. This approach writes the audio output as a file in the file system. However, this method has two drawbacks: first, similar to the entire buffer approach, we would need to wait for the file download to complete before proceeding, and second, after the response is sent, the files would need to be manually removed.

Fortunately, Azure also provides the [`fromStreamOutput`](https://learn.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/audioconfig?view=azure-node-latest#microsoft-cognitiveservices-speech-sdk-audioconfig-fromstreamoutput) function, as documented here, which allows us to use a stream as the output. Two types of streams are available: `PullAudioOutputStream` and `PushAudioOutputStream`. The pull stream requires the caller to invoke its `read()` method to obtain data, while the push stream uses the `write()` and `close()` methods of the callback object. In this case, we will use the PushAudioOutputStream and mimic the behavior of a write stream by utilizing [PassThrough](https://nodejs.org/api/stream.html#class-streampassthrough), as we did previously. It's important to note that while PassThrough doesn't have a `close()` method, calling `end()` serves the same purpose in this context.


Additionally, the [speakTextAsync](https://learn.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/speechsynthesizer?view=azure-node-latest#microsoft-cognitiveservices-speech-sdk-speechsynthesizer-speaktextasync) function, which utilizes the speakTextAsync method of the SpeechSynthesizer class, accepts callback functions instead of returning a promise. To provide convenience, a simple promise wrapper is used.

Below is a code snippet from a Nuxt 3 project that demonstrates the integration with Azure's Text-to-Speech API:

{% highlight javascript %}
import sdk, { SpeechSynthesizer } from "microsoft-cognitiveservices-speech-sdk";
import { PassThrough } from "stream";

function speakTextAsync(synthesizer, text) {
  return new Promise((resolve, reject) => {
    synthesizer.speakTextAsync(text,
      function (result) {
        synthesizer.close();
        resolve(result)
      },
      function (err) {
        synthesizer.close();
        reject(err);
    })
  })
}

// ...

const bufferStream = new PassThrough();
const stream = sdk.PushAudioOutputStream.create({
  write: (a) => bufferStream.write(Buffer.from(a)),
  close: () => bufferStream.end(),
});
const audioConfig = sdk.AudioConfig.fromStreamOutput(stream);
const speechConfig = sdk.SpeechConfig.fromSubscription(subscriptionKey, serviceRegion);

speechConfig.speechSynthesisVoiceName = LANG_TO_NAME[language];
speechConfig.speechSynthesisOutputFormat = sdk.SpeechSynthesisOutputFormat.Ogg16Khz16BitMonoOpus;

const synthesizer = new sdk.SpeechSynthesizer(speechConfig, audioConfig);
synthesizer.SynthesisCanceled = function (s, e) {
  const cancellationDetails = sdk.CancellationDetails.fromResult(e.result);
  let str = "(cancel) Reason: " + sdk.CancellationReason[cancellationDetails.reason];
  if (cancellationDetails.reason === sdk.CancellationReason.Error) {
      str += ": " + e.result.errorDetails;
  }
  console.error(str);
};

await speakTextAsync(synthesizer, text)
setHeader(event, 'content-type', 'audio/ogg; codecs=opus');
setHeader(event, 'content-disposition', 'attachment; filename="speech.ogg"');
return sendStream(event, bufferStream);

{% endhighlight %}
