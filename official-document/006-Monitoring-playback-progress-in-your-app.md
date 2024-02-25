# Monitoring playback progress in your app
Observe the playback of a media asset to update your app’s user-interface state.

## Overview
미디어 재생 앱이 경우 플레이어 UI의 상태를 구동하거나 다른 작업을 수행하기 위해서 재생 진행 상황을 모니터링해야 합니다. 이 상태를 모니터링하려면 KVO보다 더 높은 수준의 정밀도가 필요합니다. 그래서 AVPlayer 재생 시간을 관찰할 수 있는 특정 API를 제공합니다. 이 문서에서는 일정한 간격으로 또는 재생이 특정 시간 경계를 넘을 때 이 상태를 관찰하는 방법을 설명합니다.

## Observe the current playback time at regular intervals
플레이어의 현재 시간을 관찰하는 가장 일반적인 방법은 일정한 간격으로 관찰하는 것입니다. 이러한 방식으로 관찰하는 것은 플레이어의 사용자 인터페이스에서 시간 표시 상태를 구동할 때 유용합니다.

플레이어의 현재 시간을 일정한 간격으로 관찰하려면 `addPeriodicTimeObserver(forInterval:queue:using:)` 메서드를 호출하면 됩니다. 이 메서드는 시간을 관찰할 간격을 나타내는 CMTime 값, 직렬 디스패치 큐, 플레이어가 지정된 시간 간격에 호출하는 콜백을 받습니다. 다음 예시는 플레이어가 정상 재생 중에 0.5초마다 호출하는 옵저버를 추가하는 예시입니다:

```swift
@Published private(set) var duration: TimeInterval = 0.0
@Published private(set) var currentTime: TimeInterval = 0.0


private let player = AVPlayer()
private var timeObserver: Any?


/// Adds an observer of the player timing.
private func addPeriodicTimeObserver() {
    // Create a 0.5 second interval time.
    let interval = CMTime(value: 1, timescale: 2)
    timeObserver = player.addPeriodicTimeObserver(forInterval: interval,
                                                  queue: .main) { [weak self] time in
        guard let self else { return }
        // Update the published currentTime and duration values.
        currentTime = time.seconds
        duration = player.currentItem?.duration.seconds ?? 0.0
    }
}


/// Removes the time observer from the player.
private func removePeriodicTimeObserver() {
    guard let timeObserver else { return }
    player.removeTimeObserver(timeObserver)
    self.timeObserver = nil
}
```

상태 모니터링이 끝나면 항상 플레이어의 `addPeriodicTimeObserver(forInterval:queue:using:)` 메서드 호출을 `removeTimeObserver(_:)` 호출과 페어링하세요. 이 규칙을 준수하지 않으면 정의되지 않은 동작이 발생합니다.

# Observe the playback of specific times within a media presentation

플레이어를 관찰하는 또 다른 방법은 재생 중에 플레이어가 특정 시간 경계를 넘을 때입니다. 플레이어 UI를 업데이트하거나 다른 작업을 수행하여 이러한 시간 경과에 대응할 수 있습니다.

플레이어가 미디어 타임라인의 특정 지점을 지날 때 앱에 알림을 보내려면 플레이어의 `addBoundaryTimeObserver(forTimes:queue:using:)` 메서드를 호출합니다. 이 메서드는 경계 시간을 정의하는 CMTime 값, 직렬 디스패치 큐 및 콜백 클로저를 래핑하는 NSValue 객체 배열을 받습니다. 다음 예제는 각 재생 분기에 대한 바운더리 시간을 정의하는 방법을 보여줍니다:

```swift
/// Adds an observer of the player traversing specific times during standard playback.
private func addBoundaryTimeObserver() async throws {
    
    // Asynchronously load the duration of the asset.
    let duration = try await asset.load(.duration)
    
    // Divide the asset's duration into quarters.
    let quarterDuration = CMTimeMultiplyByRatio(duration,
                                                multiplier: 1,
                                                divisor: 4)
    
    var currentTime = CMTime.zero
    var times = [NSValue]()
    
    // Calculate boundary times.
    while currentTime < duration {
        currentTime = currentTime + quarterDuration
        times.append(NSValue(time:currentTime))
    }
    
    timeObserver = player.addBoundaryTimeObserver(forTimes: times,
                                                  queue: .main) { [weak self] in
        // Update the percentage complete in the user interface.
    }
}
```

주기적 또는 경계 시간 관찰자를 추가하는 경우, 완료되면 `removeTimeObserver(_:)`를 호출하여 관찰을 제거해야 합니다.
