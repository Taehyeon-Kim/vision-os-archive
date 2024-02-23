# Controlling the transport behavior of a player
Play, pause, and seek through a media presentation.

## Overview
- AVFoundation은 로컬, 원격 기반 미디어와 HLS 미디어를 포함한 미디어 에셋 재생에 대한 포괄적인 지원을 제공합니다.
- 이 프레임워크는 유형이나 위치에 관계없이 미디어를 로드하고 검사할 수 있는 일관된 인터페이스를 제공하는 AVAsset 클래스를 사용해서 미디어 에셋을 모델링합니다.
- AVPlayer 객체를 사용하면 `currentTime()`과 같은 에셋의 동적 상태를 모델링하는 AVPlayerItem 객체 형태로 미디어 에셋을 재생할 수 있습니다.
- 커스텀 플레이어 UI를 구축하거나 재생을 프로그래밍 방식으로 제어해야 하는 경우 AVPlayer를 효과적으로 사용하는 방법을 이해하는 것은 필수입니다.

## Observe playback readiness
플레이어 항목을 생성하면 `AVPlayerItem.Status.unknown`로 시작합니다. 이는 시스템이 재생을 위해 미디어 로드를 시도하지 않았음을 의미합니다. PlayerItem을 AVPlayer에 연결해야만 시스템에서 에셋의 미디어를 로드하기 시작합니다. 

PlayerItem이 재생할 준비가 되었는지 확인하려면 `status` 프로퍼티 값을 관찰하세요. PlayerItem을 Player와 연결하는 것은 Item의 미디어를 로드하라는 시스템의 신호입니다. Player의 `replaceCurrentItem(with:)`를 호출하기 전에 이 관찰을 추가하세요.

```swift
func playMedia(at url: URL) {
    let asset = AVAsset(url: url)
    let playerItem = AVPlayerItem(
        asset: asset,
        automaticallyLoadedAssetKeys: [.tracks, .duration, .commonMetadata]
    )
    // Register to observe the status property before associating with player.
    playerItem.publisher(for: \.status)
        .removeDuplicates()
        .receive(on: DispatchQueue.main)
        .sink { [weak self] status in
            guard let self else { return }
            switch status {
            case .readyToPlay:
                // Ready to play. Present playback UI.
            case .failed:
                // A failure while loading media occurred.
            default:
                break
            }
        }
        .store(in: &subscriptions)
    
    // Set the item as the player's current item.
    player.replaceCurrentItem(with: playerItem)
}
```

Player item이 `AVPlayerItem.Status.readyToPlay` 상태에 도달하면 재생 UI를 표시하거나 활성화합니다. 또는 실패가 발생하면 플레이어에 적절한 상태를 표시합니다.

상태는 총 3가지가 있습니다:
- `.unknown`
- `.readyToPlay`
- `.failed`

## Control the playback rate

Player는 재생 속도를 제어하는 수단으로 `play()`와 `pause()` 메서드를 제공합니다. 각각 재생과 일시정지를 의미합니다. Player Item이 재생할 준비가 되면 플레이어의 play() 메서드를 호출하여 초기값이 1.0인 defaultRate로 재생을 시작하도록 요청합니다.

기본적으로 Player는 미디어 데이터가 충분히 확보될 때까지 자동으로 재생 시작을 대기하여 지연을 최소화합니다. Player가 일시 중지 상태인지, 재생 대기 상태인지, 재생 중인지 여부는 timeControlStatus 값을 관찰하여 확인할 수 있습니다:

```swift
@Published var isPlaying = false

private func observePlayingState() {
    player.publisher(for: \.timeControlStatus)
        .receive(on: DispatchQueue.main)
        .map { $0 == .playing }
        .assign(to: &$isPlaying)
}
```

`rateDidChangeNotification` 타입의 notifications을 관찰하여 `rate` 프로퍼티의 변경 사항을 관찰합니다. 이 notifications을 관찰하는 것은 키 값으로 `rate` 프로퍼티를 관찰하는 것과 비슷하지만 `rate` 프로퍼티 변경 이유에 대한 추가 정보를 제공합니다. `rateDidChangeReasonKey` 상수를 사용하여 notification의 userInfo 딕셔너리에서 이유를 검색합니다:

```swift
// Observe changes to the playback rate asynchronously.
private func observeRateChanges() async {
    let name = AVPlayer.rateDidChangeNotification
    for await notification in NotificationCenter.default.notifications(named: name) {
        guard let reason = notification.userInfo?[AVPlayer.rateDidChangeReasonKey] as? AVPlayer.RateDidChangeReason else {
            continue
        }
        switch reason {
        case .appBackgrounded:
            // The app transitioned to the background.
        case .audioSessionInterrupted:
            // The system interrupts the app’s audio session.
        case .setRateCalled:
            // The app set the player’s rate.
        case .setRateFailed:
            // An attempt to change the player’s rate failed.
        default:
            break
        }
    }
}
```

## Seek through the media timeline

AVPlayer 및 AVPlayerItem의 메서드를 사용하여 여러 가지 방법으로 미디어 타임라인을 탐색할 수 있습니다. 가장 일반적인 방법은 플레이어의 seek(to:) 메서드를 사용하여 대상 CMTime 값을 전달하는 것입니다. 이 메서드는 비동기 컨텍스트에서 호출합니다:


```swift
// Handle time update request from user interface.
func seek(to timeInterval: TimeInterval) async {
    // Create a CMTime value for the passed in time interval.
    let time = CMTime(seconds: timeInterval, preferredTimescale: 600)
    await avPlayer.seek(to: time)
}
```

이 메서드를 한 번만 호출하여 위치를 찾을 수도 있지만 Slider View를 사용할 때처럼 연속적으로 호출할 수도 있습니다. (커스텀 UI를 구성한다면 반드시 필요한 메서드입니다.)

`seek(to:)` 메서드는 프레젠테이션을 빠르게 탐색할 수 있는 편리한 방법이지만, 정확성보다는 속도에 맞춰져 있습니다.

즉, 플레이어가 실제로 찾는 시간은 사용자가 요청한 시간과 약간 다를 수 있습니다. 정확한 탐색 동작을 구현해야 하는 경우, 목표 시간(전후)에서 허용되는 편차 범위를 지정할 수 있는 `seek(to:toleranceBefore:toleranceAfter:)` 메서드를 사용하면 됩니다. 예를 들어 샘플 수준의 정확한 탐색 동작을 제공해야 하는 경우 허용 오차 값을 0으로 지정합니다:

```swift
// Seek precisely to the specified time.
await avPlayer.seek(to: time, toleranceBefore: .zero, toleranceAfter: .zero)
```
이 코드를 활용하여 정확한 탐색을 할 수 있습니다.