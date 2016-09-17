---
title: iOS开发：如何在视频中添加水印
date: 2016/7/15 12:25:25
---



先上个图

![image](http://oaayo42x2.bkt.clouddn.com/watermark_gif.gif)



### 一. 根据AVAsset得到AVMutableComposition
从asset中取得视频的音视频轨道：
```objc
AVAssetTrack *assetVideoTrack = nil;
AVAssetTrack *assetAudioTrack = nil;
/**
 *  判断asset是否有音视频的轨道，并获取
 */
if ([[_asset tracksWithMediaType:AVMediaTypeVideo] count] != 0) {
    assetVideoTrack = [[_asset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
}
if ([[_asset tracksWithMediaType:AVMediaTypeAudio] count] != 0) {
    assetAudioTrack = [[_asset tracksWithMediaType:AVMediaTypeAudio] objectAtIndex:0];
}
```
使用AVAsset中的`tracksWithMediaType:` 方法可以获取视频中的音频或视频的轨道。

<!--more-->

根据获取的音视频轨道合成 `AVMutableComposition`：
```objc
_composition = [AVMutableComposition composition];
CMTime insertionPoint = kCMTimeZero;
if (assetVideoTrack != nil) {
    AVMutableCompositionTrack *compositionVideoTrack =
    [_composition addMutableTrackWithMediaType:AVMediaTypeVideo
                              preferredTrackID:kCMPersistentTrackID_Invalid];
    [compositionVideoTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, [_asset duration])
                                   ofTrack:assetVideoTrack
                                    atTime:insertionPoint error:nil];
}
if (assetAudioTrack != nil) {
    AVMutableCompositionTrack *compositionAudioTrack =
    [_composition addMutableTrackWithMediaType:AVMediaTypeAudio
                              preferredTrackID:kCMPersistentTrackID_Invalid];
    [compositionAudioTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, [_asset duration])
                                   ofTrack:assetAudioTrack
                                    atTime:insertionPoint error:nil];
}
```
方法`addMutableTrackWithMediaType:preferredTrackID:`用于为composition添加一个新的轨道，如果添加视频轨道类型参数填 `AVMediaTypeVideo`，添加音频轨道用 `AVMediaTypeAudio` ；TrackID是追踪轨道的标识，默认填 `kCMPersistentTrackID_Invalid` 就可以。

方法 `insertTimeRanges:ofTracks:atTime:error:` 将刚才在asset获取的视频轨道插进compositionVideoTrack中去，TimeRange参数代表想要插入的视频资源的时间范围，这里是从开始到结束。

### 二. 组装AVMutableVideoComposition（包括水印合成）
得到`AVMutableComposition`后就可以对视频搞事情了。

首先了解一下 `AVMutableVideoComposition`，这个类干嘛用的，看上去很像上面生成的 `AVMutableVideoComposition`，其实根本没有什么直接关系，更不是父子类（`AVMutableVideoComposition` 是 `AVVideoComposition` 的可变类型，`AVMutableComposition` 是 `AVComposition` 的可变类型）。`AVVideoComposition` 对象是对两个或多个视频轨道组合在一起的方法的一个总体描述，也就是说对象里面配置了视频合成时的一些细节，例如渲染尺寸、缩放、帧时长等。

先配置简单的设置（帧时长，视频长宽）：
```objc
AVMutableVideoComposition *videoComposition = [AVMutableVideoComposition videoComposition];
videoComposition.frameDuration = CMTimeMake(1, 30);
videoComposition.renderSize = self.composition.naturalSize;
```

核心加水印：
```objc
CGSize videoSize = self.composition.naturalSize; /*获取视频长宽大小*/
CALayer *customLayer = [WatermarkTool
 videoWatermaskLayerWithVideoNaturalSize:videoSize 
 videoDuration:movieDuration]; /*获取自定义layer，下面有解释*/
CALayer *overLayer = customLayer /*自定义的水印layer*/;
CALayer *parentLayer = [CALayer layer];
CALayer *videoLayer = [CALayer layer];
parentLayer.frame = CGRectMake(0, 0, videoSize.width, videoSize.height);
videoLayer.frame = CGRectMake(0, 0, videoSize.width, videoSize.height);
[parentLayer addSublayer:videoLayer];
[parentLayer addSublayer:overLayer];

/**
 *  加水印的核心，使用coreAnimation的视频合成工具
 *  animationLayer（就是上面的parentLayer）必须作为根图层，videoLayer和水印图层作为子图层
 *  视频的图层会加在这里传入的videoLayer上面
 */
videoComposition.animationTool =
[AVVideoCompositionCoreAnimationTool
videoCompositionCoreAnimationToolWithPostProcessingAsVideoLayer:videoLayer inLayer:parentLayer];
```
这个超长的方法 `videoCompositionCoreAnimationToolWithPostProcessingAsVideoLayers:inLayer:` 就是合成水印图层的核心方法，上面的图层大概是这样的：

`parentLayer` -> `videoLayer` -> 视频 -> `overLayer`

上面gif图的水印有个渐现的效果，`customLayer` 是这么设置的：
```objc
+ (CALayer *)videoWatermaskLayerWithVideoNaturalSize:(CGSize)videoSize videoDuration:(CGFloat)duration {
    CGRect videoFrame = CGRectMake(0, 0, videoSize.width, videoSize.height);
    //动画过程
    CABasicAnimation * animation = [CABasicAnimation animationWithKeyPath:@"opacity"];
    [animation setFromValue:[NSNumber numberWithFloat:0]];
    [animation setToValue:[NSNumber numberWithFloat:1.0]];
    [animation setDuration:3.0];
    [animation setBeginTime:duration-(duration > 8 ? 8 : duration)];
    [animation setFillMode:kCAFillModeForwards];
    [animation setRemovedOnCompletion:NO];
    //水印layer
    CALayer *overLayer = [CALayer layer];
    overLayer.frame = CGRectMake(0, 0, videoFrame.size.width, videoFrame.size.height);
    overLayer.opacity = 0.f;
    [overLayer addAnimation:animation forKey:@"animateOpacity"];
    //文字
    CATextLayer * textLayer = [CATextLayer layer];
    textLayer.string = @"我是水印";
    textLayer.fontSize = 50;
    textLayer.alignmentMode = kCAAlignmentCenter;
    [textLayer setForegroundColor:[UIColor whiteColor].CGColor];
    textLayer.frame = CGRectMake(0, 0, videoFrame.size.width, videoFrame.size.height);
    [overLayer addSublayer:textLayer];
    return overLayer;
}
```

配置一些乱七八糟的东西：
```objc
/**
 *  AVVideoCompositionInstruction：完成AVVideoComposition合成的指令的集合
 *  AVVideoComposition是由一组AVVideoCompositionInstruction对象格式定义的指令组成的
 */
AVMutableVideoCompositionInstruction *instruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
CGFloat durSec = CMTimeGetSeconds(self.composition.duration);
CMTime timeDur = CMTimeMake(durSec * TIME_SCALE, TIME_SCALE);
instruction.timeRange = CMTimeRangeMake(kCMTimeZero, timeDur);

/**
 *  AVVideoCompositionLayerInstruction：用于定义对给定视频轨道应用的模糊、变形和裁剪效果
 */
AVAssetTrack *clipVideoTrack = [[composition tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
AVMutableVideoCompositionLayerInstruction *transformer =
[AVMutableVideoCompositionLayerInstruction videoCompositionLayerInstructionWithAssetTrack:clipVideoTrack];

instruction.layerInstructions = [NSArray arrayWithObject:transformer];
videoComposition.instructions = [NSArray arrayWithObject:instruction];
```
暂时没弄清楚这些所谓指令的意思，不过如果 `AVMutableVideoComposition`对象没有设置instructions的话运行时会报错。
```
Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '*** -[AVAssetExportSession setVideoComposition:] video composition must have composition instructions'
```

### 二. 视频导出
到这里 `AVMutableVideoComposition` 已经配置好了，水印也加上了，可以将视频导出来了。

用 `AVAssetExportSession` 对象将视频导出：
```objc
_exportSession = [[AVAssetExportSession alloc] initWithAsset:composition presetName:exportSize];
_exportSession.outputURL = exportURL;
_exportSession.outputFileType = AVFileTypeQuickTimeMovie;
_exportSession.shouldOptimizeForNetworkUse = YES;
_exportSession.videoComposition = videoComposition; /*刚才组装好的videoComposition*/
_exportSession.timeRange = CMTimeRangeMake(kCMTimeZero, timeDur);

__weak typeof(self) weakSelf = self;
[_exportSession exportAsynchronouslyWithCompletionHandler:^{
    typeof(weakSelf) strongSelf = weakSelf;
    if (_exportSession.status == AVAssetExportSessionStatusCompleted) {
        NSLog(@"export success    !!");
    }
}];
```

源码：[code](https://github.com/bt67123/AVFoundation_Watermark_Example)

使用真机调试

以上。