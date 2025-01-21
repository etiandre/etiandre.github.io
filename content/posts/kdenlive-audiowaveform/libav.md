+++
title = 'Using libav* to decode audio streams in C/C++'
date = 2025-01-16T09:58:59+01:00
draft = true
+++

This post presents the notions and thought process behind audio processing using [ffmpeg's libav](https://trac.ffmpeg.org/wiki/Using%20libav*), with annotated example code.

The `libav*` set of libraries is the foundation behind the ubiquitous `ffmpeg`, and has a reputation for being hard to grasp - most people<sup>*[citation needed]*</sup> prefer using the command-line API rather than having to deal with it. Thankfully, [the official documentation](https://www.ffmpeg.org/documentation.html) and the [API docs & examples](https://www.ffmpeg.org/doxygen/trunk/index.html) provide solid guidance for those who want to dive deeper. Let’s demystify the basics and get started!

## Introduction

Before diving in, let's clarify a few key terms.  

A media file is essentially a **container**[^container] wrapping one or more **streams**—these could be video, audio, subtitles, or other data types. These streams are accessed in small chunks called **packets** (in FFmpeg terminology), which can contain one or more **frames**.  

[^container]: Containerless media files also exist, we can still view them as a "transparent" container that does nothing and contains only 1 stream.

Streams are often compressed, so they need to be decoded into raw data by a **codec** before an application can process them. For audio streams, the raw data is decoded as **raw pcm audio**, then stored in an **audio buffer**. However, several types and layouts of raw audio exist:

- **Channel Layout**: defines not only the number of channels, but also their specific roles and arrangement (e.g. what we call "stereo" audio means 2 channels, first channel is on the left, second channel is on the Right)
- **Sample Rate**: The number of audio samples per second, measured in Hertz, determining the audio's temporal resolution.  
- **Sample Format**: Describes how each audio sample is stored, such as integers (e.g., signed or unsigned, 8/16/24/32-bit) or floating-point values.  
- **Buffer Layout**: Dictates how channel data is organized in memory:  
  - In **deinterleaved** (or **planar**) audio, samples for each channel are stored separately, e.g., `[L1, L2, L3]` for the left channel and `[R1, R2, R3]` for the right.
  - In **interleaved** (or **packed**) audio, samples from all channels are stored sequentially, e.g., `[L1, R1, L2, R2, L3, R3]`.


Writing code to handle all raw audio formats can be done but would be tiresome. Instead, we can opt to **convert** it towards a fixed format of our choosing, depending on the application.

When processing, it could be tempting to load the whole audio stream in one go in a big buffer, then process it afterwards. Let's take an example of a typical audio stream: 1 hour of 16-bit, 5.1 surround (= 6 channel), raw audio sampled at 48kHz. This would result in a buffer of size $3,600 \text{ s} \times 48,000 \text{ Hz} \times 6 \text{ channels} \times 2 \text{ bytes (16 bits)} \approx 2.07 \text{ Gigabytes} $ ! Even if most computers today have enough RAM to handle this, let's not be greedy, and process it instead piece by piece, which is what we'll do next.

## The plan

Our overarching goal is to be able **to process some audio audio stream** (to filter it, send it to the soundcard, or whatever) stored inside some media file. For that we need to decode the stream to obtain a series of raw audio buffers. For this example, we assume that our imaginary audio processing function only accepts **signed 16-bit interleaved audio**, of any sample rate and any channel layout.

Using `libav*`, this boils down to the following steps:

**Initialization & checks**

1. Open the file and decode the container format
2. Find out which streams are audio, and check that our stream index is valid
3. Find out the codec suitable for decoding the stream and initialize it
4. Get the audio stream details (sample format, etc) and use it to setup the sample converter
5. Allocate an output buffer to store the decoded raw audio

**Main processing loop:**

- For each packet in the file:
  - Make sure it corresponds to the audio stream we want, else toss it and go to the next
  - For each frame in the packet:
    - Use the codec to decode the compressed audio into raw audio
    - Convert its sample format, if needed, and store the result in the buffer
    - Process the data

Phew ! Now that's out of the way, let's get into the details.

## Code walkthrough

### Initialization

First, let's include and initialize all the `libav` structures we need. We also declare some variables to store information about the stream later.

```c
#include <stdio.h>
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/channel_layout.h>
#include <libavutil/samplefmt.h>
#include <libswresample/swresample.h>

AVFormatContext *fmt_ctx = NULL;
AVCodec *codec = NULL;
AVPacket *packet = av_packet_alloc();
AVFrame *frame = av_frame_alloc();
AVStream *stream = NULL;
AVCodecContext *codec_ctx = NULL;
SwrContext *swr_ctx = NULL;

AVSampleFormat src_sample_fmt, dst_sample_fmt;
AVChannelLayout *src_ch_layout, *dst_ch_layout;
int ret = 0;
uint8_t **buf = NULL;
int dst_nb_samples = 0, max_dst_nb_samples = 0;
int src_rate = 0, dst_rate = 0;
int src_nb_channels = 0, dst_nb_channels = 0;
int dst_linesize = 0;

// Let's say the  stream we're interested in the first one.
const int STREAM_IDX = 0;
```

Let's then go trough the plan!

First, we open the file and populate the `AVFormatContext` struct, which will contain info on the container format. In each step, we must make sure that each API call succeeds before calling the next one, or our program will segfault (in the best scenario).

```c
ret = avformat_open_input(&fmt_ctx, "uri://to/media.file", NULL, NULL);
if (ret < 0) {
    fprintf(stderr, "Could not open input file: %s\n", av_err2string(ret));
    goto cleanup;
}

// Some container formats, such as MPEG, don't have a header.
// This function reads a few packets to fully populate fmt_ctx.
ret = avformat_find_stream_info(fmt_ctx, NULL);
if (ret < 0) {
    fprintf(stderr, "Could not find stream information: %s\n", av_err2string(ret));
    goto cleanup;
}

```

Let's then get a pointer to the desired `AVStream`, and use the info it contains to initialize the codec.

```c
stream = fmt_ctx->streams[STREAM_IDX];

// Find the corresponding codec for the requested stream
codec = avcodec_find_decoder(stream->codecpar->codec_id);
if (!codec) {
    fprintf(stderr, "No suitable decoder found for %s\n", avcodec_get_name(stream->codecpar->codec_id));
    goto cleanup;
}

// Create the codec context
codec_ctx = avcodec_alloc_context3(codec);
if (!codec_ctx) {
    fprintf(stderr, "Failed to allocate codec context");
    goto cleanup;
}

// Pass the stream info to the codec
ret = avcodec_parameters_to_context(codec_ctx, stream->codecpar);
if (ret < 0) {
    fprintf(stderr, "Failed to copy codec parameters to codec: %s\n", av_err2string(ret));
    goto cleanup;
}
```

Some codecs are able to output multiple sample formats. If we're lucky, we can directly get the desired sample format (in our case, interleaved signed 16 bits integers[^integers]), but it can't be guaranteed to be the case.

[^integers]: In pro-audio applications, and if you can spare the processing power, you probably want to do your computation using floats.

```c
codec_ctx->request_sample_fmt = AV_SAMPLE_FMT_S16;
```

Now that we have everything, we can initialize the codec.

```c
ret = avcodec_open2(codec_ctx, codec, NULL);
if (ret < 0) {
    fprintf(stderr, "Failed to open codec: %s\n", av_err2string(ret));
    goto cleanup;
}
```

Next, we allocate the sample converter. We use `libswresample`, which is included with ffmpeg. Let's set the source and destination sample formats. If the codec is already outputting `AV_SAMPLE_FMT_S16`, then the converter will do nothing.

```c
src_sample_fmt = codec_ctx->sample_fmt;
dst_sample_fmt = AV_SAMPLE_FMT_S16;
```

`libswresample` can also perform resampling and up- and down-mixing, which we're uninterested in: we simply set the output settings to the same thing as the input settings.

```c
src_ch_layout = &codec_ctx->ch_layout;
dst_ch_layout = src_ch_layout;
src_nb_channels = codec_ctx->ch_layout.nb_channels;
dst_nb_channels = src_nb_channels;
src_rate = codec_ctx->sample_rate;
dst_rate = src_rate;
```

We are now able to initialize the library.

```c
ret = swr_alloc_set_opts2(&swr_ctx, dst_ch_layout, dst_sample_fmt, dst_rate, src_ch_layout, src_sample_fmt, src_rate, 0, NULL);
if (ret < 0) {
    fprintf(stderr, "Failed to set SwrContext options: %s\n", av_err2string(ret));
    goto cleanup;
}

if ((ret = swr_init(swr_ctx)) < 0) {
    fprintf(stderr, "Failed to initialize SwrContext: %s\n", av_err2string(ret));
    goto cleanup;
}
```

That's it for initialization ! Now, for the *pièce de résistance*.

### The loop

The outer loop iterates on packets. For each of them, we must make sure that it actually corresponds to the stream we're interested in, and then send it to the codec for decoding.

```c
while (av_read_frame(fmt_ctx, packet) >= 0) {
    // Discard the packets that correspond to other streams
    if (packet->stream_index != STREAM_IDX) {
        av_packet_unref(packet);
        continue;
    }

    // Decode the packet
    if ((ret = avcodec_send_packet(codec_ctx, packet)) < 0) {
        fprintf(stderr, "Error sending packet for decoding: %s\n", av_err2string(ret));
        break;
    }
```

Each packet may contain more than 1 frame of audio data. This inner loop iterates on them and stores the raw audio data in an `AVFrame`.

```c
    while (ret >= 0) {
        ret = avcodec_receive_frame(codec_ctx, frame);
        if (ret == AVERROR(EAGAIN)) {
            break; // we're done with this packet, go to the next one
        }

        if (ret < 0) {
            fprintf(stderr, "Error during decoding: %s\n", av_err2string(ret));
            goto cleanup;
        }
```

Now, we want to convert the sample format and store the result in `buf`. But remember that we never allocated it till now.

The reason is that the audio buffer that the codec gives us is of variable size: we need to check on each frame if `buf` is able to handle a full frame's worth of data, and grow it if not.

```c
        // get the number of samples that will be produced by libswr
        dst_nb_samples = swr_get_out_samples(swr_ctx, frame->nb_samples);

        // do we need to grow the buffer ?
        if (dst_nb_samples > max_dst_nb_samples) {
            // first, free it if it was previously allocated
            if (buf) {
                av_freep(&buf[0]);
            }
            av_freep(&buf);

            // do the actual memoryallocation
            ret = av_samples_alloc_array_and_samples(&buf, &dst_linesize, dst_nb_channels, dst_nb_samples, dst_sample_fmt, 0);
            if (ret < 0) {
                fprintf(stderr, "Failed to allocate output buffer: %s\n", av_err2string(ret));
                goto cleanup;
            }
            // store the allocated size in max_dst_nb_samples
            max_dst_nb_samples = dst_nb_samples;
        }
```

Finally, the last step:  sample format conversion of the samples in `frame->extended_data`. If the codec already outputs the desired format, this will just copy the untouched samples to `buf`.

And That's it ! You can then use the buffer as you see fit. The inner loop continues until there is no more frames in the packet. Once we're done with the packet, we deallocate it and go back to the top of the outer loop.

```c
        ret = swr_convert(swr_ctx, buf, dst_nb_samples, const_cast<const uint8_t **>(frame->extended_data), frame->nb_samples);
        if (ret <= 0) {
            fprintf(stderr, "Failed to convert samples: %s\n", av_err2string(ret));
            goto cleanup;
        }

        my_custom_audio_processing_function(buf, dst_nb_samples);
    }

    av_packet_unref(packet);
}
```

After we get out of the outer processing loop, all that's left to do is housekeeping. All the `goto cleanup`s make sure that everything is deallocated properly should anything go wrong.

```c
cleanup:

if (buf) {
    av_freep(&buf[0]);
}
av_freep(&buf);
av_frame_free(&frame);
av_packet_free(&packet);
avcodec_free_context(&codec_ctx);
swr_free(&swr_ctx);
avformat_close_input(&fmt_ctx);
```

### But I don't want variable-size buffers!

It can sometimes be more convenient to process audio data in chunks of a fixed number of samples (such as a power of 2 for [fast fourier transforms](https://en.wikipedia.org/wiki/Fast_Fourier_transform), for example).

A common technique is to use a [FIFO queue](https://en.wikipedia.org/wiki/Queue_(abstract_data_type)), of which [circular buffers](https://en.wikipedia.org/wiki/Circular_buffer) are a common implementation in the audio world[^fifoqueue]. Each time we obtain a variable-size buffer, we put its contents at the back of the queue. Once the queue has filled up enough, we can obtain a chunk of the desired, fixed size from the front of the queue.

[^fifoqueue]: I will be using the terms "FIFO" and "queue" interchangeably. Don't @ me.

Fortunately for us, `libav` provides a ready-to-use audio FIFO queue: `AVAudioFIFO`. It even grows automatically, which is nice!

Let's include the header and allocate the queue and fixed buffer first.

```c
#include <libavutil/audio_fifo.h>

const size_t CHUNK_NSAMPLES = 1024

// Allocate fixed-length buffer, of size CHUNK_NSAMPLES
uint8_t **chunk = NULL;
ret = av_samples_alloc_array_and_samples(&chunk, &dst_linesize, dst_nb_channels, CHUNK_NSAMPLES, dst_sample_fmt, 0);
if (ret < 0) {
    fprintf(stderr, "Failed to allocate output buffer: %s\n", av_err2string(ret));
    goto cleanup;
}

// Allocate the fifo with a bit of space. It will be grown automatically if needed, anyway.
AVAudioFifo *fifo = av_audio_fifo_alloc(dst_sample_fmt, dst_nb_channels, 2 * CHUNK_NSAMPLES);
```

In the inner loop, instead of using `buf` directly, we instead enqueue it onto `fifo`.

```c
ret = av_audio_fifo_write(fifo, reinterpret_cast<void **>(buf), dst_nb_samples);
if (ret < 0) {
    fprintf(stderr, "Failed to write samples to audio fifo: %s\n", av_err2string(ret));
    goto cleanup;
}
```

Next, we check if there is enough data in `fifo`. We use a while loop because we may have enqueued more that one chunk's worth of data, in which case we want to dequeue as many chunks as is possible.

We store the data in `chunk`[^reuse] and then we can process it however we want.

[^reuse]: You might want to reuse `buf` instead of having a separate buffer, if you care about memory usage.

```c
while (av_audio_fifo_size(fifo) >= CHUNK_NSAMPLES) {
    av_audio_fifo_read(fifo, reinterpret_cast<void **>(chunk), CHUNK_NSAMPLES);

    my_custom_fixed_audio_processing_function(chunk);
}
```

Of course, don't forget to deallocate the fifo and your fixed-size buffer once you're done with them.

```c
cleanup:
...
av_audio_fifo_free(fifo);
if (chunk) {
    av_freep(&chunk[0]);
}
av_freep(&chunk);

...
```

### Conclusion

TODO