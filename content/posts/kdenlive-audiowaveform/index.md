+++
title = 'Rendering Audio Waveforms in Kdenlive'
date = 2024-12-14T18:24:37+01:00
draft = true
+++

<!--
 - Typical implementations use a sliding window and store the minimum and maximum values. We can make another assumption, and hypothesize that waveforms are symmetrical: thus we only store the maximum absolute value over the windows.
 - I could include an example of the space taken by the summary vs the original audio file here.
 - The state of the old implementation was slow and not very precise: only one point is taken per frame, (so usually 30 points per second).
 - We upped the temporal resolution to 5 points per frame, so it looks better.
 - Kdenlive is based on MLT, a C library for multimedia applications. It is very versatile, and is made for lots of random access to media that passes through complex filter graphs, exactly what is needed for a multi track video editing application.
 - But it is not that well suited for dumb, linear audio stream crunching like we need here. The old implementation did exactly that: it used MLT to request one video frame's worth of audio at a time, by using a "producer" with an "audiolevel" filter on it. The filter then computed the audio level for the frame and stored it in the frame's metadata. This was very inefficient, and profiling of the process showed that very little time was spent actually decoding the streams and calculating stuff, and the majority of the time was spent on book-keeping (managing memory, properties, etc).
 - Include a step-by-step section on how to use "perf" to profile the application, and pictures of the results to supplement the discussion of the previous point.
 - Also include a discussion of what a visual summary is supposed to look like, with pictures from audacity or sonic vizualiser for examples. A naive implementation would be to sample the audio stream at regular intervals, this is fast but wrong. We actually have to reduce the maximum function over a sliding window.
 - Talk about the new implementation, using libav-* directly. It is a bit complex because it's a C API with lots of corner cases and manual memory management. Talk about the tricks to make it fast (e.g. using a FIFO), along with code.
 - Show the speed results I recorded earlier: the improved MLT method gains 1.5x to 2x speed, the libav based method gains 2x to 4x speed, depending on input.
 - Also talk about rendering. I did not touch it much, but it is interesting too. If the audio is zoomed out, we can draw a vertical line for every horizontal pixel. If the audio is zoomed in, this does not look good ! We have then to draw a path and fill it. A little trick we use is to draw half the waveform, and then just mirror it.
-->
*(Consider a catchy subtitle, e.g., "Improving Performance and Precision in Video Editing Audio Visualization")*

---

## Introduction

Recently, I worked on audio waveform generation in [Kdenlive](https://kdenlive.org), a free and open source [non-linear video editor](https://en.wikipedia.org/wiki/Non-linear_editing).

Being a full-featured video editor, Kdenlive is capable of manipulating multiple audio tracks. In the timeline, these are visually represented by their **waveform**. The old implementation was slow and quite ugly to look at, so I have been tasked - for my first job as a contractor ! - to make it both faster and more precise.

Join me as I dive deep into waveform rendering !

## Understanding Audio Waveform Rendering

### How to visually represent audio signals ?

{{< figure src="spectrogram.png" caption="[Sonic Visualiser](https://www.sonicvisualiser.org/) is able to mix and match multiple audio visualization methods." >}}

We've all seen what we colloquially call audio waveforms. Other visual audio representations exist, but this is the ubiquitous one, instantaneously recognizable by all, an universally understood symbol for audio signals.

{{< figure caption="10 kHz sine wave displayed on an analog oscilloscope ([Pittigrilli](https://commons.wikimedia.org/wiki/File:Sine_wave_10_kHz_displayed_on_analog_oscilloscope.jpg), [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0), via Wikimedia Commons)" src="https://upload.wikimedia.org/wikipedia/commons/thumb/0/0f/Sine_wave_10_kHz_displayed_on_analog_oscilloscope.jpg/512px-Sine_wave_10_kHz_displayed_on_analog_oscilloscope.jpg?20221129181436" >}}

The way we represent audio waveforms finds its origins in old analogue oscilloscopes. The horizontal axis is time, and the vertical axis is the amplitude. This simple idea can be extended to fit more specialized use-cases or design choices.

{{< figure src="mixcloud.png" caption="[Mixcloud](https://www.mixcloud.com/), an audio sharing website, uses this snake-like waveform representation." >}}

{{< figure src="mixxx.png" caption="Specialized DJ software (here, [Mixxx](https://mixxx.org)) can display the frequency content of the audio signal as color - red for bass, green for midrange, and blue for high frequencies." >}}

### Visualising digital audio

When working with digital audio, drawing a waveform can be as simple as iterating over all the samples and connecting the dots (or, alternatively, drawing lollipops[^sinc], or bars[^bars]).

[^sinc]: Or better yet, draw the estimated analog waveform using [sinc interpolation](https://en.wikipedia.org/wiki/Whittaker%E2%80%93Shannon_interpolation_formula) !

[^bars]: The bar drawing method makes the signal appear as a "staircase" - which has misled many people on the supposed shortcomings of digital audio. [See here.](https://xiph.org/video/vid2.shtml)
  
If you only have a few samples to draw, this can be enough. But what happens if you have more than a few seconds of audio ? Usual audio sample rates [go up from 44100 Hz](https://en.wikipedia.org/wiki/Sampling_(signal_processing)#Audio_sampling), meaning at least 44100 points per second of audio. Drawing each sample individually would take **forever**, and all the samples would be so close to each other you could really only see the overall shape of the signal, anyway.

{{< figure src="3kinds.png" caption="Most specialized audio software (here, [Audacity](https://www.audacityteam.org/)), switch rendering methods depending on the zoom level. Here, lollipops, line, and summary.">}}

In this case, the approach is to "summarize" the signal. For each pixel column that is to be drawn on the screen, find out the corresponding start and end times in the signal. In this interval, compute the negative and positive peaks, and simply draw a vertical line between them. Do that enough times and you end up with that typical spiky blob-like shapes. That's way more efficient !

{{< figure src="rms.png" caption="Some audio editing also show the RMS amplitude in the summarie with a different color, which more closely matches the percieved loudness." >}}

### Waveform Differences: Audio Editing vs Video Editing Software

It is understood that users of audio editing software would want to be able to edit, cut and modify individual samples. However, users of video editing software do not expect to be able to zoom enough to see individual samples. When dealing with video, the smallest indivisible unit of time often considered is the *video frame*, usually 1/30 to 1/60 of a second: at these resolutions, drawing summaries is more than enough.

Another key difference is that in typical audio editing tasks, and with today's machines, the entire audio files can be loaded in memory. This makes accessing the audio data very fast when time comes to draw it on the screen.

In video editing (and in heavy audio applications, such as DAWs), it is is unthinkable to fit all the media in memory as multiple audio and video tracks are typically handled at once. It is also not feasible to simply read the files on disk each time we want to draw them on screen. Even the fastest SSDs we can find nowadays would not be fast enough, and that's not even accounting for decoding time - not to mention that not all media formats are easily seekable.

So, what do we do ? We can't fit the files in memory, and we can't read them off the disk fast enough.

The usual approach is to summarize the waveform on load, at a lower resolution, then the summary is cached in so-called *peak files*[^peakfiles]. When drawing the audio waveform on screen, the application simply uses the cached summary instead of having to re-read the files on disk each time[^nolollipops].

[^nolollipops]: Of course, using this method, we still have to read the actual files on disk if we want to draw at higher resolutions than our cached summaries.

[^peakfiles]: These files are littered *everywhere*. `.asd` for Ableton Live, `.pkf` for Adobe products, and so on...

Then, the actual rendering can also be a bit different. The timeline of a video editing application can easily become messy, and sacrifying accuracy for visual clarity can improve the user experience:

- The audio channels can be merged together to save screen real estate;
- Only display half of them, since waveforms are always almost symmetrical anyway;
- The user may want to normalize the audio streams - so that even quiet audio tracks are easily identifiable in a cluttered timeline.

## 3. KDenlive's previous implementation

### How It Worked

And, in fact, Kdenlive's implementation did just that. But to understand how exactly, we most go a bit deeper into Kdenlive's architecture.

Kdenlive is based on [MLT](https://www.mltframework.org/), a C framework which aims to be a toolkit for multiple multimedia-based applications, including non-linear video editing. This library provides the abstractions that software like Kdenlive (and [Shotcut](https://www.shotcut.org/), and others) use.

All objects in MLT (called *services*) produce, consume or transform *frames* (a *frame* includes video and audio). Among them, the *producers* read frames from a video file or another composition, the *consumers* ingest frames to display them on the screen, or encode them to a media file, the *filters* transform frames. All these services can be then connected in a *network*, to create complex audio/video graphs. There's a lot more going into it, if you're interested, the [MLT Docs](https://www.mltframework.org/docs/framework/) provide a great overview.

As a matter of fact, there is a [`audiolevel` MLT filter](https://www.mltframework.org/plugins/FilterAudiolevel/) that does exactly what we want: attach it to a producer, and each time you read a frame, it will write the sound amplitude to its metadata. Generating a summary is as simple as reading frames in a loop, and storing the sound amplitude in a vector somewhere. That's it ! No need to fiddle with sample rates or audio decoding, everything is abstracted away by MLT.

The resulting summary vector is then cached by serializing its content as color data in a png file (thus gaining free compression !) to avoid recalculating it the next time the project is opened, and stored in-memory for consumption by the drawing routines.

### Limitations

Unfortunately, as straightforward as this approach is, it suffers from a few limitations.

Keeping only one point per frame in the summary is in fact not sufficient for the users. Even if we're not talking about the levels of precision required in audio editing apps, a resolution of 1/30 of a second simply hides too much information. Also, the audio levels were stored in a 8-bit int, and thus was able to store only $2^8 = 256$ discrete steps, which did not really help.

The rendering also looked like it did somme funky stuff: comparing it with some reference renderings from other audio software, something was defenitly off.

### But why is it so slow ?

But the worst was the performance. Generating audio summaries usually take some time (after all, you have to read and decode the whole file), but the competition is able to do it much, much quicker. In fact, the operations that should consume the majority of time are disk access and decoding the audio streams.

One of the ways to identify the root cause of the performance problems is using a [profiler](https://en.wikipedia.org/wiki/Profiling_(computer_programming)). A profiler instruments the program and measures statistics like the amount of function calls. What we're after here is to get a report on how much time is spent in which functions.

Profiling for performance in a complex program like Kdenlive is a bit tricky: to obtain representative results, one has to be careful on the profiling approach. It would be infeasible to use traditional event-based profilers, as they would slow down the execution too much. Since we suspect that a lot of time is spent doing i/o, we would skew the results if we altered the program's execution speed too much.

So, we turn ourselves to sampling profilers. Instead of instrumenting every single function call in a target program, these profilers take "snapshots" of their call stack at regular intervals. I will be using [perf](https://perfwiki.github.io/main/) for profiling and [hotspot](https://github.com/KDAB/hotspot) for visualisation, but many other tools are available.

### Profiling the audio summary generation in Kdenlive

- Insert start and stop collecting signals in source
- Build with optimizations and debug symbols
- perf record, use high frequency
- run kdenlive and load a preferably long audio file
- use hotspot


- Discuss inefficiencies revealed through profiling: most time was spent on memory and property management rather than actual decoding or processing.  
- Include a **step-by-step guide on using "perf"** for profiling, with visuals of profiling results to emphasize the bottlenecks.  


## **4. Designing the New Approach**  

### **Summarizing Audio Data Efficiently**  

- Describe the key idea: precomputing a coarse summary of the audio, storing it in memory, and dynamically summarizing it further for rendering.  
- Explain the **sliding window approach** for calculating summaries, focusing on:  
  - Why symmetrical waveforms justify storing only the maximum absolute value.  
  - How this reduces memory use (with an example comparing space used by summaries vs original files).

### **Switching to libav-* for Performance**  

- Overview of the decision to use libav-* directly instead of MLT.  
- Challenges of working with the libav C API, including corner cases and manual memory management.  
- Performance tricks: FIFO buffers, efficient data flow.  
- **Show results**: speed improvements of the new method (2x to 4x, depending on input).  

---

## **5. Improvements in Visual Rendering**  

### **What Makes a Good Waveform Visualization?**  

- Discuss the trade-offs between accuracy and speed:  
  - Naive sampling is fast but produces misleading visuals.  
  - Accurate rendering involves reducing the maximum function over a sliding window.  

### **Rendering Techniques**  

- Low zoom: draw vertical lines for each horizontal pixel.  
- High zoom: use paths to represent waveforms and fill them, mirroring the path for symmetry.  

---

## **6. Results and Reflections**  

### **Before and After**  

- Summarize the changes:  
  - Temporal resolution increased (5 points per frame).  
  - Performance improvements from profiling and using libav-* directly.  
- Show visuals comparing the old and new waveforms.  

### **Lessons Learned**  

- Highlight key engineering insights and any open challenges.  
- Discuss how this experience contributed to your skills and understanding of audio processing in video editing.

---

## **7. Conclusion**  

- Reflect on the broader impact of this work on Kdenlive and its users.  
- Encourage others to contribute to open-source projects and explore technical challenges.  
- Provide links to Kdenlive’s repo, any related documentation, or your blog’s contact section for feedback/questions.  

---

## **8. Optional Bonus Section: Get Technical**  

- Include advanced details or annotated snippets of the new implementation for readers interested in digging deeper.

---

Would you like help expanding a specific section, drafting introductions, or creating visuals to illustrate your points?
