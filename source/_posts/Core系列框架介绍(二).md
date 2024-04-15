---
title: Core系列框架介绍(二)
date: 2017-12-26 15:33:00
tags: "Foundation"
categories: "Objective-C"
---

### 音视频相关框架：CoreMedia、CoreAudio、CoreVideo、CoreAudioKit、AVFoundation、AVKit

<!-- more -->

1.CoreMedia 提供对媒体文件操作的底层接口
2.CoreAudio 提供对音频文件操作的底层接口
3.CoreVideo 提供对视频文件操作的底层接口
以上三个框架，在需要对音频或视频创建及展示进行精确控制的应用程序才会涉及，一般应用程序应该都用不上，而我们常用的是下面几个

4.CoreAudioKit
```
import CoreAudioKit.AUViewController
import CoreAudioKit.CABTMIDICentralViewController
import CoreAudioKit.CABTMIDILocalPeripheralViewController
import CoreAudioKit.CAInterAppAudioSwitcherView
import CoreAudioKit.CAInterAppAudioTransportView
```
CoreAudioKit提供了一个简单的音频界面，并且是跨应用的

5.AVFoundation
```
import AVFoundation.AVAnimation
// 媒体资源和元数据
import AVFoundation.AVAsset
import AVFoundation.AVAssetCache
import AVFoundation.AVAssetDownloadStorageManager
import AVFoundation.AVAssetDownloadTask
import AVFoundation.AVAssetExportSession
import AVFoundation.AVAssetImageGenerator
import AVFoundation.AVAssetReader
import AVFoundation.AVAssetReaderOutput
import AVFoundation.AVAssetResourceLoader
import AVFoundation.AVAssetTrack
import AVFoundation.AVAssetTrackGroup
import AVFoundation.AVAssetTrackSegment
import AVFoundation.AVAssetWriter
import AVFoundation.AVAssetWriterInput
import AVFoundation.AVAsynchronousKeyValueLoading
// 音频
import AVFoundation.AVAudioFormat
import AVFoundation.AVAudioMix
import AVFoundation.AVAudioProcessingSettings
import AVFoundation.AVBase
// 媒体捕捉
import AVFoundation.AVCameraCalibrationData
import AVFoundation.AVCaptureAudioDataOutput
import AVFoundation.AVCaptureAudioPreviewOutput
import AVFoundation.AVCaptureDataOutputSynchronizer
import AVFoundation.AVCaptureDepthDataOutput
import AVFoundation.AVCaptureDevice
import AVFoundation.AVCaptureFileOutput
import AVFoundation.AVCaptureInput
import AVFoundation.AVCaptureMetadataOutput
import AVFoundation.AVCaptureOutput
import AVFoundation.AVCaptureOutputBase
import AVFoundation.AVCapturePhotoOutput
import AVFoundation.AVCaptureSession
import AVFoundation.AVCaptureSessionPreset
import AVFoundation.AVCaptureStillImageOutput
import AVFoundation.AVCaptureSystemPressure
import AVFoundation.AVCaptureVideoDataOutput
import AVFoundation.AVCaptureVideoPreviewLayer
// 视频过渡
import AVFoundation.AVComposition
import AVFoundation.AVCompositionTrack
import AVFoundation.AVCompositionTrackSegment
import AVFoundation.AVContentKeySession
import AVFoundation.AVDepthData
import AVFoundation.AVError
import AVFoundation.AVFAudio
import AVFoundation.AVMediaFormat
import AVFoundation.AVMediaSelection
import AVFoundation.AVMediaSelectionGroup
// 元数据
import AVFoundation.AVMetadataFormat
import AVFoundation.AVMetadataIdentifiers
import AVFoundation.AVMetadataItem
import AVFoundation.AVMetadataObject
// 缓存
import AVFoundation.AVSampleBufferAudioRenderer
import AVFoundation.AVSampleBufferDisplayLayer
import AVFoundation.AVSynchronizedLayer
import AVFoundation.AVTextStyleRule
import AVFoundation.AVTime
import AVFoundation.AVTimedMetadataGroup
import AVFoundation.AVUtilities
import AVFoundation.AVVideoCompositing
import AVFoundation.AVVideoComposition
import AVFoundation.AVVideoSettings
import AVFoundation
import AVFoundation.AVOutputSettingsAssistant
// 视图
import AVFoundation.AVPlayer
import AVFoundation.AVPlayerItem
import AVFoundation.AVPlayerItemMediaDataCollector
import AVFoundation.AVPlayerItemOutput
import AVFoundation.AVPlayerItemTrack
import AVFoundation.AVPlayerLayer
import AVFoundation.AVPlayerLooper
import AVFoundation.AVPlayerMediaSelectionCriteria
import AVFoundation.AVQueuedSampleBufferRendering
import AVFoundation.AVRouteDetector
import AVFoundation.AVSampleBufferRenderSynchronizer
import CoreGraphics
import CoreMedia
import Foundation
```
AVFoundation是一个强大的多媒体处理框架，它基于CoreMedia、CoreAudio、CoreVideo、CoreAnimation等框架，所以我们对音视频的处理大多数时候都是用它，我们可以用它：
* 音视频播放和录制
* 操作媒体资源和元数据(混合音频、视频过渡效果、使用CoreAnimation动画等)

6.AVKit
```
import AVKit.AVError
import AVKit.AVKitDefines
import AVKit.AVPictureInPictureController
import AVKit.AVPlayerViewController
import AVKit.AVRoutePickerView
```
AVKit基于AVFoundation封装的框架，它提供了视频的播放界面，如果我们的设计是符合原生系统的话，毫不犹豫就应该使用它了
