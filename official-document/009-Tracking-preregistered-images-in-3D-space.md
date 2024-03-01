# Tracking preregistered images in 3D space

> Place content based on the current position of a known image in a person’s surroundings.  
> 사용자 주변의 이미지의 현재 위치를 기반으로 콘텐츠를 배치합니다.

## Overview
- 2D 이미지 추적에 대한 ARKit 지원을 사용해서 공간에 3D 콘텐츠를 배치할 수 있습니다.
- ARKit은 사람을 기준으로 이미지가 이동함에 따라 이미지의 위치를 업데이트합니다.
- 앱의 Asset Catalog에 레퍼런스 이미지를 하나 이상 제공하면 사람들은 해당 이미지의 복사본을 사용해서 앱에 가상 3D 콘텐츠를 배치할 수 있습니다.
- 예를 들어서 커스텀 놀이 카드 팩을 디자인하고 이러한 에셋을 실제 카드 덱의 형태로 사람들에게 제공하면 완전히 몰입감 있는 환경에서 카드별로 고유한 콘텐츠를 배치할 수 있습니다.

<br />

다음의 예시는 앱의 Asset Catalog에서 로드된 이미지 세트를 추적합니다:
```swift
/// ARKitSession 생성, Image 추적을 위한 Provider 생성
let session = ARKitSession()
let imageInfo = ImageTrackingProvider(
    referenceImages: ReferenceImage.loadReferenceImages(inGroupNamed: "playingcard-photos")
)


if ImageTrackingProvider.isSupported {
    Task {
        try await session.run([imageInfo])
        for await update in imageInfo.anchorUpdates {
            updateImage(update.anchor)
        }
    }
}


func updateImage(_ anchor: ImageAnchor) {
    if imageAnchors[anchor.id] == nil {
        // Add a new entity to represent this image.
        let entity = ModelEntity(mesh: .generateSphere(radius: 0.05))
        entityMap[anchor.id] = entity
        rootEntity.addChild(entity)
    }
    
    if anchor.isTracked {
        entityMap[anchor.id]?.transform = Transform(matrix: anchor.originFromAnchorTransform)
    }
}
```

추적 중인 이미지이 실제 크기를 알고 있는 경우 physicalSize 프로퍼티를 사용해서 추적 정확도를 높일 수 있습니다. `estimatedScaleFactor` 프로퍼티는 추적된 이미지의 크기가 사용자가 제공한 예상 물리적 크기와 어떻게 다른지에 대한 정보를 제공합니다.