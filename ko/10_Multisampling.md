## 소개

이제 우리 프로그램은 텍스처에 대해 여러 레벨의 디테일(LOD)을 로드할 수 있게 되어, 뷰어에서 멀리 떨어진 객체를 렌더링할 때 발생하는 아티팩트를 수정합니다. 이제 이미지가 훨씬 부드러워졌지만, 자세히 보면 그려진 기하학적 도형의 가장자리를 따라 들쭉날쭉한 톱니 모양의 패턴을 발견할 수 있습니다. 이 현상은 초반에 사각형을 렌더링했던 프로그램에서 특히 잘 보입니다:

![](/images/texcoord_visualization.png)

이러한 원치 않는 효과를 "에일리어싱(aliasing)"이라고 부르며, 렌더링에 사용할 수 있는 픽셀 수가 제한적이기 때문에 발생합니다. 무한한 해상도를 가진 디스플레이는 없으므로, 어느 정도는 항상 눈에 띄게 될 것입니다. 이 문제를 해결하는 여러 방법이 있으며, 이번 장에서는 가장 널리 사용되는 방법 중 하나인 [멀티샘플 안티에일리어싱](https://en.wikipedia.org/wiki/Multisample_anti-aliasing)(MSAA)에 초점을 맞출 것입니다.

일반적인 렌더링에서는 픽셀 색상이 단일 샘플 포인트에 기반하여 결정되며, 대부분의 경우 이 포인트는 화면의 대상 픽셀 중앙에 위치합니다. 만약 그려진 선의 일부가 특정 픽셀을 지나가지만 샘플 포인트를 덮지 않는다면, 그 픽셀은 비어있게 되어 들쭉날쭉한 "계단 현상"을 유발합니다.

![](/images/aliasing.png)

MSAA는 이름에서 알 수 있듯이 픽셀당 여러 개의 샘플 포인트를 사용하여 최종 색상을 결정합니다. 예상할 수 있듯이, 샘플이 많을수록 결과는 좋아지지만, 계산 비용도 더 많이 듭니다.

![](/images/antialiasing.png)

우리의 구현에서는 사용 가능한 최대 샘플 수를 사용하는 데 중점을 둘 것입니다. 애플리케이션에 따라 이것이 항상 최선의 접근 방식은 아닐 수 있으며, 최종 결과가 품질 요구사항을 충족한다면 더 높은 성능을 위해 더 적은 샘플을 사용하는 것이 더 나을 수 있습니다.

## 사용 가능한 샘플 수 얻기

먼저 하드웨어가 얼마나 많은 샘플을 사용할 수 있는지 확인하는 것부터 시작하겠습니다. 대부분의 최신 GPU는 최소 8개의 샘플을 지원하지만, 이 숫자가 모든 곳에서 동일하다고 보장되지는 않습니다. 새로운 클래스 멤버를 추가하여 이 값을 추적하겠습니다:

```c++
...
VkSampleCountFlagBits msaaSamples = VK_SAMPLE_COUNT_1_BIT;
...
```

기본적으로 픽셀당 하나의 샘플만 사용할 것이며, 이는 멀티샘플링을 사용하지 않는 것과 같습니다. 이 경우 최종 이미지는 변경되지 않습니다. 정확한 최대 샘플 수는 선택한 물리적 장치와 연관된 `VkPhysicalDeviceProperties`에서 추출할 수 있습니다. 우리는 깊이 버퍼를 사용하고 있으므로, 색상과 깊이 모두에 대한 샘플 수를 고려해야 합니다. 둘 다 지원하는(&) 가장 높은 샘플 수가 우리가 지원할 수 있는 최대치가 됩니다. 이 정보를 가져올 함수를 추가합니다:

```c++
VkSampleCountFlagBits getMaxUsableSampleCount() {
    VkPhysicalDeviceProperties physicalDeviceProperties;
    vkGetPhysicalDeviceProperties(physicalDevice, &physicalDeviceProperties);

    VkSampleCountFlags counts = physicalDeviceProperties.limits.framebufferColorSampleCounts & physicalDeviceProperties.limits.framebufferDepthSampleCounts;
    if (counts & VK_SAMPLE_COUNT_64_BIT) { return VK_SAMPLE_COUNT_64_BIT; }
    if (counts & VK_SAMPLE_COUNT_32_BIT) { return VK_SAMPLE_COUNT_32_BIT; }
    if (counts & VK_SAMPLE_COUNT_16_BIT) { return VK_SAMPLE_COUNT_16_BIT; }
    if (counts & VK_SAMPLE_COUNT_8_BIT) { return VK_SAMPLE_COUNT_8_BIT; }
    if (counts & VK_SAMPLE_COUNT_4_BIT) { return VK_SAMPLE_COUNT_4_BIT; }
    if (counts & VK_SAMPLE_COUNT_2_BIT) { return VK_SAMPLE_COUNT_2_BIT; }

    return VK_SAMPLE_COUNT_1_BIT;
}
```

이제 물리적 장치 선택 과정에서 이 함수를 사용하여 `msaaSamples` 변수를 설정할 것입니다. 이를 위해, `pickPhysicalDevice` 함수를 약간 수정해야 합니다:

```c++
void pickPhysicalDevice() {
    ...
    for (const auto& device : devices) {
        if (isDeviceSuitable(device)) {
            physicalDevice = device;
            msaaSamples = getMaxUsableSampleCount();
            break;
        }
    }
    ...
}
```

## 렌더 타겟 설정

MSAA에서는 각 픽셀이 오프스크린 버퍼에서 샘플링된 후, 이 버퍼가 화면에 렌더링됩니다. 이 새로운 버퍼는 우리가 지금까지 렌더링해 온 일반적인 이미지와는 약간 다릅니다. 픽셀당 하나 이상의 샘플을 저장할 수 있어야 합니다. 멀티샘플 버퍼가 생성되면, 기본 프레임버퍼(픽셀당 단일 샘플만 저장)로 리졸브(resolve)되어야 합니다. 이것이 우리가 추가적인 렌더 타겟을 만들고 현재의 드로잉 프로세스를 수정해야 하는 이유입니다. 깊이 버퍼와 마찬가지로 한 번에 하나의 드로잉 작업만 활성화되므로, 렌더 타겟은 하나만 필요합니다. 다음 클래스 멤버를 추가합니다:

```c++
...
VkImage colorImage;
VkDeviceMemory colorImageMemory;
VkImageView colorImageView;
...
```

이 새로운 이미지는 픽셀당 원하는 수의 샘플을 저장해야 하므로, 이미지 생성 과정에서 이 숫자를 `VkImageCreateInfo`에 전달해야 합니다. `createImage` 함수에 `numSamples` 매개변수를 추가하여 수정합니다:

```c++
void createImage(uint32_t width, uint32_t height, uint32_t mipLevels, VkSampleCountFlagBits numSamples, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory) {
    ...
    imageInfo.samples = numSamples;
    ...
```

우선, 이 함수에 대한 모든 호출을 `VK_SAMPLE_COUNT_1_BIT`를 사용하도록 업데이트합니다. 구현을 진행하면서 적절한 값으로 교체할 것입니다:

```c++
createImage(swapChainExtent.width, swapChainExtent.height, 1, VK_SAMPLE_COUNT_1_BIT, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
...
createImage(texWidth, texHeight, mipLevels, VK_SAMPLE_COUNT_1_BIT, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
```

이제 멀티샘플 색상 버퍼를 생성하겠습니다. `createColorResources` 함수를 추가하고, 여기서 `createImage`의 함수 매개변수로 `msaaSamples`를 사용한다는 점에 유의하세요. 또한, 픽셀당 샘플이 하나 이상인 이미지의 경우 Vulkan 사양에 따라 강제되므로 밉 레벨은 하나만 사용합니다. 그리고 이 색상 버퍼는 텍스처로 사용되지 않으므로 밉맵이 필요 없습니다:

```c++
void createColorResources() {
    VkFormat colorFormat = swapChainImageFormat;

    createImage(swapChainExtent.width, swapChainExtent.height, 1, msaaSamples, colorFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT | VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, colorImage, colorImageMemory);
    colorImageView = createImageView(colorImage, colorFormat, VK_IMAGE_ASPECT_COLOR_BIT, 1);
}
```

일관성을 위해, `createDepthResources` 바로 전에 이 함수를 호출합니다:

```c++
void initVulkan() {
    ...
    createColorResources();
    createDepthResources();
    ...
}
```

이제 멀티샘플 색상 버퍼가 준비되었으니 깊이를 처리할 차례입니다. `createDepthResources`를 수정하여 깊이 버퍼가 사용하는 샘플 수를 업데이트합니다:

```c++
void createDepthResources() {
    ...
    createImage(swapChainExtent.width, swapChainExtent.height, 1, msaaSamples, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
    ...
}
```

이제 몇 가지 새로운 Vulkan 리소스를 생성했으므로, 필요할 때 해제하는 것을 잊지 말아야 합니다:

```c++
void cleanupSwapChain() {
    vkDestroyImageView(device, colorImageView, nullptr);
    vkDestroyImage(device, colorImage, nullptr);
    vkFreeMemory(device, colorImageMemory, nullptr);
    ...
}
```

그리고 `recreateSwapChain`을 업데이트하여 창 크기가 조절될 때 새로운 색상 이미지가 올바른 해상도로 다시 생성되도록 합니다:

```c++
void recreateSwapChain() {
    ...
    createImageViews();
    createColorResources();
    createDepthResources();
    ...
}
```

초기 MSAA 설정을 마쳤습니다. 이제 이 새로운 리소스를 그래픽 파이프라인, 프레임버퍼, 렌더 패스에서 사용하여 결과를 확인해야 합니다!

## 새로운 어태치먼트 추가

먼저 렌더 패스를 처리합시다. `createRenderPass`를 수정하여 색상 및 깊이 어태치먼트 생성 정보 구조체를 업데이트합니다:

```c++
void createRenderPass() {
    ...
    colorAttachment.samples = msaaSamples;
    colorAttachment.finalLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    ...
    depthAttachment.samples = msaaSamples;
    ...
```

`finalLayout`을 `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`에서 `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`로 변경한 것을 알 수 있습니다. 이는 멀티샘플 이미지를 직접 화면에 표시(present)할 수 없기 때문입니다. 먼저 일반 이미지로 리졸브해야 합니다. 이 요구사항은 어떤 시점에서도 화면에 표시되지 않을 깊이 버퍼에는 적용되지 않습니다. 따라서 색상을 위해 소위 리졸브 어태치먼트(resolve attachment)라는 새로운 어태치먼트를 하나만 추가해야 합니다:

```c++
    ...
    VkAttachmentDescription colorAttachmentResolve{};
    colorAttachmentResolve.format = swapChainImageFormat;
    colorAttachmentResolve.samples = VK_SAMPLE_COUNT_1_BIT;
    colorAttachmentResolve.loadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    colorAttachmentResolve.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
    colorAttachmentResolve.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    colorAttachmentResolve.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    colorAttachmentResolve.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    colorAttachmentResolve.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
    ...
```

이제 렌더 패스가 멀티샘플 색상 이미지를 일반 어태치먼트로 리졸브하도록 지시해야 합니다. 리졸브 대상이 될 색상 버퍼를 가리키는 새로운 어태치먼트 참조를 생성합니다:

```c++
    ...
    VkAttachmentReference colorAttachmentResolveRef{};
    colorAttachmentResolveRef.attachment = 2;
    colorAttachmentResolveRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    ...
```

`pResolveAttachments` 서브패스 구조체 멤버가 새로 생성된 어태치먼트 참조를 가리키도록 설정합니다. 이것만으로도 렌더 패스가 멀티샘플 리졸브 작업을 정의하여 이미지를 화면에 렌더링할 수 있게 됩니다:

```
    ...
    subpass.pResolveAttachments = &colorAttachmentResolveRef;
    ...
```

멀티샘플 색상 이미지를 재사용하고 있으므로, `VkSubpassDependency`의 `srcAccessMask`를 업데이트해야 합니다. 이 업데이트는 색상 어태치먼트에 대한 쓰기 작업이 다음 작업이 시작되기 전에 완료되도록 보장하여, 불안정한 렌더링 결과를 초래할 수 있는 쓰기-후-쓰기(write-after-write) 위험을 방지합니다:

```c++
    ...
    dependency.srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
    ...
```

이제 렌더 패스 정보 구조체를 새로운 색상 어태치먼트로 업데이트합니다:

```c++
    ...
    std::array<VkAttachmentDescription, 3> attachments = {colorAttachment, depthAttachment, colorAttachmentResolve};
    ...
```

렌더 패스가 준비되었으니, `createFramebuffers`를 수정하고 목록에 새로운 이미지 뷰를 추가합니다:

```c++
void createFramebuffers() {
        ...
        std::array<VkImageView, 3> attachments = {
            colorImageView,
            depthImageView,
            swapChainImageViews[i]
        };
        ...
}
```

마지막으로, `createGraphicsPipeline`을 수정하여 새로 생성된 파이프라인이 하나 이상의 샘플을 사용하도록 지시합니다:

```c++
void createGraphicsPipeline() {
    ...
    multisampling.rasterizationSamples = msaaSamples;
    ...
}
```

이제 프로그램을 실행하면 다음과 같은 화면을 볼 수 있습니다:

![](/images/multisampling.png)

밉매핑과 마찬가지로, 차이가 바로 눈에 띄지 않을 수 있습니다. 자세히 보면 가장자리가 더 이상 들쭉날쭉하지 않고, 전체 이미지가 원본에 비해 약간 더 부드러워진 것을 알 수 있습니다.

![](/images/multisampling_comparison.png)

가장자리 중 하나를 가까이에서 보면 차이가 더 두드러집니다:

![](/images/multisampling_comparison2.png)

## 품질 개선

현재 우리의 MSAA 구현에는 몇 가지 한계가 있어 더 디테일한 장면에서 출력 이미지의 품질에 영향을 미칠 수 있습니다. 예를 들어, 우리는 현재 셰이더 에일리어싱으로 인해 발생할 수 있는 잠재적인 문제를 해결하고 있지 않습니다. 즉, MSAA는 지오메트리의 가장자리만 부드럽게 할 뿐 내부 채우기는 처리하지 않습니다. 이로 인해 화면에 부드러운 폴리곤이 렌더링되더라도, 적용된 텍스처에 대비가 강한 색상이 포함되어 있다면 여전히 에일리어싱이 있는 것처럼 보일 수 있습니다. 이 문제에 접근하는 한 가지 방법은 [샘플 셰이딩(Sample Shading)](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap27.html#primsrast-sampleshading)을 활성화하는 것입니다. 이는 추가적인 성능 비용이 들지만 이미지 품질을 더욱 향상시킬 수 있습니다:

```c++

void createLogicalDevice() {
    ...
    deviceFeatures.sampleRateShading = VK_TRUE; // 디바이스에 샘플 셰이딩 기능 활성화
    ...
}

void createGraphicsPipeline() {
    ...
    multisampling.sampleShadingEnable = VK_TRUE; // 파이프라인에서 샘플 셰이딩 활성화
    multisampling.minSampleShading = .2f; // 샘플 셰이딩을 위한 최소 비율; 1에 가까울수록 부드러워짐
    ...
}
```

이 예제에서는 샘플 셰이딩을 비활성화 상태로 두겠지만, 특정 시나리오에서는 품질 향상이 눈에 띄게 나타날 수 있습니다:

![](/images/sample_shading.png)

## 결론

여기까지 오기까지 많은 노력이 필요했지만, 이제 여러분은 마침내 Vulkan 프로그램을 위한 좋은 기반을 갖추게 되었습니다. 여러분이 지금 가진 Vulkan의 기본 원리에 대한 지식은 다음과 같은 더 많은 기능을 탐색하기에 충분할 것입니다:

*   Push constants
*   인스턴스 렌더링(Instanced rendering)
*   동적 유니폼(Dynamic uniforms)
*   분리된 이미지 및 샘플러 디스크립터(Separate images and sampler descriptors)
*   파이프라인 캐시(Pipeline cache)
*   다중 스레드 커맨드 버퍼 생성
*   다중 서브패스(Multiple subpasses)
*   컴퓨트 셰이더(Compute shaders)

현재 프로그램은 블린-퐁(Blinn-Phong) 조명, 후처리(post-processing) 효과, 그림자 매핑(shadow mapping) 등을 추가하여 여러 방식으로 확장될 수 있습니다. Vulkan의 명시성에도 불구하고 많은 개념이 여전히 동일하게 작동하기 때문에, 다른 API를 위한 튜토리얼을 통해서도 이러한 효과들이 어떻게 작동하는지 배울 수 있을 것입니다.

[C++ 코드](/code/30_multisampling.cpp) /
[정점 셰이더](/code/27_shader_depth.vert) /
[프래그먼트 셰이더](/code/27_shader_depth.frag)