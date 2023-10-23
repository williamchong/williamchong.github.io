---
layout: post
title:  "Convert Azure text to speech API result to streamed response"
date:   2023-10-15 02:40:00 +0800
categories: code
image: /assets/images/2023-10-15-convert-azure-text-to-speech-to-nodejs-stream/azure.png
tags: javascript nodejs azure
---
![Azure text to speech API](/assets/images/2023-10-15-convert-azure-text-to-speech-to-nodejs-stream/azure.png)

Previously, we covered [Google's Text-to-Speech API]({% post_url 2023-10-13-convert-google-text-to-speech-to-nodejs-stream %}). Similarly, when using the [Azure version of the API](https://azure.microsoft.com/en-us/products/ai-services/text-to-speech), we might prefer to receive a streamed response instead of a buffer.

Unlike Google, which only supports the whole buffer as the response format, Azure API allows us to set different [audioConfig](https://learn.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/audioconfig?view=azure-node-latest) options as output, such as `fromAudioFileOutput`, `fromDefaultSpeakerOutput` and `fromStreamOutput`. It's worth noting that `audioConfig` is also used for output configuration in other instances, but we won't cover that here.

The official API guide in Azure provides a [sample](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/get-started-text-to-speech?tabs=macos%2Cterminal&pivots=programming-language-javascript) using `fromAudioFileOutput`, which writes the audio output as a file in a filesystem. However, this approach can be inconvenient for two reasons: first, similar to the whole buffer approach, we would need to wait for the file to finish downloading before proceeding, and second, after the response is sent, the files would need to be manually removed.

Fortunately, Azure also provides [fromStreamOutput](https://learn.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/audioconfig?view=azure-node-latest#microsoft-cognitiveservices-speech-sdk-audioconfig-fromstreamoutput) function, which uses a stream as the output. There are two types of streams available: `PullAudioOutputStream` and `PushAudioOutputStream`. The pull stream requires the caller to invoke its `read()` method to obtain data, while the push stream, on the other hand, uses the `write()` and `close()` methods of the `callback` object. To mimic the behavior of a write stream and utilize `PassThrough`, as we did last time, we will use `PushAudioOutputStream` here. It's important to note that [PassThrough](https://nodejs.org/api/stream.html#class-streampassthrough) doesn't have a `close()` method, but calling `end()` serves the same purpose in this context.

Additionally, [speakTextAsync](https://learn.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/speechsynthesizer?view=azure-node-latest#microsoft-cognitiveservices-speech-sdk-speechsynthesizer-speaktextasync) accepts callback functions instead of returning a promise. For convenience, a simple promise wrapper is used.

Below is a sample code snippet from a Nuxt 3 project,

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
