# Pausing iframe media when not rendered

Authors: [Gabriel Brito](https://github.com/gabrielsanbrito), [Steve Becker](https://github.com/SteveBeckerMSFT), [Sunggook Chue](https://github.com/sunggook), Ravikiran Ramachandra

## Status of this Document
This document is a starting point for engaging the community and standards bodies in developing collaborative solutions fit for standardization. As the solutions to problems described in this document progress along the standards-track, we will retain this document as an archive and use this section to keep the community up-to-date with the most current standards venue and content location of future work and discussions.
* This document status: **`ACTIVE`**
* Expected venue: [Web Hypertext Application Technology Working Group (WHATWG)](https://whatwg.org/)
* Current version: **https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/IframeMediaPause/iframe_media_pausing.md**

## Introduction

Web applications that host embedded media content via iframes may wish to respond to application input by temporarily hiding the media content. These applications may not want to unload the entire iframe when it's not rendered since it could generate user-perceptible performance and experience issues when showing the media content again. At the same time, the user could have a negative experience if the media continues to play and emit audio when not rendered. This proposal aims to provide web applications with the ability to control embedded media content in such a way that guarantees their users have a good experience when the iframe's render status is changed.

## Goals

Propose a mechanism to allow embedder documents to limitedly control embedded iframe media playback based on whether the embedded iframe is [rendered](https://html.spec.whatwg.org/multipage/rendering.html#being-rendered) or not:
- When the iframe is not rendered, the embedder is able to pause the iframe media playback; and
-  When the iframe becomes rendered again, the embedder is able to resume the iframe media playback.

## Non-Goals
It is not a goal of this proposal to allow embedders to arbitrarily control when to play, pause, stop, resume, etc, the media playback of a rendered iframe.

## Use Cases
There are scenarios where a website might want to just not render an iframe. For example:
- A website, in response to an user action, might decide to temporarily not show an iframe that is playing media. However, since it is not possible to mute it, the only option is for the website to remove the iframe completely from the DOM and recreate it from scratch when it should be visible again. Since the embedded iframe can also load many resources, the iframe recreation operation might make the web page slow and spend resources unnecessarily.

## Proposed solution: media-playback-while-not-rendered Permission Policy

We propose creating a new "media-playback-while-not-rendered" [Permission Policy](https://www.w3.org/TR/permissions-policy/) that would pause any media being played by iframes which are not currently [rendered](https://html.spec.whatwg.org/multipage/rendering.html#being-rendered). This would apply whenever the iframe’s `"display"` CSS property is set to `"none"`.

This policy will have a default value of '*', meaning that all of the nested iframes are allowed to play media when not rendered. The example below show how this permission policy could be used to prevent all the nested iframes from playing media. By doing it this way, even other iframes embedded by "foo.media.com" shouldn’t be allowed to play media if not rendered.

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>iframe pausing</title>
  </head>
  <body>
    <iframe src="https://foo.media.com" allow="media-playback-while-not-rendered 'none'"></iframe>
  </body>
</html>
```

Similarly, the top-level document is also capable of setting this policy on itself by setting the `Permissions-Policy` HTTP header. In the example below, lets' consider a top-level document served by `example.com`. Given the current `Permissions-Policy` HTTP header setup, only iframes that have the same origin as the top-level document (`example.com`) will be able to enable the `media-playback-while-not-rendered` policy.

`example.com`:

```HTTP
Permissions-Policy: media-playback-while-not-rendered=(self)
```

```HTML
<iframe src="https://foo.media.com" allow="media-playback-while-not-rendered 'none'"></iframe>
```

In this case, `example.com` serves a document that embeds an iframe with a document from `https://foo.media.com`. Since the HTTP header only allows documents from `https://example.com` to inherit `media-playback-while-not-rendered`, the iframe will not be able to use the feature.

In the past, the ["execution-while-not-rendered" and "execution-while-out-of-viewport"](https://wicg.github.io/page-lifecycle/#feature-policies) permission policies have been proposed as additions to the Page Lifecycle draft specification. However, these policies freeze all iframe JavaScript execution when not rendered, which is not desirable for the featured use case. Moreover, this proposal has not been adopted or standardized.

## Interoperability with other Web API's

Given that there exists many ways for a website to render audio in the broader web platform, this proposal has points of contact with many API's. To be more specific, there are two scenarios where this interaction might happen:
- Scenario 1: When the iframe is not rendered and it attempts to play audio; and
- Scenario 2: When the iframe is already playing audio and stops rendering. Once rendering again, to resume playback, we recommend that websites wait for a new user gesture to do so.

For the first scenario, the APIs should behave as if they didn't have permission in the first place to play audio. Like when the [`autoplay` permission policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Permissions-Policy/autoplay) is set to `'none'` for an iframe. Regarding the second scenario, we propose that the API's behave as if the user had paused the media playback.

The following subsections covers how this proposal could interact with Web APIs that render audio.

### HTMLMediaElements

HTMLMediaElement media playback is started and paused, respectively, with the [`play`](https://html.spec.whatwg.org/multipage/media.html#dom-media-play) and [`pause`](https://html.spec.whatwg.org/multipage/media.html#dom-media-pause) methods. For scenario 1, the media element shouldn't be [allowed to play](https://html.spec.whatwg.org/multipage/media.html#allowed-to-play) and `play` should return a promise rejected with `"NotAllowedError"`. In this case, the website could easily handle this like shown below.

```js
let startPlayPromise = videoElem.play();

if (startPlayPromise !== undefined) {
  startPlayPromise
    .then(() => {
      // Start whatever you need to do only after playback
      // has begun.
    })
    .catch((error) => {
      if (error.name === "NotAllowedError") {
        showPlayButton(videoElem);
      } else {
        // Handle a load or playback error
      }
    });
}
```
<*Snippet extracted from [MDN](https://developer.mozilla.org/en-US/docs/Web/Media/Autoplay_guide)*>

For the scenario 2, when the iframe is not rendered anymore, the media should be [paused](https://html.spec.whatwg.org/multipage/media.html#dom-media-pause) and a [`pause`](https://html.spec.whatwg.org/multipage/media.html#event-media-pause) event should be dispatched.  

### Web Audio API

The Web Audio API renders audio through an [AudioContext](https://webaudio.github.io/web-audio-api/#AudioContext) object. For scenario 1, if the iframe is not rendered, attempting to start audio from an `AudioContext` shouldn't output anything and put it into a suspended state. It would be recommended for the iframe to wait for a new user interaction event to resume playback - e.g., `click`.

```js
// AudioContext being create in a not rendered iframe, where
// media-playback-while-not-rendered is not allowed. 
let audioCtx = new AudioContext();
let oscillator = audioCtx.createOscillator();
oscillator.connect(audioCtx.destination);

const resume_button = document.querySelector("resume_button");
resume_button.addEventListener('click', () => {
  if (audioCtx.state === "suspended") {
    audioCtx.resume().then(() => {
      console.log("Context resumed");
    });
  }
})

oscillator.start(0);
// should print 'suspended'
console.log(audioCtx.state)
```

Similarly, for scenario 2, when the iframe is not rendered, the audio context state should change to `'suspended'` and the website can monitor this by listening to the [`statechange`](https://webaudio.github.io/web-audio-api/#eventdef-baseaudiocontext-statechange) event. Then, when the iframe is rendered again, it should wait for a new user interaction, like in the first scenario, to resume playback.

### Web Speech API

The [Web Speech API](https://wicg.github.io/speech-api/) proposes a [SpeechSynthesis interface](https://wicg.github.io/speech-api/#tts-section). The latter interface allows websites to create text-to-speech output by calling [`window.speechSynthesis.speak`](https://wicg.github.io/speech-api/#dom-speechsynthesis-speak) with a [`SpeechSynthesisUtterance`](https://wicg.github.io/speech-api/#speechsynthesisutterance), which represents the text-to-be-said.

For both scenarios, the iframe should listen for utterance errors when calling `window.speechSynthesis.speak`. For scenario 1 it should fail with a [`"not-allowed"`](https://wicg.github.io/speech-api/#dom-speechsynthesiserrorcode-not-allowed) SpeechSyntesis error; and, for scenario 2, it should fail with an [`"interrupted"`](https://wicg.github.io/speech-api/#dom-speechsynthesiserrorcode-interrupted) error.

```js
let utterance = new SpeechSynthesisUtterance('blabla');

utterance.addEventListener('error', (event) => {
  if (event.error === "not-allowed") {
    console.log("iframe is not rendered yet");
  } else if (event.error === "interrupted") {
    console.log("iframe was hidden during speak call");
  }
})

window.speechSynthesis.speak(utterance);
```

### Interoperability with autoplay

If an iframe with the permission policy property set to `allow="media-playback-while-not-rendered 'none'; autoplay *"` stops being rendered, the `media-playback-while-not-rendered` permission policy should take precedence and no media should be played while it remains not rendered. The same behavior should happen if the [`autoplay`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/audio#autoplay) property has been set in a HTMLMediaElement 

### Interoperability with `execution-while-not-rendered` and `execution-while-out-of-viewport`

Both `execution-while-not-rendered` and `execution-while-out-of-viewport` permission policies should take precedence over `media-playback-while-not-rendered`. Therefore, in the case that we have an iframe with colliding permissions for the same origin, `media-playback-while-not-rendered` should only be considered if the iframe is allowed to execute. The user agent should perform the following checks:

1. If the origin is not [allowed to use](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#allowed-to-use) the [`"execution-while-not-rendered"`](https://wicg.github.io/page-lifecycle/#execution-while-not-rendered) feature, then:
    1. If the iframe is not [being rendered](https://html.spec.whatwg.org/multipage/rendering.html#being-rendered), freeze execution of the iframe context and return. 
2. If the origin is not [allowed to use](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#allowed-to-use) the [`"execution-while-out-of-viewport"`](https://wicg.github.io/page-lifecycle/#execution-while-out-of-viewport) feature, then:
    1. If the iframe does not intersect the [viewport](https://www.w3.org/TR/CSS2/visuren.html#viewport), freeze execution of the iframe context and return.
3. If the origin is not [allowed to use](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#allowed-to-use) the [`"media-playback-while-not-rendered"`](#proposed-solution-media-playback-while-not-rendered-permission-policy) feature, then:
    1. If the iframe is not [being rendered](https://html.spec.whatwg.org/multipage/rendering.html#being-rendered), pause all media playback from the iframe context and return.

## Alternative Solutions

This section exposes some of the alternative solutions that we came across before coming up with the chosen proposal.

### Add a "muted" attribute to the HTMLIFrameElement 

Similarly to the [HTMLMediaElement.muted](https://html.spec.whatwg.org/multipage/media.html#htmlmediaelement) attribute, the [HTMLIFrameElement](https://www.w3.org/TR/2011/WD-html5-20110525/the-iframe-element.html#the-iframe-element) could have a `muted` boolean attribute. Whenever it is set, all the HTMLMediaElements – i.e., audio and video elements – embedded in the nested iframes should also be muted. As shown in the example below, this attribute could be set directly on the iframe HTML tag and dynamically modified using JavaScript.

```html
<!DOCTYPE html> 
<html> 
  <head> 
    <meta charset="utf-8"> 
    <title>iframe muting</title> 
  </head> 
  <body> 
    <iframe id="media_iframe" muted src="https://foo.media.com"></iframe> 
    <button id="mute_iframe_btn">Mute iframe</button> 
    <script> 
      function onMuteButtonPressed() { 
        media_iframe = document.getElementById("media_iframe") 
        mute_button = document.getElementById("mute_iframe_btn") 
        if(media_iframe.muted) { 
          media_iframe.muted = false 
          mute_button.innerText = "Mute iframe" 
        } else { 
          media_iframe.muted = true 
          mute_button.innerText = "Unmute iframe" 
        } 
      } 

      mute_button = document.getElementById("mute_iframe_btn") 
      mute_button.addEventListener("click", onMuteButtonPressed) 
    </script> 
  </body> 
</html> 
```

This alternative was not selected as the preferred one, because we think that pausing media playback is preferable to just muting it.