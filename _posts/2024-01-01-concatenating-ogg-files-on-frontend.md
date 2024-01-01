---
layout: post
title: "Concatenating ogg vorbis (.ogg) audio files on frontend"
date: 2024-01-02 02:00:00 +0800
categories: code
image: /assets/images/2024-01-02-concatenating-ogg-files-on-browser/cover.png
tags: javascript frontend ogg audio ffmpeg
---

![](/assets/images/2024-01-02-concatenating-ogg-files-on-browser/cover.png)

## Background
When working with the [Azure Text-to-Speech API]({% post_url 2023-10-15-convert-azure-text-to-speech-to-nodejs-stream %}), there is no size limit for the input text, but the resulting output audio is limited to a [maximum length of 10 minutes](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/rest-text-to-speech?tabs=streaming#sample-request-1). If the input exceeds this limit, the output audio will be truncated, causing unexpected issues. This limitation is particularly problematic for scenarios involving long texts, such as whole chapters from ePub ebooks, which often surpass the threshold and result in undesired truncation.

## Working around the output limit
An easy workaround for this limitation is to split the input into 10-minute chunks when making API calls to the Text-to-Speech service. The resulting audio files can then be concatenated into a single file. However, calculating the exact output audio length for a given set of plaintext input can be challenging, especially when dealing with bilingual texts like Chinese and English. In English, words are separated by spaces, making it relatively straightforward to estimate the length by counting number of words. However, in Chinese, characters and words are not separated by spaces, making the issue more complex.

Fortunately, in this particular case, the input text consists of well-formatted paragraphs with new lines (or `<br>` and `<p>` tags in the case of HTML input). A simple approach would be to split the text by newline characters (`\n`) and merge the parts until the maximum character length is reached. We can set a more conservative maximum character length here, as having the audio files shorter than 10 minutes will not affect the final concatenated result.

{% highlight javascript %}

function splitText (text: string, maxLength = 2000) {
  const sections = []
  const words = text.split('\n')
  let currentSection = ''

  for (const word of words) {
    if (currentSection.length + word.length + 1 > maxLength) {
      sections.push(currentSection)
      currentSection = ''
    }
    currentSection += word + '\n'
  }

  sections.push(currentSection.trim())
  return sections
}

{% endhighlight %}


By utilizing this method, the Azure Text-to-Speech API can be called with these split text sections. When making the API call, it's recommended to select [OGG as the audio output type](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/rest-text-to-speech?tabs=streaming#audio-outputs), as OGG files are smaller in size compared to MP3 files. Following these steps, you will have a collection of OGG files, each shorter than 10 minutes.

## Concatenating the audio on the frontend? A challenge

Concatenating audio files is relatively straightforward in a server environment, thanks to powerful tools like [FFmpeg](https://ffmpeg.org/). It involves saving all the intermediate audio files on the server, merging them using FFmpeg, and then storing the resulting file until it is downloaded by the user.

However, in this project, a more challenging approach was taken: merging the OGG audio files on the frontend using JavaScript. This approach eliminates the need for server file storage, simplifies the architecture, reduces costs, and fits more seamlessly into the current API structure powered by Nuxt.js.

The challenge arises when trying to find a browser-side replacement for the powerful FFmpeg [concat function](https://trac.ffmpeg.org/wiki/Concatenate).

## Solution #1: Simple concatenation using `cat`

After conducting some research, it seems that the OGG specification allows for the direct concatenation of two OGG files to create a single OGG file with two logical streams. For example, in a Linux bash environment, one can simply use the `cat` command to achieve this.

{% highlight bash %}

`cat 1.ogg 2.ogg > result.ogg`

{% endhighlight %}

One prerequisite of this approach is each OGG file [must have a unique serial](https://lists.xiph.org/pipermail/ogg-dev/2018-November/001946.html) metadata. Fortunately, it appears that the Azure Text-to-Speech API returns OGG files with randomized serial numbers. Kudos to Azure!

Implementing this in the browser is surprisingly simple using `Blob`.

{% highlight javascript %}

const sections = splitText(text, 2000)
const blobs = []
for (let i = 0; i < sections.length; i++) {
  const text = sections[i]
  const blob = await convertTextToAudio(text)
  blobs.push(blob) // blob is the ogg file of the text sections
}
const blob = new Blob(blobs, { type: 'audio/ogg' })

{% endhighlight %}

However, despite the merged OGG file being playable, most audio players are unable to properly seek within this track. I tested Chrome, VLC player, and Audacity, and none of them could display the correct total duration or seek the track correctly. Since the ability to seek is crucial for our audio ebook use case, this solution is not acceptable.

## Solution #2: Web Audio and MediaStream Recording API

When seeking a solution for concatenating OGG files, next thing I do is to ask ChatGPT, and it suggested using the [`AudioContext`](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext) provided by the `Web Audio API`. By using the `decodeAudioData` function to decode each input OGG file, the files can be concatenated using `audioContext.createBuffer` and [`AudioBuffer`](https://developer.mozilla.org/en-US/docs/Web/API/AudioBuffer). It is important to note that this approach assumes that both audio files have the same number of channels and sample rate.

{% highlight javascript %}

// Function to concatenate two OGG files, provided by ChatGPT
function concatenateOGGFiles(file1, file2) {
  return new Promise((resolve, reject) => {
    // Create audio context
    const audioContext = new (window.AudioContext || window.webkitAudioContext)();

    // Load the first file
    const reader1 = new FileReader();
    reader1.onload = function(e) {
      audioContext.decodeAudioData(e.target.result, function(buffer1) {
        // Load the second file
        const reader2 = new FileReader();
        reader2.onload = function(e) {
          audioContext.decodeAudioData(e.target.result, function(buffer2) {
            // Create a new buffer for the concatenated audio
            const length = buffer1.length + buffer2.length;
            const audioBuffer = audioContext.createBuffer(buffer1.numberOfChannels, length, buffer1.sampleRate);

            // Copy the samples from the first buffer
            for (let channel = 0; channel < buffer1.numberOfChannels; channel++) {
              const channelData = buffer1.getChannelData(channel);
              audioBuffer.getChannelData(channel).set(channelData, 0);
            }

            // Copy the samples from the second buffer
            for (let channel = 0; channel < buffer2.numberOfChannels; channel++) {
              const channelData = buffer2.getChannelData(channel);
              audioBuffer.getChannelData(channel).set(channelData, buffer1.length);
            }

            resolve(audioBuffer);
          });
        };
        reader2.onerror = reject;
        reader2.readAsArrayBuffer(file2);
      });
    };
    reader1.onerror = reject;
    reader1.readAsArrayBuffer(file1);
  });
}

{% endhighlight %}

One issue remains: how to save the concatenated `AudioBuffer` as an OGG file. It turns out an other modern browser API, the `MediaStream Recording API` would come in handy. The `AudioBuffer` can be sent to a [`MediaRecorder`](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder), and the result can be saved as an OGG file after the recording ends. Here is a snippet of [example code from MDN's `MediaRecorder` page](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder/dataavailable_event#example).

{% highlight javascript %}
const chunks = [];

mediaRecorder.onstop = (e) => {
  console.log("data available after MediaRecorder.stop() called.");

  const audio = document.createElement("audio");
  audio.controls = true;
  const blob = new Blob(chunks, { type: "audio/ogg; codecs=opus" });
  const audioURL = window.URL.createObjectURL(blob);
  audio.src = audioURL;
  console.log("recorder stopped");
};

mediaRecorder.ondataavailable = (e) => {
  chunks.push(e.data);
};

{% endhighlight %}

Unfortunately, [Chrome does not support saving the file as OGG using MediaRecorder](https://stackoverflow.com/questions/41739837/all-mime-types-supported-by-mediarecorder-in-firefox-and-chrome/42307926#42307926). In fact, only the `.webm` format is supported for audio. While sending an uncompressed `.wav` file is undesirable due to its large size, using `.webm` severely limits the compatibility of the resulting audio file. Since Chrome is a widely used browser, and we cannot ignore this limitation, I ultimately did not implement this approach.

## Final Solution: Bring FFmpeg to the browser side

Due to the absence of FFmpeg on the frontend, we attempted two alternative methods, both of which proved unsuccessful. However, what if we can actually use FFmpeg on the client side? This would resolve all the issues and provide a straightforward solution.

As it turns out, this is indeed possible with [`ffmpeg.wasm`](https://FFmpegwasm.netlify.app/docs/overview/), thanks to powerful WebAssembly technologies that allow running FFmpeg in the browser.

By incorporating FFmpeg into the browser, the problem of concatenating OGG files simplifies to a single FFmpeg `concat` command. The resulting code would be similar to the following snippet. Please note that if you use `vite` as your build system (in my case, Nuxt 3), you may need to [apply additional configuration](https://ffmpegwasm.netlify.app/docs/getting-started/usage#transcode-webm-to-mp4-video) for `baseURL`, `coreURL`, and `wasmURL` due to a CORS issue.

{% highlight javascript %}

export async function concatOggs (files: Blob[]) {
  if (FFmpeg === null) {
    const baseURL = 'https://unpkg.com/@FFmpeg/core@0.12.4/dist/esm'
    FFmpeg = new FFmpeg()
    FFmpeg.on('log', ({ message }) => {
      console.log(message)
    })
    await FFmpeg.load({
      coreURL: await toBlobURL(`${baseURL}/FFmpeg-core.js`, 'text/javascript'),
      wasmURL: await toBlobURL(`${baseURL}/FFmpeg-core.wasm`, 'application/wasm')
    })
  }
  const inputPaths = []
  for (let i = 0; i < files.length; i++) {
    const file = files[i]
    const { name = i.toString() } = file
    FFmpeg.writeFile(name, new Uint8Array(await file.arrayBuffer()))
    inputPaths.push(`file ${name}`)
  }
  await FFmpeg.writeFile('concat_list.txt', inputPaths.join('\n'))
  await FFmpeg.exec(['-f', 'concat', '-safe', '0', '-i', 'concat_list.txt', 'output.ogg'])
  const data = await FFmpeg.readFile('output.ogg')
  return (
    new Blob([data], {
      type: 'audio/ogg'
    })
  )
};

{% endhighlight %}

Once this function is called, the Web worker will handle the rest, and the concatenated OGG file will be available as a downloadable blob. Since FFmpeg recreates the metadata rather than simply concatenating the files, seeking and duration work correctly in the players I tested.

## Additional remarks on FFmpeg license

It's important to note the license issue associated with FFmpeg. FFmpeg is [LGPL licensed](https://www.ffmpeg.org/legal.html), which means that if it is directly distributed as WebAssembly to clients, your entire web application would also need to be distributed under a LGPL-compatible license. While this may not be a significant concern for personal or open-source projects, it may not be applicable in other cases.
