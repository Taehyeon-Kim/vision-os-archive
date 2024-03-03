# Media Asset의 MV-HEVC 사양 검사

## MV-HEVC
- MV-HEVC 미디어 파일에는 왼쪽 눈과 오른쪽 눈에 각각 하나씩 Stereoscopic 프레임을 생성해서 깊이감을 주고 3D 비디오를 만들 수 있는 정보가 포함되어 있습니다.
- 이는 visionOS에서 3D 비디오를 표시하기 위한 표준 포맷입니다.
- MPEG-4 또는 QuickTime 파일로 인코딩됩니다.

## MV-HEVC 사양을 충족하고 유효한 Stereo 데이터가 있는지 검사
- AVURLAsset으로 파일을 먼저 로딩합니다.
- Track이 읽을 수 있는 비디오 데이터인지 확인합니다.
- 앱은 `loadTracks(withMediaCharacteristic:completionHandler:)`를 호출해서 `containStereoMultiviewVideo track`을 요청합니다.

```swift
// ...
/// asset은 AVURLAsset 타입입니다.
if let track = try await asset.loadTracks(withMediaCharacteristic: .containsStereoMultiviewVideo).first {
    self.track = track
}
// ...
```