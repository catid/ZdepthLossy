# ZdepthLossy

Lossy depth buffer compression designed and tested for Azure Kinect DK.
Based on the Facebook Zstd library and H.264/HEVC for compression.

ZdepthLossy defines a file format and performs full input checking.

Hardware acceleration for H.264/HEVC video compression is leveraged when
possible to reduce CPU usage during encoding and decoding.


## Benchmarks

It runs in ~4 milliseconds and produces compressed lossy depth video at 2.5 Mbps with reasonable quality.

Detailed benchmarks and test vectors available at the end of the document.


## Applications

This is useful as part of a larger real-time volumetric video streaming system
enabled by the Microsoft Azure Kinect DK.  The only data that needs to be sent
for each frame are the compressed depth frames generated by this library,
and the JPEGs from the camera.  The only hope of decoding multiple JPEGs from
multiple cameras in real-time on a Windows laptop is using the nvJPEG library.

The video recorder reads a 16-bit depth image that is compressed with this
library.  These depth images are transmitted to the video player computer over
a network.  The video parameter set must include the calibration data for the
depth sensor.  The video player uses this library to decompress the depth image
and then uses the calibration data to generate xyz, uv, and indices for
rendering the mesh.

Code for quickly regenerating the x,y,z,u,v coordinates and indices is provided
for the Azure Kinect DK in `DepthMesh.hpp`.  It runs in about 3 milliseconds/frame.

If you are looking for other libraries that are useful for making the most of the
Azure Kinect DK, I recommend looking at Intel's Open3D library, which supports
50x faster global point cloud registration than Pointcloud or OpenGR, and Open3D
also has a capture plugin for single cameras.


## Building and Using

Currently requires Windows, an NVidia graphics card, and CUDA v10.1.
If anyone is using this on other platforms please share your code changes back.

There's example usage in the `tests` folder.

This project provides a CMakeLists.txt that generates a `zdepth` target.
Currently the best way to bring this into your project is with a submodule.

Link to the target and include the header:

    #include "zdepth.hpp"

Compress 16-bit depth image to vector of bytes:

    zdepth::DepthCompressor compressor;

    ...

    std::vector<uint8_t> compressed;
    const bool is_keyframe = true;
    compressor.Compress(Width, Height, frame, compressed, is_keyframe);

Re-use the same DepthCompressor object for multiple frames for best performance.

Setting `is_keyframe = false` will use ~2x less bandwidth (e.g. 2.5 Mbps instead of 5 Mbps).
The first frame generated is always a keyframe to allow the decoder to synchronize, since
this frame contains the VPS/SPS/PPS parameter sets for the rest of the video.
Intra-refresh is used where a sweep of macroblocks are recorded at high quality.
Keyframes are IDRs in the video stream.

Decompress vector of bytes back to 16-bit depth image:

    zdepth::DepthCompressor decompressor;

    int width, height;
    std::vector<uint16_t> depth;
    zdepth::DepthResult result = decompressor.Decompress(compressed, width, height, depth);
    if (result != zdepth::DepthResult::Success) {
        // Handle input error
    }

You must use different objects for the compressor and decompressor, or it will fail to decode.


## Lossy versus Lossless

The lossless encoder is in a separate repo here:
https://github.com/catid/Zdepth
The two are split because this lossy version is harder to use.

You may want to compare the quality and size of the output with a
lossless compressor as well before deciding to use the lossy version.

The results vary with the scene.  For the unit test set, the lossless
encoder gets 4-5 Mbps.  The lossy encoder gets 2.5 Mbps with okay quality.
For the 320x288 depth mode of the Azure Kinect DK I would not recommend
setting the bitrate lower than 2 Mbps because quality goes off a cliff.
If this difference is not significant (e.g. streaming just one camera)
then the lossless version may be a decent option.


## Compression Algorithm

    High-level:

        (0) Special case for zero.
            This ensures that the H.264 encoders do not flip zeroes.
        (1) Quantize depth to 11 bits based on sensor accuracy at range.
            Eliminate data that we do not need to encode.
        (2) Rescale the data so that it ranges full-scale from 0 to 2047.
        (3) Compress high 3 bits with Zstd.
        (4) Compress low 8 bits with H.264.

    High 3-bit compression with Zstd:

        (1) Combine 4-bit nibbles together into bytes.
        (2) Encode with Zstd.

    Low 8-bit compression with a video encoder:

        (1) Folding to avoid sharp transitions in the low bits.
            The low bits are submitted to the H.264 encoder, meaning that
            wherever the 8-bit value rolls over it transitions from
            255..0 again.  This sharp transition causes problems for the
            encoder so to solve that we fold every other 8-bit range
            by subtracting it from 255.  So instead the roll-over becomes
            253, 254, 255, 254, 253, ... 1, 0, 1, 2, ...
        (2) Compress the resulting data as an image with a video encoder.
            We use the best hardware acceleration available on the platform.

    Further details are in the DepthCompressor::Filter() code.


## File Format

Format Magic is used to quickly check that the file is of this format.
Words are stored in little-endian byte order.

    struct DepthHeader
    {
        /*  0 */ uint8_t Magic;
        /*  1 */ uint8_t Flags;
        /*  2 */ uint16_t FrameNumber;
        /*  4 */ uint16_t Width;
        /*  6 */ uint16_t Height;
        /*  8 */ uint16_t MinimumDepth;
        /* 10 */ uint16_t MaximumDepth;
        /* 12 */ uint32_t HighUncompressedBytes;
        /* 16 */ uint32_t HighCompressedBytes;
        /* 20 */ uint32_t LowCompressedBytes;
        /* 24 */ uint8_t LowMinimum;
        /* 25 */ uint8_t LowMaximum;
        // Compressed data follows: High bits, then low bits.
    };

The compressed and uncompressed sizes are of packed data for Zstd.

    Flags:
        1 = Keyframe.
        2 = Using H.265 instead of H.264 for video encoding.

For more details on algorithms and format please check out the source code.
Feel free to modify the format for your data to improve the performance.


## Detailed Benchmark Results

The results are from an i9 9900K high end desktop and are not too rigorous.
Using H265 for video encoding.

It seems like the primary advantage of using a video encoder is that
non-keyframes are significantly smaller than the P-frames of my lossless format.
The keyframes are about the same size for both.

    -------------------------------------------------------------------
    Test vector: Ceiling
    -------------------------------------------------------------------

![Ceiling Image](tests/ceiling.jpg?raw=true "Ceiling Image")

    ===================================================================
    + Test: Frame 0 Keyframe=true compression
    ===================================================================
    Error hist: 1 : 28960
    Error hist: 2 : 5143
    Error hist: 3 : 1505
    Error hist: 4 : 593
    Error hist: 5 : 213
    Error hist: 6 : 118
    Error hist: 7 : 75
    Error hist: 8 : 22
    Error hist: 9 : 21
    Error hist: 10 : 6
    Error hist: 12 : 4
    Error hist: 13 : 1
    Error hist: 15 : 1

    ZdepthLossy Compression: 184320 bytes -> 12923 bytes (ratio = 14.2629:1) (3.10152 Mbps @ 30 FPS)
    ZdepthLossy Speed: Compressed in 647.109 msec. Decompressed in 30.096 msec

    Zdepth Compression: 184320 bytes -> 16499 bytes (ratio = 11.1716:1) (3.95976 Mbps @ 30 FPS)
    Zdepth Speed: Compressed in 1.463 msec. Decompressed in 0.845 msec

    Quantization+RVL+Zstd Compression: 184320 bytes -> 23251 bytes (ratio = 7.9274:1) (5.58024 Mbps @ 30 FPS)
    Quantization+RVL+Zstd Speed: Compressed in 0.69 msec. Decompressed in 0.583 msec

    Quantization+RVL Compression: 184320 bytes -> 42084 bytes (ratio = 4.37981:1) (10.1002 Mbps @ 30 FPS)
    Quantization+RVL Speed: Compressed in 0.376 msec. Decompressed in 0.461 msec

    ===================================================================
    + Test: Frame 1 Keyframe=false compression
    ===================================================================
    Error hist: 1 : 28845
    Error hist: 2 : 5111
    Error hist: 3 : 1601
    Error hist: 4 : 741
    Error hist: 5 : 355
    Error hist: 6 : 133
    Error hist: 7 : 54
    Error hist: 8 : 23
    Error hist: 9 : 12
    Error hist: 10 : 4
    Error hist: 11 : 5
    Error hist: 12 : 2
    Error hist: 13 : 3
    Error hist: 14 : 1
    Error hist: 15 : 1
    Error hist: 18 : 1

    ZdepthLossy Compression: 184320 bytes -> 9473 bytes (ratio = 19.4574:1) (2.27352 Mbps @ 30 FPS)
    ZdepthLossy Speed: Compressed in 4.614 msec. Decompressed in 3.471 msec

    Zdepth Compression: 184320 bytes -> 15495 bytes (ratio = 11.8955:1) (3.7188 Mbps @ 30 FPS)
    Zdepth Speed: Compressed in 1.447 msec. Decompressed in 0.667 msec

    Quantization+RVL+Zstd Compression: 184320 bytes -> 23143 bytes (ratio = 7.9644:1) (5.55432 Mbps @ 30 FPS)
    Quantization+RVL+Zstd Speed: Compressed in 0.658 msec. Decompressed in 0.586 msec

    Quantization+RVL Compression: 184320 bytes -> 41948 bytes (ratio = 4.39401:1) (10.0675 Mbps @ 30 FPS)
    Quantization+RVL Speed: Compressed in 0.374 msec. Decompressed in 0.465 msec

    -------------------------------------------------------------------
    Test vector: Room
    -------------------------------------------------------------------

![Room Image](tests/room.jpg?raw=true "Room Image")

    ===================================================================
    + Test: Frame 0 Keyframe=true compression
    ===================================================================
    Error hist: 1 : 25994
    Error hist: 2 : 10461
    Error hist: 3 : 4410
    Error hist: 4 : 2137
    Error hist: 5 : 1093
    Error hist: 6 : 601
    Error hist: 7 : 277
    Error hist: 8 : 175
    Error hist: 9 : 109
    Error hist: 10 : 55
    Error hist: 11 : 33
    Error hist: 12 : 25
    Error hist: 13 : 11
    Error hist: 14 : 9
    Error hist: 15 : 4
    Error hist: 16 : 1
    Error hist: 19 : 1
    Error hist: 21 : 1
    Error hist: 24 : 1

    ZdepthLossy Compression: 184320 bytes -> 20870 bytes (ratio = 8.83182:1) (5.0088 Mbps @ 30 FPS)
    ZdepthLossy Speed: Compressed in 4.846 msec. Decompressed in 4.032 msec

    Zdepth Compression: 184320 bytes -> 22626 bytes (ratio = 8.14638:1) (5.43024 Mbps @ 30 FPS)
    Zdepth Speed: Compressed in 1.674 msec. Decompressed in 0.804 msec

    Quantization+RVL+Zstd Compression: 184320 bytes -> 29460 bytes (ratio = 6.25662:1) (7.0704 Mbps @ 30 FPS)
    Quantization+RVL+Zstd Speed: Compressed in 0.609 msec. Decompressed in 0.592 msec

    Quantization+RVL Compression: 184320 bytes -> 40312 bytes (ratio = 4.57234:1) (9.67488 Mbps @ 30 FPS)
    Quantization+RVL Speed: Compressed in 0.379 msec. Decompressed in 0.493 msec

    ===================================================================
    + Test: Frame 1 Keyframe=false compression
    ===================================================================
    Error hist: 1 : 26058
    Error hist: 2 : 10081
    Error hist: 3 : 4041
    Error hist: 4 : 2134
    Error hist: 5 : 1146
    Error hist: 6 : 663
    Error hist: 7 : 388
    Error hist: 8 : 198
    Error hist: 9 : 112
    Error hist: 10 : 52
    Error hist: 11 : 22
    Error hist: 12 : 10
    Error hist: 13 : 2
    Error hist: 14 : 2
    Error hist: 15 : 1

    ZdepthLossy Compression: 184320 bytes -> 11123 bytes (ratio = 16.5711:1) (2.66952 Mbps @ 30 FPS)
    ZdepthLossy Speed: Compressed in 4.789 msec. Decompressed in 3.675 msec

    Zdepth Compression: 184320 bytes -> 18703 bytes (ratio = 9.8551:1) (4.48872 Mbps @ 30 FPS)
    Zdepth Speed: Compressed in 1.471 msec. Decompressed in 0.679 msec

    Quantization+RVL+Zstd Compression: 184320 bytes -> 29291 bytes (ratio = 6.29272:1) (7.02984 Mbps @ 30 FPS)
    Quantization+RVL+Zstd Speed: Compressed in 0.683 msec. Decompressed in 0.593 msec

    Quantization+RVL Compression: 184320 bytes -> 40188 bytes (ratio = 4.58644:1) (9.64512 Mbps @ 30 FPS)
    Quantization+RVL Speed: Compressed in 0.446 msec. Decompressed in 0.491 msec

    -------------------------------------------------------------------
    Test vector: Person
    -------------------------------------------------------------------

![Person Image](tests/person.jpg?raw=true "Person Image")

    ===================================================================
    + Test: Frame 0 Keyframe=true compression
    ===================================================================
    Error hist: 1 : 29204
    Error hist: 2 : 8270
    Error hist: 3 : 2846
    Error hist: 4 : 1045
    Error hist: 5 : 378
    Error hist: 6 : 157
    Error hist: 7 : 94
    Error hist: 8 : 37
    Error hist: 9 : 19
    Error hist: 10 : 13
    Error hist: 11 : 1
    Error hist: 12 : 2
    Error hist: 14 : 3

    ZdepthLossy Compression: 184320 bytes -> 20946 bytes (ratio = 8.79977:1) (5.02704 Mbps @ 30 FPS)
    ZdepthLossy Speed: Compressed in 4.977 msec. Decompressed in 3.991 msec

    Zdepth Compression: 184320 bytes -> 23665 bytes (ratio = 7.78872:1) (5.6796 Mbps @ 30 FPS)
    Zdepth Speed: Compressed in 1.442 msec. Decompressed in 0.69 msec

    Quantization+RVL+Zstd Compression: 184320 bytes -> 33524 bytes (ratio = 5.49815:1) (8.04576 Mbps @ 30 FPS)
    Quantization+RVL+Zstd Speed: Compressed in 0.67 msec. Decompressed in 0.61 msec

    Quantization+RVL Compression: 184320 bytes -> 45016 bytes (ratio = 4.09454:1) (10.8038 Mbps @ 30 FPS)
    Quantization+RVL Speed: Compressed in 0.422 msec. Decompressed in 0.506 msec

    ===================================================================
    + Test: Frame 1 Keyframe=false compression
    ===================================================================
    Error hist: 1 : 28831
    Error hist: 2 : 8423
    Error hist: 3 : 2910
    Error hist: 4 : 1107
    Error hist: 5 : 315
    Error hist: 6 : 65
    Error hist: 7 : 16
    Error hist: 8 : 2
    Error hist: 9 : 2
    Error hist: 10 : 1

    ZdepthLossy Compression: 184320 bytes -> 10787 bytes (ratio = 17.0872:1) (2.58888 Mbps @ 30 FPS)
    ZdepthLossy Speed: Compressed in 4.76 msec. Decompressed in 3.66 msec

    Zdepth Compression: 184320 bytes -> 19848 bytes (ratio = 9.28658:1) (4.76352 Mbps @ 30 FPS)
    Zdepth Speed: Compressed in 1.514 msec. Decompressed in 0.718 msec

    Quantization+RVL+Zstd Compression: 184320 bytes -> 33667 bytes (ratio = 5.4748:1) (8.08008 Mbps @ 30 FPS)
    Quantization+RVL+Zstd Speed: Compressed in 0.642 msec. Decompressed in 0.619 msec

    Quantization+RVL Compression: 184320 bytes -> 45120 bytes (ratio = 4.08511:1) (10.8288 Mbps @ 30 FPS)
    Quantization+RVL Speed: Compressed in 0.4 msec. Decompressed in 0.514 msec

    Test success


#### Credits

This is based on the research of:

F. Nenci, L. Spinello and C. Stachniss,
"Effective compression of range data streams for remote robot operations
using H.264," 2014 IEEE/RSJ International Conference on Intelligent Robots
and Systems, Chicago, IL, 2014, pp. 3794-3799.

The main departure is in losslessly compressing the high bits,
and using a single encoder for the low bits since in practice
having more than one encoder is problematic when recording from
multiple cameras: NVENC only supports two parallel encoders.

* Uses libdivide: https://github.com/ridiculousfish/libdivide
* Uses Zstd: https://github.com/facebook/zstd

Software by Christopher A. Taylor mrcatid@gmail.com

Please reach out if you need support or would like to collaborate on a project.
