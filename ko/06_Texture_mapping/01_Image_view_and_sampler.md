이번 장에서는 그래픽스 파이프라인이 이미지를 샘플링하는 데 필요한 두 가지 리소스를 더 생성해 보겠습니다. 첫 번째 리소스는 스왑체인 이미지를 다루면서 이미 본 적이 있지만, 두 번째 리소스는 새로운 것입니다. 이는 셰이더가 이미지에서 텍셀을 어떻게 읽을지와 관련이 있습니다.

## 텍스처 이미지 뷰 (Texture image view)

이전에 스왑체인 이미지와 프레임버퍼에서 보았듯이, 이미지는 직접 접근하는 것이 아니라 이미지 뷰를 통해 접근합니다. 텍스처 이미지에도 이러한 이미지 뷰를 생성해야 합니다.

텍스처 이미지의 `VkImageView`를 담을 클래스 멤버를 추가하고, 이를 생성할 새로운 함수 `createTextureImageView`를 만듭니다:

```c++
VkImageView textureImageView;

...

void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createVertexBuffer();
    ...
}

...

void createTextureImageView() {

}
```

이 함수의 코드는 `createImageViews`를 그대로 기반으로 할 수 있습니다. 변경해야 할 부분은 `format`과 `image` 두 가지뿐입니다:

```c++
VkImageViewCreateInfo viewInfo{};
viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
viewInfo.image = textureImage;
viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
viewInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
viewInfo.subresourceRange.baseMipLevel = 0;
viewInfo.subresourceRange.levelCount = 1;
viewInfo.subresourceRange.baseArrayLayer = 0;
viewInfo.subresourceRange.layerCount = 1;
```

`VK_COMPONENT_SWIZZLE_IDENTITY`가 어차피 `0`으로 정의되어 있기 때문에 명시적인 `viewInfo.components` 초기화는 생략했습니다. `vkCreateImageView`를 호출하여 이미지 뷰 생성을 마칩니다:

```c++
if (vkCreateImageView(device, &viewInfo, nullptr, &textureImageView) != VK_SUCCESS) {
    throw std::runtime_error("failed to create texture image view!");
}
```

`createImageViews`와 많은 로직이 중복되므로, 이를 새로운 `createImageView` 함수로 추상화할 수 있습니다:

```c++
VkImageView createImageView(VkImage image, VkFormat format) {
    VkImageViewCreateInfo viewInfo{};
    viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
    viewInfo.image = image;
    viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
    viewInfo.format = format;
    viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    viewInfo.subresourceRange.baseMipLevel = 0;
    viewInfo.subresourceRange.levelCount = 1;
    viewInfo.subresourceRange.baseArrayLayer = 0;
    viewInfo.subresourceRange.layerCount = 1;

    VkImageView imageView;
    if (vkCreateImageView(device, &viewInfo, nullptr, &imageView) != VK_SUCCESS) {
        throw std::runtime_error("failed to create image view!");
    }

    return imageView;
}
```

이제 `createTextureImageView` 함수를 다음과 같이 단순화할 수 있습니다:

```c++
void createTextureImageView() {
    textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB);
}
```

그리고 `createImageViews`는 다음과 같이 단순화됩니다:

```c++
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

    for (uint32_t i = 0; i < swapChainImages.size(); i++) {
        swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat);
    }
}
```

프로그램이 끝날 때, 이미지 자체를 파괴하기 직전에 이미지 뷰를 파괴해야 합니다:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyImageView(device, textureImageView, nullptr);

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);
```

## 샘플러 (Samplers)

셰이더가 이미지에서 직접 텍셀을 읽는 것도 가능하지만, 텍스처로 사용될 때는 흔한 방법이 아닙니다. 텍스처는 보통 샘플러를 통해 접근하며, 샘플러는 필터링과 변환을 적용하여 최종적으로 검색될 색상을 계산합니다.

이러한 필터는 오버샘플링과 같은 문제를 해결하는 데 도움이 됩니다. 텍셀보다 프래그먼트가 더 많은 지오메트리에 매핑된 텍스처를 생각해 봅시다. 각 프래그먼트의 텍스처 좌표에 가장 가까운 텍셀을 그냥 사용한다면, 첫 번째 이미지와 같은 결과를 얻게 될 것입니다:

![](/images/texture_filtering.png)

가장 가까운 4개의 텍셀을 선형 보간으로 조합하면 오른쪽과 같이 더 부드러운 결과를 얻을 수 있습니다. 물론 애플리케이션의 아트 스타일에 따라 왼쪽 스타일이 더 적합할 수도 있지만(마인크래프트를 생각해보세요), 일반적인 그래픽 애플리케이션에서는 오른쪽 방식이 선호됩니다. 샘플러 객체는 텍스처에서 색상을 읽을 때 이 필터링을 자동으로 적용해 줍니다.

언더샘플링은 반대의 문제로, 프래그먼트보다 텍셀이 더 많은 경우입니다. 이는 체커보드 텍스처와 같은 고주파 패턴을 예리한 각도에서 샘플링할 때 아티팩트를 유발합니다:

![](/images/anisotropic_filtering.png)

왼쪽 이미지에서 보듯이, 텍스처가 멀리 갈수록 흐릿한 덩어리로 변합니다. 이에 대한 해결책은 [비등방성 필터링(anisotropic filtering)](https://en.wikipedia.org/wiki/Anisotropic_filtering)이며, 이 또한 샘플러에 의해 자동으로 적용될 수 있습니다.

이러한 필터 외에도 샘플러는 변환을 처리할 수도 있습니다. 샘플러는 *주소 지정 모드*를 통해 이미지 외부의 텍셀을 읽으려고 할 때 어떻게 처리할지 결정합니다. 아래 이미지는 몇 가지 가능한 옵션을 보여줍니다:

![](/images/texture_addressing.png)

이제 이러한 샘플러 객체를 설정하기 위해 `createTextureSampler` 함수를 생성하겠습니다. 나중에 셰이더에서 이 샘플러를 사용하여 텍스처의 색상을 읽게 될 것입니다.

```c++
void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createTextureSampler();
    ...
}

...

void createTextureSampler() {

}
```

샘플러는 `VkSamplerCreateInfo` 구조체를 통해 구성되며, 이 구조체는 적용해야 할 모든 필터와 변환을 명시합니다.

```c++
VkSamplerCreateInfo samplerInfo{};
samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
samplerInfo.magFilter = VK_FILTER_LINEAR;
samplerInfo.minFilter = VK_FILTER_LINEAR;
```

`magFilter`와 `minFilter` 필드는 확대(magnified)되거나 축소(minified)된 텍셀을 어떻게 보간할지 지정합니다. 확대는 위에서 설명한 오버샘플링 문제를, 축소는 언더샘플링 문제를 다룹니다. 선택지는 `VK_FILTER_NEAREST`와 `VK_FILTER_LINEAR`이며, 이는 위 이미지에서 시연된 모드에 해당합니다.

```c++
samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;
```

주소 지정 모드는 `addressMode` 필드를 사용하여 각 축(axis)별로 지정할 수 있습니다. 사용 가능한 값은 아래와 같습니다. 대부분은 위 이미지에서 시연되었습니다. 축이 X, Y, Z 대신 U, V, W로 불리는 점에 유의하세요. 이는 텍스처 공간 좌표에 대한 관례입니다.

*   `VK_SAMPLER_ADDRESS_MODE_REPEAT`: 이미지 크기를 벗어날 때 텍스처를 반복합니다.
*   `VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT`: repeat과 같지만, 크기를 벗어날 때 이미지를 미러링하기 위해 좌표를 반전시킵니다.
*   `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE`: 이미지 크기를 벗어난 좌표에 가장 가까운 가장자리의 색상을 사용합니다.
*   `VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE`: clamp to edge와 같지만, 가장 가까운 가장자리의 반대쪽 가장자리를 사용합니다.
*   `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER`: 이미지 크기 밖을 샘플링할 때 단색을 반환합니다.

이 튜토리얼에서는 이미지 외부를 샘플링하지 않을 것이므로 어떤 주소 지정 모드를 사용하든 상관없습니다. 하지만 repeat 모드는 바닥이나 벽과 같은 텍스처를 타일링하는 데 사용될 수 있기 때문에 아마도 가장 일반적인 모드일 것입니다.

```c++
samplerInfo.anisotropyEnable = VK_TRUE;
samplerInfo.maxAnisotropy = ???;
```

이 두 필드는 비등방성 필터링을 사용할지 여부를 지정합니다. 성능이 우려되는 경우가 아니라면 사용하지 않을 이유가 없습니다. `maxAnisotropy` 필드는 최종 색상을 계산하는 데 사용할 수 있는 텍셀 샘플의 양을 제한합니다. 값이 낮을수록 성능은 좋아지지만 품질은 떨어집니다. 어떤 값을 사용할 수 있는지 알아내려면 다음과 같이 물리 디바이스의 속성을 가져와야 합니다:

```c++
VkPhysicalDeviceProperties properties{};
vkGetPhysicalDeviceProperties(physicalDevice, &properties);
```

`VkPhysicalDeviceProperties` 구조체의 문서를 보면 `limits`라는 이름의 `VkPhysicalDeviceLimits` 멤버가 포함되어 있습니다. 이 구조체에는 `maxSamplerAnisotropy`라는 멤버가 있으며, 이것이 `maxAnisotropy`에 지정할 수 있는 최대값입니다. 최고 품질을 원한다면 이 값을 직접 사용하면 됩니다:

```c++
samplerInfo.maxAnisotropy = properties.limits.maxSamplerAnisotropy;
```

프로그램 시작 시에 속성을 쿼리하여 필요한 함수에 전달하거나, `createTextureSampler` 함수 자체에서 쿼리할 수 있습니다.

```c++
samplerInfo.borderColor = VK_BORDER_COLOR_INT_OPAQUE_BLACK;
```

`borderColor` 필드는 clamp to border 주소 지정 모드로 이미지 외부를 샘플링할 때 반환되는 색상을 지정합니다. float 또는 int 형식으로 검은색, 흰색 또는 투명을 반환할 수 있습니다. 임의의 색상을 지정할 수는 없습니다.

```c++
samplerInfo.unnormalizedCoordinates = VK_FALSE;
```

`unnormalizedCoordinates` 필드는 이미지의 텍셀 주소를 지정하는 데 사용할 좌표계를 지정합니다. 이 필드가 `VK_TRUE`이면 `[0, texWidth)` 및 `[0, texHeight)` 범위 내의 좌표를 그대로 사용할 수 있습니다. `VK_FALSE`이면 모든 축에서 `[0, 1)` 범위를 사용하여 텍셀 주소를 지정합니다. 실제 애플리케이션에서는 거의 항상 정규화된 좌표를 사용합니다. 그래야 해상도가 다른 텍스처들을 정확히 동일한 좌표로 사용할 수 있기 때문입니다.

```c++
samplerInfo.compareEnable = VK_FALSE;
samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;
```

비교 함수가 활성화되면 텍셀은 먼저 특정 값과 비교되고, 그 비교 결과가 필터링 연산에 사용됩니다. 이는 주로 섀도 맵의 [percentage-closer filtering](https://developer.nvidia.com/gpugems/GPUGems/gpugems_ch11.html)에 사용됩니다. 이에 대해서는 나중 장에서 살펴보겠습니다.

```c++
samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
samplerInfo.mipLodBias = 0.0f;
samplerInfo.minLod = 0.0f;
samplerInfo.maxLod = 0.0f;
```

이 필드들은 모두 밉매핑(mipmapping)에 적용됩니다. 밉매핑에 대해서는 [나중 장](/Generating_Mipmaps)에서 다룰 것이지만, 기본적으로 적용할 수 있는 또 다른 유형의 필터입니다.

이제 샘플러의 작동 방식이 완전히 정의되었습니다. 샘플러 객체의 핸들을 담을 클래스 멤버를 추가하고 `vkCreateSampler`로 샘플러를 생성합니다:

```c++
VkImageView textureImageView;
VkSampler textureSampler;

...

void createTextureSampler() {
    ...

    if (vkCreateSampler(device, &samplerInfo, nullptr, &textureSampler) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture sampler!");
    }
}
```

샘플러는 어디에서도 `VkImage`를 참조하지 않는다는 점에 유의하세요. 샘플러는 텍스처에서 색상을 추출하는 인터페이스를 제공하는 별개의 객체입니다. 1D, 2D, 3D 등 원하는 모든 이미지에 적용할 수 있습니다. 이는 텍스처 이미지와 필터링을 단일 상태로 결합했던 많은 구형 API와는 다른 점입니다.

더 이상 이미지에 접근하지 않을 프로그램의 끝에서 샘플러를 파괴합니다:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroySampler(device, textureSampler, nullptr);
    vkDestroyImageView(device, textureImageView, nullptr);

    ...
}
```

## 비등방성 디바이스 기능 (Anisotropy device feature)

지금 프로그램을 실행하면 다음과 같은 유효성 검사 레이어 메시지가 나타날 것입니다:

![](/images/validation_layer_anisotropy.png)

이는 비등방성 필터링이 사실 선택적 디바이스 기능이기 때문입니다. 이를 요청하도록 `createLogicalDevice` 함수를 업데이트해야 합니다:

```c++
VkPhysicalDeviceFeatures deviceFeatures{};
deviceFeatures.samplerAnisotropy = VK_TRUE;
```

그리고 최신 그래픽 카드가 이를 지원하지 않을 가능성은 매우 낮지만, `isDeviceSuitable`을 업데이트하여 사용 가능한지 확인해야 합니다:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    ...

    VkPhysicalDeviceFeatures supportedFeatures;
    vkGetPhysicalDeviceFeatures(device, &supportedFeatures);

    return indices.isComplete() && extensionsSupported && swapChainAdequate && supportedFeatures.samplerAnisotropy;
}
```

`vkGetPhysicalDeviceFeatures`는 `VkPhysicalDeviceFeatures` 구조체를 재사용하여, boolean 값을 설정함으로써 요청된 기능이 아닌 지원되는 기능을 나타냅니다.

비등방성 필터링의 가용성을 강제하는 대신, 다음과 같이 조건부로 설정하여 사용하지 않도록 할 수도 있습니다:

```c++
samplerInfo.anisotropyEnable = VK_FALSE;
samplerInfo.maxAnisotropy = 1.0f;
```

다음 장에서는 이미지와 샘플러 객체를 셰이더에 노출하여 사각형에 텍스처를 그리겠습니다.

[C++ 코드](/code/25_sampler.cpp) /
[버텍스 셰이더](/code/22_shader_ubo.vert) /
[프래그먼트 셰이더](/code/22_shader_ubo.frag)