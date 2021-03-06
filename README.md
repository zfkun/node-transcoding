node-transcoding -- Media transcoding and streaming for node.js
====================================

node-transcoding is a library for enabling both offline and real-time media
transcoding. In addition to enabling the manipulation of the input media,
utilities are provided to ease serving of the output.

Currently supported features:

* Nothing!

Coming soon (maybe):

* Everything!

## Quickstart

    npm install transcoding
    node
    > var transcoding = require('transcoding');
    > transcoding.process('input.flv', 'output.m4v',
        transcoding.profiles.APPLE_TV_2, function(err, sourceInfo, targetInfo) {
          console.log('completed!');
        });

## Installation

With [npm](http://npmjs.org):

    npm install transcoding

From source:

    cd ~
    git clone https://benvanik@github.com/benvanik/node-transcoding.git
    npm link node-transcoding/

### Dependencies

node-transcoding requires `ffmpeg` and its libraries `avformat` and `avcodec`.
Make sure it's installed and on your path. It must be compiled with libx264 to
support most output - note that some distributions don't include this and you
may have to compile it yourself. Annoying, I know.

#### Source

    ./configure \
        --enable-gpl --enable-nonfree --enable-pthreads \
        --enable-libfaac --enable-libfaad --enable-libmp3lame \
        --enable-libx264
    sudo make install

#### Mac OS X

The easiest way to get ffmpeg is via [MacPorts](http://macports.org).
Install it if needed and run the following from the command line:

    sudo port install ffmpeg +gpl +lame +x264 +xvid

You may also need to add the MacPorts paths to your `~./profile`:

    export C_INCLUDE_PATH=$C_INCLUDE_PATH:/opt/local/include/
    export CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:/opt/local/include/
    export LIBRARY_PATH=$LIBRARY_PATH:/opt/local/lib/

#### FreeBSD

    sudo pkg_add ffmpeg

#### Linux

    # HAHA YEAH RIGHT GOOD LUCK >_>
    sudo apt-get install ffmpeg

## API

### Sources/targets

All APIs take a source and target. Sources can be either strings representing
file paths or Node readable streams. Targets can be either strings representing
file paths or Node writable streams. Note that if possible you should use file
paths, as it enables much faster access to the underlying data (~2x speed
increase).

### Media Information

Whenever 'info' is used, it refers to a MediaInfo object that looks something
like this:

    {
      container: 'flv',
      duration: 126,         // seconds
      start: 0,              // seconds
      bitrate: 818000,       // bits/sec
      streams: [
        {
          type: 'video',
          codec: 'h264',
          profile: 'Main',
          profileId: 77,
          profileLevel: 30,
          resolution: { width: 640, height: 360 },
          bitrate: 686000,
          fps: 29.97
        }, {
          type: 'audio',
          language: 'eng',
          codec: 'aac',
          sampleRate: 44100, // Hz
          channels: 2,
          bitrate: 131000
        }
      ]
    }

Note that many of these fields are optional, such as bitrate, language, profile
information, and even fps/duration. Don't go using the values without checking
for undefined first.

### Querying Media Information

To quickly query media information (duration, codecs used, etc) use the
`queryInfo` API:

    var transcoding = require('transcoding');
    transcoding.queryInfo(source, function(err, info) {
      // Completed
    });

### Transcoding Profiles

Transcoding requires a ton of parameters to get the best results. It's a pain in
the ass. So what's exposed right now is a profile set that tries to set the
best options for you. Pick your profile and pass it into the transcoding APIs.

    var transcoding = require('transcoding');
    for (var profileName in transcoding.profiles) {
      var profile = transcoding.profiles[profileName];
      console.log(profileName + ':' + util.inspect(profile));
    }

### Simple Transcoding

If you are doing simple offline transcoding (no need for streaming, extra
options, progress updates, etc) then you can use the `process` API:

    var transcoding = require('transcoding');
    transcoding.process(source, target, transcoding.profiles.APPLE_TV_2, {}, function(err, sourceInfo, targetInfo) {
      // Completed
    });

Note that this effectively just wraps the advanced API, without the need to
track events.

### Advanced Transcoding

    var transcoding = require('transcoding');
    var task = transcoding.createTask(source, target, transcoding.profiles.APPLE_TV_2);

    task.on('begin', function(sourceInfo, targetInfo) {
      // Transcoding beginning, info available
      console.log('transcoding beginning...');
      console.log('source:');
      console.log(util.inspect(sourceInfo));
      console.log('target:');
      console.log(util.inspect(targetInfo));
    });
    task.on('progress', function(progress) {
      // New progress made, currrently at timestamp out of duration
      // progress = {
      //   timestamp: 0,        // current seconds timestamp in the media
      //   duration: 0,         // total seconds in the media
      //   timeElapsed: 0,      // seconds elapsed so far
      //   timeEstimated: 0,    // seconds estimated for total task
      //   timeRemaining: 0,    // seconds remaining until done
      //   timeMultiplier: 2    // multiples of real time the transcoding is
      //                        // occuring in (2 = 2x media time)
      // }
      console.log(util.inspect(progress));
      console.log('progress ' + (progress.timestamp / progress.duration) + '%');
    });
    task.on('error', function(err) {
      // Error occurred, transcoding ending
      console.log('error: ' + err);
    });
    task.on('end', function() {
      // Transcoding has completed
      console.log('finished');
    });

    // Start transcoding
    task.start();

    // At any time, abort transcoding
    task.stop();

### HTTP Live Streaming

* [IETF Spec](http://tools.ietf.org/html/draft-pantos-http-live-streaming-07)
* [Apple Docs](http://developer.apple.com/library/ios/#documentation/networkinginternet/conceptual/streamingmediaguide/Introduction/Introduction.html)

If you are targeting devices that support HTTP Live Streaming (like iOS), you
can have the transcoder build the output in realtime as it processes. This
enables playback while the transcoding is occuring, as well as some other fancy
things such as client-side stream switching (changing audio channels/etc).

    var task = transcoding.createTask(source, null, profile, {
      liveStreaming: {
        path: '/some/path/',
        name: 'base_name',
        segmentDuration: 10,
        allowCaching: true
      }
    });

This will result in a playlist file and the MPEGTS segments being placed under
`/some/path/` with the name `base_name.m3u8`. The playlist file will be
automatically updated as new segments are generated, and files can be assumed to
be static once they are available.
