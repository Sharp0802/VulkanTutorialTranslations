렌더 파이프라인에서 스왑 체인의 이미지를 포함한 모든 `VkImage`를 사용하려면 `VkImageView` 객체를 생성해야 합니다. 이미지 뷰는 말 그대로 이미지를 들여다보는 창(view)입니다. 이미지에 접근하는 방법과 접근할 이미지의 부분(예: 밉매핑 레벨이 없는 2D 텍스처나 깊이 텍스처로 처리할지 여부)을 기술합니다.

이번 장에서는 스왑 체인의 모든 이미지에 대해 기본적인 이미지 뷰를 생성하는 `createImageViews` 함수를 작성하여, 나중에 이를 색상 타겟으로 사용할 수 있도록 할 것입니다.

먼저 이미지 뷰를 저장할 클래스 멤버를 추가합니다.

```c++
std::vector<VkImageView> swapChainImageViews;
```

`createImageViews` 함수를 만들고 스왑 체인 생성 직후에 호출합니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
}

void createImageViews() {

}
```

가장 먼저 할 일은 우리가 생성할 모든 이미지 뷰를 담을 수 있도록 리스트의 크기를 조절하는 것입니다.

```c++
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

}
```

다음으로, 스왑 체인의 모든 이미지를 순회하는 루프를 설정합니다.

```c++
for (size_t i = 0; i < swapChainImages.size(); i++) {

}
```

이미지 뷰 생성을 위한 매개변수는 `VkImageViewCreateInfo` 구조체에 명시됩니다. 처음 몇 개의 매개변수는 간단합니다.

```c++
VkImageViewCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
createInfo.image = swapChainImages[i];
```

`viewType`과 `format` 필드는 이미지 데이터를 어떻게 해석할지를 지정합니다. `viewType` 매개변수를 사용하면 이미지를 1D 텍스처, 2D 텍스처, 3D 텍스처 및 큐브 맵으로 처리할 수 있습니다.

```c++
createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
createInfo.format = swapChainImageFormat;
```

`components` 필드를 사용하면 색상 채널을 재배치(swizzle)할 수 있습니다. 예를 들어, 흑백 텍스처를 위해 모든 채널을 적색(red) 채널에 매핑할 수 있습니다. 또한 `0`과 `1` 같은 상수 값을 특정 채널에 매핑할 수도 있습니다. 여기서는 기본 매핑을 그대로 사용할 것입니다.

```c++
createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
```

`subresourceRange` 필드는 이미지의 용도와 접근해야 할 이미지의 부분을 기술합니다. 우리가 만들 이미지는 밉매핑 레벨이나 다중 레이어 없이 색상 타겟으로 사용될 것입니다.

```c++
createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.levelCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.layerCount = 1;
```

만약 스테레오스코픽 3D 애플리케이션을 개발한다면, 다중 레이어를 가진 스왑 체인을 생성할 것입니다. 그런 다음 각 이미지에 대해 여러 이미지 뷰를 생성하여, 서로 다른 레이어에 접근함으로써 왼쪽 눈과 오른쪽 눈에 대한 뷰를 표현할 수 있습니다.

이제 `vkCreateImageView`를 호출하여 이미지 뷰를 생성합니다.

```c++
if (vkCreateImageView(device, &createInfo, nullptr, &swapChainImageViews[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image views!");
}
```

이미지와 달리, 이미지 뷰는 우리가 명시적으로 생성했으므로 프로그램이 끝날 때 이를 다시 파괴하기 위한 비슷한 루프를 추가해야 합니다.

```c++
void cleanup() {
    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    ...
}
```

이미지를 텍스처로 사용하기 시작하는 데에는 이미지 뷰만으로도 충분하지만, 렌더 타겟으로 사용하기에는 아직 준비가 되지 않았습니다. 이를 위해서는 프레임버퍼라고 알려진 한 단계의 간접(indirection) 과정이 더 필요합니다. 하지만 그 전에 먼저 그래픽 파이프라인을 설정해야 합니다.

[C++ 코드](/code/07_image_views.cpp)