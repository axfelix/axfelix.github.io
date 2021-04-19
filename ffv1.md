# Keeping Archivematica Video Filesizes Manageable.

By default, Archivematica default creates an FFV1-encoded MKV of every video file it ingests in order to store as an archival copy. This is great for anyone whose digital preservation pipeline entails ingesting "raw" lossless video, for which FFV1 is generally considered to be the most efficient and most open (i.e. least patent-encumbered) codec available, but it's not so great if you're ingesting video that's already been compressed, even with fairly good lossy compression like a commercial blu-ray which may use H264 or H265 encoded video. At that point, you're making a larger copy than you need which will never look any better than your source. There is an argument elsewhere in favour of using uncompressed file formats wherever possible as they are generally less vulnerable to bit rot, but this generally isn't applicable where video is concerned, as lossless video files are so much larger than most other media, and the odds of being able to display an FFV1 video with a couple bad bits are not too much greater than the odds of being able to display an H264 video with a couple bad bits.

Thus, we -- and, anecdotally, other Archivematica users -- manually override this functionality in cases where we're ingesting video which is already in a common, stable format by making use of Archivematica's manual normalization workflow to generate the desired AIP and DIP derivative copies pre-ingest, and packaging them all together. This, however, is kind of a pain, and entails setting up out-of-band processes out of Archivematica, which we'd rather not be doing wherever possible. As a solution, we override the default Archivematica FPR entry for making FFV1 MKVs for preservation from this:

```
#!/bin/bash

inputFile="%fileFullName%"
outputFile="%outputDirectory%%prefix%%fileName%%postfix%.mkv"
audioCodec="pcm_s16le"
videoCodec="ffv1 -level 3"

command="ffmpeg -vsync passthrough -i \"${inputFile}\" "
command="${command} -vcodec ${videoCodec} -g 1 "
command="${command} -acodec ${audioCodec}"


command="${command} ${outputFile}"

echo $command
eval $command
```

To this:

```
#!/bin/bash

inputFile="%fileFullName%"
outputFile="%outputDirectory%%prefix%%fileName%%postfix%.mkv"
audioCodec="pcm_s16le"
videoCodec="ffv1 -level 3"

command="ffmpeg -vsync passthrough -i \"${inputFile}\" "
if ffprobe "${inputFile}" 2>&1 | grep "Video: h264" | grep "yuv420p"; then
	command="${command} -vcodec copy "
else	
	command="${command} -vcodec ${videoCodec} -g 1 "
fi
command="${command} -acodec ${audioCodec}"


command="${command} ${outputFile}"

echo $command
eval $command
```

This checks for what is arguably the most common and least at-risk video format in widespread use -- yuv420p-encoded H264 -- and, if found, copies the existing video stream over when creating an MKV for preservation.

Thanks to Andrew Berger for suggestions on this work.