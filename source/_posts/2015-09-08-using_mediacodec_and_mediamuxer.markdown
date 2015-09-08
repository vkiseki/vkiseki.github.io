---
layout: post
title: "Using MediaCodec and MediaMuxer"
date: 2015-09-08 12:38:20 +0800
comments: true
categories: [Android, media]
---

For API 18+, `MediaCodec` and `MediaMuxer` can be used for re-encoding videos efficiently.  
This article will explain how to complete such tasks with the help of `ExtractDecodeEditEncodeMuxTest.java` from android cts.

<!--more-->

You can find the base codes from [here](https://android.googlesource.com/platform/cts/+/kitkat-release/tests/tests/media/src/android/media/cts?autodive=0/).  
<br/>

#####Summary of **ExtractDecodeEditEncodeMuxTest**:  
1. `extractDecodeEditEncodeMux()` initializes the extractors, decoders, encoders and muxer.  
You can config the output format here.  
2. `doExtractDecodeEditEncodeMux()` performs the actual work. Data flows like:  
InputFile -> Extractor -> Decoder -> OutputSurface -> (eglSwapBuffers) -> InputSurface -> Encoder -> Muxer -> OutputFile  
<br/>

#####Advantages:
1. Using `MediaCodec` allows developers to access low-level media codecs for better performance.
2. Using OpenGL `Surface` to copy data from decoder to encoder is much faster than using `ByteBuffer`.
3. With `Surface-to-surface`, there is no need to consider the compatibility of YUV format on various devices.  
(See the differences of three data feeding approaches: [Section EncodeDecodeTest](http://bigflake.com/mediacodec/#EncodeDecodeTest))  
<br/>

#####Modifications:
* **Fix the presentation time issue**  
Sometimes the original codes will throw an error indicating that the last presentationTimeUs of encoder is smaller than current presentationTimeUs.  
We can add an extra determination to avoid it:  
{% codeblock lang:java ExtractDecodeEditEncodeMuxTest.java %}
// add these line before sending data to muxer.
// make sure presentation time of buffer info is larger than last presentation time
if (mLastAudioPresentationTime == 0) {
	mLastAudioPresentationTime = audioEncoderOutputBufferInfo.presentationTimeUs;
} else {
	if (mLastAudioPresentationTime > audioEncoderOutputBufferInfo.presentationTimeUs) {
		audioEncoderOutputBufferInfo.presentationTimeUs = mLastAudioPresentationTime + 1;
	}
	mLastAudioPresentationTime = audioEncoderOutputBufferInfo.presentationTimeUs;
}

// send data to muxer
if (audioEncoderOutputBufferInfo.size != 0) {
	muxer.writeSampleData(
		outputAudioTrack, encoderOutputBuffer, audioEncoderOutputBufferInfo);
}
{% endcodeblock %}
<br/>

* **Fix the format issue**  
If incorrect output format was set, strange thing might happen when playing videos.  
It seems that converting the auido channel from one to two results in playback problems,  
thus we need to set the output format with same base attributes as input format.
{% codeblock lang:java ExtractDecodeEditEncodeMuxTest.java %}
MediaFormat outputAudioFormat = MediaFormat.createAudioFormat(
	OUTPUT_AUDIO_MIME_TYPE,
	inputFormat.getInteger(MediaFormat.KEY_SAMPLE_RATE),
	inputFormat.getInteger(MediaFormat.KEY_CHANNEL_COUNT)
);
{% endcodeblock %}
<br/>

* **Track the encoding progress**  
The encoding of audio is very faster, maybe we can simply ignore it when re-encoding small videos.  
You should save the duration of video file when initializing (Line 2),  
And then track current position of video encoder's output (Line 7)  
{% codeblock lang:java ExtractDecodeEditEncodeMuxTest.java %}
// save the duration of video
mDuration = inputFormat.getLong(MediaFormat.KEY_DURATION);

...

// notify progress changed
mCurrentPosition = mLastVideoPresentationTime;
// send data to muxer
if (videoEncoderOutputBufferInfo.size != 0) {
	muxer.writeSampleData(
		outputVideoTrack, encoderOutputBuffer, videoEncoderOutputBufferInfo);
});
{% endcodeblock %}
<br/>

* **Process frame color**  
You can process the frame color with `FRAGMENT_SHADER`.  
The preset shader swaps green and blue channels.  
Below is the default shader that make no changes to frame color. (Quoted from `TextureRender`)
{% codeblock lang:java ExtractDecodeEditEncodeMuxTest.java %}
private static final String FRAGMENT_SHADER =
	"#extension GL_OES_EGL_image_external : require\n" +
		"precision mediump float;\n" +
		"varying vec2 vTextureCoord;\n" +
		"uniform samplerExternalOES sTexture;\n" +
		"void main() {\n" +
		"  gl_FragColor = texture2D(sTexture, vTextureCoord);\n" +
		"}\n";
{% endcodeblock %}
<br/>

* **Transform frames**  
Sadly there is no interface for transforming frames, so we need to implement this manually.  
Using a transform matrix, we can easily scale, rotate or flip the video.
{% codeblock lang:java TextureRender.java %}
public void drawFrame(SurfaceTexture st) {
	...

	// calculate MVPMatrix
	Matrix.setIdentityM(MMatrix, 0);
	Matrix.setIdentityM(VMatrix, 0);
	Matrix.multiplyMM(mMVMatrix, 0, VMatrix, 0, MMatrix, 0);
	Matrix.multiplyMM(mMVPMatrix, 0, mPMatrix, 0, mMVMatrix, 0);
	GLES20.glUniformMatrix4fv(muMVPMatrixHandle, 1, false, mMVPMatrix, 0);
	
	...
}

public void changeProjectionMatrix(float[] matrix) {
	mPMatrix = matrix;
}
{% endcodeblock %}
{% codeblock lang:java OutputSurface.java %}
public void changeProjectionMatrix(float[] matrix) {
	mTextureRender.changeProjectionMatrix(matrix);
}
{% endcodeblock %}
{% codeblock lang:java ExtractDecodeEditEncodeMuxTest.java %}
private void extractDecodeEditEncodeMux() throws Exception {
	...

	// Create a MediaCodec for the decoder, based on the extractor's format.
	outputSurface = new OutputSurface();
	outputSurface.changeFragmentShader(FRAGMENT_SHADER);

	// scale, rotate or flip the matrix
	float[] matrix = new float[16];
	Matrix.setIdentityM(matrix, 0);
	outputSurface.changeProjectionMatrix(matrix);

	videoDecoder = createVideoDecoder(inputFormat, outputSurface.getSurface());
	
	...
}
{% endcodeblock %}
<br/>

* **Change frame rate**  
Although we can set the frame rate of the output format, it actually does not change the frame rate.  
We need to drop frames manually when decoding videos, there are two approaches:  
1. Drop frames by frame index.  
Calculate frame step, then drop frames whose index can be divided with no remainder.  
Sadly again, `MediaExtrator` is not able to provide you with the frame rate of a video file. Maybe you should use other tools to achieve this.  
2. Drop frames by frame presentation time.  
Divide one second to n pieces, then simply drop frames by comparing with `videoDecoderOutputBufferInfo.presentationTimeUs`.  
<br/>
Drop frames when frame rate of source video is known:
{% codeblock lang:java ExtractDecodeEditEncodeMuxTest.java %}
private void extractDecodeEditEncodeMux() throws Exception {
	...
	if (mSourceFrameRate > OUTPUT_VIDEO_FRAME_RATE) {
		mFrameStep = (float) mSourceFrameRate / OUTPUT_VIDEO_FRAME_RATE;
	}
	...
}

private void doExtractDecodeEditEncodeMux(...) {
	...
	if (render) {
		boolean dropFrame = mFrameStep != 0 && (videoDecodedFrameCount % mFrameStep) == 0;
		if (dropFrame) {
			if (VERBOSE) Log.d(TAG, "drop frame: " + videoDecodedFrameCount);
		} else {
			if (VERBOSE) Log.d(TAG, "output surface: await new image");
			outputSurface.awaitNewImage();
			// Edit the frame and send it to the encoder.
			if (VERBOSE) Log.d(TAG, "output surface: draw image");
			outputSurface.drawImage();
			inputSurface.setPresentationTime(
				videoDecoderOutputBufferInfo.presentationTimeUs * 1000);
			if (VERBOSE) Log.d(TAG, "input surface: swap buffers");
			inputSurface.swapBuffers();
			if (VERBOSE) Log.d(TAG, "video encoder: notified of new frame");
		}
	}
	...
}
{% endcodeblock %}