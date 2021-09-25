---
title: iOS丨使用AVPlayer播放视频
date: 2020-09-08
categories: ['iOS']
draft: false
---

## 背景
播放视频，一个听起来十分合理的需求。iOS提供了AVFoundation，其中的AVPlayer提供了播放视频的能力。在使用的时候我们碰到了一些问题，记录如下。
<!--more-->
## 属性和方法
### AVPlayer
AVPlayer是AVassets生命周期的控制器。它可以通过两个方式来创建：通过url、或是通过AVPlayerItem。
```swift
public init(url URL: URL)
public init(playerItem item: AVPlayerItem?)
```
前者可以立刻获得一个对应资源的AVplayer的实例，一般用于生成一个新的PlayItem。后者可以通过一个已经存在的PlayItem来进行创建。

### AVPlayItem
对一个AV资源的管理类。内部包含了一些方便的方法和属性。
```swift
open var asset: AVAsset { get }
@available(iOS 4.3, *)
open var duration: CMTime { get }
open var presentationSize: CGSize { get }

open func currentTime() -> CMTime
open var forwardPlaybackEndTime: CMTime
open var reversePlaybackEndTime: CMTime
open var seekableTimeRanges: [NSValue] { get }

@available(iOS 5.0, *)
open func seek(to time: CMTime, completionHandler: ((Bool) -> Void)? = nil)
```
`seek`方法可以注意下，我们可以使用这个方法来修改视频的进度，其原理和文件的seek是一致的。

## 使用
首先生成一个AVPlayItem的实例，我们可以直接在包体内直接索引。找到对应文件的url。在生成一个AVPlayItem。
```swift
guard let url = Bundle.main.url(forResource: "key", withExtension: "mp4") else {
            return
}
```

在播放视频之前，一个比较好的做法是先生成一张静态的图片作为兜底。这张图片可以是视频的第一帧，这样一旦视频的view出现了什么问题，屏幕上还是展示视频相关的内容。

这个时候可以使用`AVAssetImageGenerator`，本质上是一个`NSObject`，英语提供生成缩略图或者预览图。通过调用
```swift
func generateCGImagesAsynchronously(forTimes requestedTimes:  [NSValue], 
                  completionHandler handler: @escaping  [AVAssetImageGeneratorCompletionHandler
```
给定一段时间，通过这段时间来生成对应的image。一般来说使用前面的第一帧就足够了。

同时可以这个类和方法可以提前返回一个小视频。比如说我们想要播放一个帧数比较高的视频，同时这个视频可能比较大，在加载的时候就会较长的黑屏状态。这显然是不太好的，我们可以通过上面的方法，将视频分段的抽帧，组合成一个帧数较少的小视频。当然这个视频的表现可能没有原先的视频表现好，但是可以先保证有视频展示在屏幕上。

接下来整个视频需要在一个UIView里面进行播放，注意生成这个View，添加进去`imageView`即可。这里需要设置一下`imageView`的`contentMode`，不然可能导致图片和视频显示的区域大小不同。

展示视频的是一个View，但是真正播放视频的是一个CALayer。因为它并不处理交互，而仅仅是展示。我们通过AVPlayer创建一个AVPlayerLayer，设置它的frame，注意这里也要设置它的videoGravity。这个属性的意义是调整视频的contentMode。最后将这个layer添加到playview上面去。

这里一定要注意的一点是：**AVPlayer的View会盖住别的同层级的view**，并且是整屏的覆盖。不管是设置了多大的frame，都会进行覆盖。因此要将这个playview添加到所有想要可见的view之后，在添加的这个层级上，将这个playview放置在后面。
```swift
self?.view?.addSubview(playerView)
self?.view?.sendSubviewToBack(playerView)
```

