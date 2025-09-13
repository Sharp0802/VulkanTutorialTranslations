Vulkan은 "기본 프레임버퍼"라는 개념이 없으므로, 화면에 시각화하기 전에 렌더링할 버퍼를 소유하는 기반 구조가 필요합니다. 이 기반 구조는 *스왑 체인(swap chain)*으로 알려져 있으며 Vulkan에서는 명시적으로 생성해야 합니다. 스왑 체인은 본질적으로 화면에 표시되기를 기다리는 이미지들의 큐입니다. 우리 애플리케이션은 이러한 이미지를 얻어서 그리고, 그런 다음 큐로 반환합니다. 큐가 정확히 어떻게 작동하고 큐에서 이미지를 표시하기 위한 조건은 스왑 체인이 어떻게 설정되었는지에 따라 다르지만, 스왑 체인의 일반적인 목적은 이미지 표시를 화면의 주사율(refresh rate)과 동기화하는 것입니다.

## 스왑 체인 지원 여부 확인하기

모든 그래픽 카드가 이미지를 화면에 직접 표시할 수 있는 것은 아닙니다. 예를 들어 서버용으로 설계되어 디스플레이 출력이 없는 경우 등 다양한 이유가 있습니다. 둘째로, 이미지 표시는 윈도우 시스템 및 윈도우와 연관된 서피스(surface)와 밀접하게 관련되어 있으므로 실제로는 Vulkan 코어의 일부가 아닙니다. `VK_KHR_swapchain` 장치 확장의 지원 여부를 쿼리한 후 활성화해야 합니다.

이를 위해 먼저 `isDeviceSuitable` 함수를 확장하여 이 확장이 지원되는지 확인하겠습니다. 이전에 `VkPhysicalDevice`에서 지원하는 확장 목록을 나열하는 방법을 보았으므로, 이는 꽤 간단할 것입니다. Vulkan 헤더 파일은 `VK_KHR_swapchain`으로 정의된 멋진 매크로 `VK_KHR_SWAPCHAIN_EXTENSION_NAME`을 제공합니다. 이 매크로를 사용하면 컴파일러가 오타를 잡아낼 수 있다는 장점이 있습니다.

먼저, 활성화할 유효성 검사 레이어 목록과 유사하게, 필요한 장치 확장 목록을 선언합니다.

```c++
const std::vector<const char*> deviceExtensions = {
    VK_KHR_SWAPCHAIN_EXTENSION_NAME
};
```

다음으로, `isDeviceSuitable`에서 추가적인 확인 절차로 호출되는 새로운 함수 `checkDeviceExtensionSupport`를 생성합니다.

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    bool extensionsSupported = checkDeviceExtensionSupport(device);

    return indices.isComplete() && extensionsSupported;
}

bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    return true;
}
```

함수 본문을 수정하여 확장들을 열거하고 필요한 모든 확장이 그 중에 있는지 확인합니다.

```c++
bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    uint32_t extensionCount;
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);

    std::vector<VkExtensionProperties> availableExtensions(extensionCount);
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());

    std::set<std::string> requiredExtensions(deviceExtensions.begin(), deviceExtensions.end());

    for (const auto& extension : availableExtensions) {
        requiredExtensions.erase(extension.extensionName);
    }

    return requiredExtensions.empty();
}
```

저는 여기서 아직 확인되지 않은 필수 확장을 표현하기 위해 문자열 집합(set of strings)을 사용하기로 했습니다. 이렇게 하면 사용 가능한 확장 시퀀스를 열거하면서 쉽게 하나씩 지워나갈 수 있습니다. 물론 `checkValidationLayerSupport`에서처럼 중첩 루프를 사용할 수도 있습니다. 성능 차이는 무시할 만합니다. 이제 코드를 실행하여 그래픽 카드가 실제로 스왑 체인을 생성할 수 있는지 확인하세요. 이전 장에서 확인했던 프레젠테이션 큐의 사용 가능성은 스왑 체인 확장이 지원되어야 함을 의미한다는 점에 유의해야 합니다. 하지만, 여전히 명시적으로 하는 것이 좋으며, 확장은 명시적으로 활성화해야 합니다.

## 장치 확장 활성화하기

스왑 체인을 사용하려면 먼저 `VK_KHR_swapchain` 확장을 활성화해야 합니다. 확장을 활성화하는 것은 논리 장치 생성 구조체를 약간 변경하기만 하면 됩니다.

```c++
createInfo.enabledExtensionCount = static_cast<uint32_t>(deviceExtensions.size());
createInfo.ppEnabledExtensionNames = deviceExtensions.data();
```

이 작업을 할 때 기존의 `createInfo.enabledExtensionCount = 0;` 줄을 교체해야 합니다.

## 스왑 체인 지원 상세 정보 쿼리하기

스왑 체인이 사용 가능한지 확인하는 것만으로는 충분하지 않습니다. 실제로는 우리 윈도우 서피스와 호환되지 않을 수 있기 때문입니다. 스왑 체인을 생성하는 것은 인스턴스 및 장치 생성보다 훨씬 더 많은 설정을 포함하므로, 계속 진행하기 전에 몇 가지 세부 사항을 더 쿼리해야 합니다.

기본적으로 확인해야 할 세 가지 종류의 속성이 있습니다.

*   기본 서피스 기능 (스왑 체인의 최소/최대 이미지 수, 이미지의 최소/최대 너비 및 높이)
*   서피스 포맷 (픽셀 포맷, 색 공간)
*   사용 가능한 프레젠테이션 모드

`findQueueFamilies`와 유사하게, 이 상세 정보들을 쿼리한 후 전달하기 위해 구조체를 사용할 것입니다. 앞서 언급한 세 가지 종류의 속성은 다음 구조체 및 구조체 목록의 형태로 제공됩니다.

```c++
struct SwapChainSupportDetails {
    VkSurfaceCapabilitiesKHR capabilities;
    std::vector<VkSurfaceFormatKHR> formats;
    std::vector<VkPresentModeKHR> presentModes;
};
```

이제 이 구조체를 채울 새로운 함수 `querySwapChainSupport`를 만들 것입니다.

```c++
SwapChainSupportDetails querySwapChainSupport(VkPhysicalDevice device) {
    SwapChainSupportDetails details;

    return details;
}
```

이 섹션에서는 이 정보를 포함하는 구조체들을 쿼리하는 방법을 다룹니다. 이 구조체들의 의미와 정확히 어떤 데이터를 포함하는지는 다음 섹션에서 논의합니다.

먼저 기본 서피스 기능부터 시작하겠습니다. 이러한 속성은 쿼리하기 간단하며 단일 `VkSurfaceCapabilitiesKHR` 구조체로 반환됩니다.

```c++
vkGetPhysicalDeviceSurfaceCapabilitiesKHR(device, surface, &details.capabilities);
```

이 함수는 지원되는 기능을 결정할 때 지정된 `VkPhysicalDevice`와 `VkSurfaceKHR` 윈도우 서피스를 고려합니다. 모든 지원 쿼리 함수는 이 두 가지를 첫 번째 매개변수로 가집니다. 왜냐하면 이것들이 스왑 체인의 핵심 구성 요소이기 때문입니다.

다음 단계는 지원되는 서피스 포맷을 쿼리하는 것입니다. 이는 구조체의 목록이므로, 익숙한 방식인 2번의 함수 호출을 따릅니다.

```c++
uint32_t formatCount;
vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, nullptr);

if (formatCount != 0) {
    details.formats.resize(formatCount);
    vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, details.formats.data());
}
```

벡터가 사용 가능한 모든 포맷을 담을 수 있도록 크기가 조절되었는지 확인하세요. 그리고 마지막으로, 지원되는 프레젠테이션 모드를 쿼리하는 것도 `vkGetPhysicalDeviceSurfacePresentModesKHR`를 사용하여 정확히 동일한 방식으로 작동합니다.

```c++
uint32_t presentModeCount;
vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, nullptr);

if (presentModeCount != 0) {
    details.presentModes.resize(presentModeCount);
    vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, details.presentModes.data());
}
```

이제 모든 상세 정보가 구조체에 담겼으니, `isDeviceSuitable`을 한 번 더 확장하여 이 함수를 활용해 스왑 체인 지원이 적절한지 확인합시다. 이 튜토리얼에서는 우리가 가진 윈도우 서피스에 대해 하나 이상의 지원되는 이미지 포맷과 하나 이상의 지원되는 프레젠테이션 모드가 있다면 스왑 체인 지원이 충분하다고 봅니다.

```c++
bool swapChainAdequate = false;
if (extensionsSupported) {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
    swapChainAdequate = !swapChainSupport.formats.empty() && !swapChainSupport.presentModes.empty();
}
```

확장이 사용 가능한지 확인한 후에만 스왑 체인 지원을 쿼리하려고 시도하는 것이 중요합니다. 함수의 마지막 줄은 다음과 같이 변경됩니다.

```c++
return indices.isComplete() && extensionsSupported && swapChainAdequate;
```

## 스왑 체인에 맞는 올바른 설정 선택하기

`swapChainAdequate` 조건이 충족되었다면 지원은 확실히 충분하지만, 최적성이 다양한 여러 가지 모드가 있을 수 있습니다. 이제 가능한 최상의 스왑 체인을 위한 올바른 설정을 찾는 몇 가지 함수를 작성하겠습니다. 결정해야 할 설정에는 세 가지 유형이 있습니다.

*   서피스 포맷 (색상 깊이)
*   프레젠테이션 모드 (이미지를 화면으로 '스왑'하기 위한 조건)
*   스왑 익스텐트 (스왑 체인 이미지의 해상도)

각 설정에 대해, 우리는 염두에 둔 이상적인 값을 가질 것이며, 그것이 사용 가능하다면 그 값을 선택하고, 그렇지 않다면 차선책을 찾는 로직을 만들 것입니다.

### 서피스 포맷

이 설정을 위한 함수는 이렇게 시작합니다. 나중에 `SwapChainSupportDetails` 구조체의 `formats` 멤버를 인자로 전달할 것입니다.

```c++
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {

}
```

각 `VkSurfaceFormatKHR` 항목은 `format` 멤버와 `colorSpace` 멤버를 포함합니다. `format` 멤버는 색상 채널과 타입을 지정합니다. 예를 들어, `VK_FORMAT_B8G8R8A8_SRGB`는 B, G, R, 알파 채널을 그 순서대로 8비트 부호 없는 정수로 저장하여 픽셀당 총 32비트를 의미합니다. `colorSpace` 멤버는 `VK_COLOR_SPACE_SRGB_NONLINEAR_KHR` 플래그를 사용하여 SRGB 색 공간이 지원되는지 여부를 나타냅니다. 이 플래그는 이전 버전의 명세에서는 `VK_COLORSPACE_SRGB_NONLINEAR_KHR`로 불렸습니다.

색 공간으로는 SRGB가 사용 가능하다면 그것을 사용할 것입니다. 왜냐하면 [더 정확하게 인지되는 색상을 결과로 내기 때문](http://stackoverflow.com/questions/12524623/)입니다. 또한 나중에 사용할 텍스처와 같은 이미지를 위한 표준 색 공간이기도 합니다. 그 때문에 우리는 SRGB 색상 포맷을 사용해야 하며, 가장 일반적인 것 중 하나는 `VK_FORMAT_B8G8R8A8_SRGB`입니다.

목록을 살펴보고 선호하는 조합이 사용 가능한지 확인해 봅시다.

```c++
for (const auto& availableFormat : availableFormats) {
    if (availableFormat.format == VK_FORMAT_B8G8R8A8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
        return availableFormat;
    }
}
```

만약 그것도 실패한다면, 사용 가능한 포맷들이 얼마나 "좋은지"에 따라 순위를 매길 수도 있지만, 대부분의 경우 지정된 첫 번째 포맷으로 만족해도 괜찮습니다.

```c++
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {
    for (const auto& availableFormat : availableFormats) {
        if (availableFormat.format == VK_FORMAT_B8G8R8A8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
            return availableFormat;
        }
    }

    return availableFormats[0];
}
```

### 프레젠테이션 모드

프레젠테이션 모드는 스왑 체인에 있어 틀림없이 가장 중요한 설정입니다. 왜냐하면 이미지를 화면에 보여주기 위한 실제 조건을 나타내기 때문입니다. Vulkan에는 네 가지 가능한 모드가 있습니다.

*   `VK_PRESENT_MODE_IMMEDIATE_KHR`: 애플리케이션이 제출한 이미지가 즉시 화면으로 전송되어 티어링(tearing)이 발생할 수 있습니다.
*   `VK_PRESENT_MODE_FIFO_KHR`: 스왑 체인은 큐이며, 디스플레이가 새로 고쳐질 때 큐의 앞에서 이미지를 가져가고 프로그램은 렌더링된 이미지를 큐의 뒤에 삽입합니다. 큐가 가득 차면 프로그램은 기다려야 합니다. 이는 현대 게임에서 볼 수 있는 수직 동기화(vertical sync)와 가장 유사합니다. 디스플레이가 새로 고쳐지는 순간을 "수직 귀선 기간(vertical blank)"이라고 합니다.
*   `VK_PRESENT_MODE_FIFO_RELAXED_KHR`: 이 모드는 애플리케이션이 늦어서 마지막 수직 귀선 기간에 큐가 비어 있었을 경우에만 이전 모드와 다릅니다. 다음 수직 귀선 기간을 기다리는 대신, 이미지가 마침내 도착하면 즉시 전송됩니다. 이로 인해 눈에 보이는 티어링이 발생할 수 있습니다.
*   `VK_PRESENT_MODE_MAILBOX_KHR`: 이것은 두 번째 모드의 또 다른 변형입니다. 큐가 가득 찼을 때 애플리케이션을 차단하는 대신, 이미 큐에 있는 이미지들이 더 새로운 이미지들로 교체됩니다. 이 모드는 티어링을 피하면서도 가능한 한 빨리 프레임을 렌더링하는 데 사용될 수 있으며, 표준 수직 동기화보다 더 적은 지연 시간 문제를 초래합니다. 이것은 흔히 "삼중 버퍼링(triple buffering)"으로 알려져 있지만, 세 개의 버퍼가 존재한다고 해서 반드시 프레임레이트가 제한 해제되었다는 의미는 아닙니다.

오직 `VK_PRESENT_MODE_FIFO_KHR` 모드만이 사용 가능함이 보장되므로, 우리는 다시 사용 가능한 최상의 모드를 찾는 함수를 작성해야 합니다.

```c++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes) {
    return VK_PRESENT_MODE_FIFO_KHR;
}
```

개인적으로 저는 에너지 사용량이 걱정되지 않는다면 `VK_PRESENT_MODE_MAILBOX_KHR`가 아주 좋은 절충안이라고 생각합니다. 이 모드는 수직 귀선 기간 직전까지 가능한 한 최신 상태의 새 이미지를 렌더링하여 티어링을 피하면서도 상당히 낮은 지연 시간을 유지할 수 있게 해줍니다. 에너지 사용량이 더 중요한 모바일 기기에서는 아마도 `VK_PRESENT_MODE_FIFO_KHR`를 사용하고 싶을 것입니다. 이제 목록을 살펴 `VK_PRESENT_MODE_MAILBOX_KHR`가 사용 가능한지 확인해 봅시다.

```c++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes) {
    for (const auto& availablePresentMode : availablePresentModes) {
        if (availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR) {
            return availablePresentMode;
        }
    }

    return VK_PRESENT_MODE_FIFO_KHR;
}
```

### 스왑 익스텐트

이제 마지막 주요 속성 하나가 남았으며, 이를 위해 마지막 함수를 하나 더 추가할 것입니다.

```c++
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {

}
```

스왑 익스텐트(swap extent)는 스왑 체인 이미지의 해상도이며, 거의 항상 우리가 그리는 윈도우의 해상도와 _픽셀 단위로_ 정확히 같습니다(이 점에 대해서는 잠시 후에 더 설명하겠습니다). 가능한 해상도의 범위는 `VkSurfaceCapabilitiesKHR` 구조체에 정의되어 있습니다. Vulkan은 `currentExtent` 멤버의 너비와 높이를 설정하여 윈도우의 해상도에 맞추라고 알려줍니다. 그러나 일부 윈도우 관리자는 여기서 다르게 설정하는 것을 허용하며, 이는 `currentExtent`의 너비와 높이를 특별한 값, 즉 `uint32_t`의 최댓값으로 설정하여 표시됩니다. 이 경우, 우리는 `minImageExtent`와 `maxImageExtent` 범위 내에서 윈도우에 가장 잘 맞는 해상도를 선택할 것입니다. 하지만 해상도를 올바른 단위로 지정해야 합니다.

GLFW는 크기를 측정할 때 픽셀과 [화면 좌표](https://www.glfw.org/docs/latest/intro_guide.html#coordinate_systems)라는 두 가지 단위를 사용합니다. 예를 들어, 이전에 윈도우를 생성할 때 지정했던 `{WIDTH, HEIGHT}` 해상도는 화면 좌표로 측정됩니다. 하지만 Vulkan은 픽셀 단위로 작동하므로, 스왑 체인 익스텐트도 픽셀 단위로 지정해야 합니다. 불행히도, 고DPI 디스플레이(Apple의 Retina 디스플레이처럼)를 사용하는 경우 화면 좌표는 픽셀과 일치하지 않습니다. 대신, 더 높은 픽셀 밀도 때문에 픽셀 단위의 윈도우 해상도는 화면 좌표 단위의 해상도보다 더 커집니다. 따라서 Vulkan이 스왑 익스텐트를 고정해주지 않는다면, 우리는 원래의 `{WIDTH, HEIGHT}`를 그냥 사용할 수 없습니다. 대신, 최소 및 최대 이미지 익스텐트와 비교하기 전에 `glfwGetFramebufferSize`를 사용하여 픽셀 단위의 윈도우 해상도를 쿼리해야 합니다.

```c++
#include <cstdint> // uint32_t에 필요
#include <limits> // std::numeric_limits에 필요
#include <algorithm> // std::clamp에 필요

...

VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != std::numeric_limits<uint32_t>::max()) {
        return capabilities.currentExtent;
    } else {
        int width, height;
        glfwGetFramebufferSize(window, &width, &height);

        VkExtent2D actualExtent = {
            static_cast<uint32_t>(width),
            static_cast<uint32_t>(height)
        };

        actualExtent.width = std::clamp(actualExtent.width, capabilities.minImageExtent.width, capabilities.maxImageExtent.width);
        actualExtent.height = std::clamp(actualExtent.height, capabilities.minImageExtent.height, capabilities.maxImageExtent.height);

        return actualExtent;
    }
}
```

여기서 `clamp` 함수는 `width`와 `height`의 값을 구현체가 지원하는 허용된 최소 및 최대 익스텐트 사이로 제한하는 데 사용됩니다.

## 스왑 체인 생성하기

이제 런타임에 내려야 할 선택을 도와주는 모든 헬퍼 함수들을 갖게 되었으므로, 마침내 작동하는 스왑 체인을 생성하는 데 필요한 모든 정보를 갖게 되었습니다.

`createSwapChain` 함수를 만들고, 이 함수가 이러한 호출들의 결과로 시작하도록 하세요. 그리고 `initVulkan`에서 논리 장치 생성 후에 이 함수를 호출해야 합니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
}

void createSwapChain() {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(physicalDevice);

    VkSurfaceFormatKHR surfaceFormat = chooseSwapSurfaceFormat(swapChainSupport.formats);
    VkPresentModeKHR presentMode = chooseSwapPresentMode(swapChainSupport.presentModes);
    VkExtent2D extent = chooseSwapExtent(swapChainSupport.capabilities);
}
```

이러한 속성 외에도, 우리는 스왑 체인에 얼마나 많은 이미지를 가질 것인지 결정해야 합니다. 구현체는 작동하는 데 필요한 최소 개수를 명시합니다.

```c++
uint32_t imageCount = swapChainSupport.capabilities.minImageCount;
```

하지만, 단순히 이 최소값에 머무르는 것은 렌더링할 다른 이미지를 얻기 전에 드라이버가 내부 작업을 완료하기를 기다려야 할 수도 있음을 의미합니다. 따라서 최소값보다 적어도 하나 더 많은 이미지를 요청하는 것이 권장됩니다.

```c++
uint32_t imageCount = swapChainSupport.capabilities.minImageCount + 1;
```

또한 이 작업을 하는 동안 이미지의 최대 개수를 초과하지 않도록 해야 합니다. 여기서 `0`은 최댓값이 없다는 것을 의미하는 특별한 값입니다.

```c++
if (swapChainSupport.capabilities.maxImageCount > 0 && imageCount > swapChainSupport.capabilities.maxImageCount) {
    imageCount = swapChainSupport.capabilities.maxImageCount;
}
```

Vulkan 객체의 전통처럼, 스왑 체인 객체를 생성하려면 큰 구조체를 채워야 합니다. 시작은 매우 익숙합니다.

```c++
VkSwapchainCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
createInfo.surface = surface;
```

스왑 체인이 어떤 서피스에 연결되어야 하는지를 지정한 후, 스왑 체인 이미지의 상세 정보가 지정됩니다.

```c++
createInfo.minImageCount = imageCount;
createInfo.imageFormat = surfaceFormat.format;
createInfo.imageColorSpace = surfaceFormat.colorSpace;
createInfo.imageExtent = extent;
createInfo.imageArrayLayers = 1;
createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;
```

`imageArrayLayers`는 각 이미지가 구성되는 레이어의 양을 지정합니다. 입체 3D 애플리케이션을 개발하지 않는 한 항상 `1`입니다. `imageUsage` 비트 필드는 스왑 체인의 이미지를 어떤 종류의 작업에 사용할지 지정합니다. 이 튜토리얼에서는 이미지에 직접 렌더링할 것이므로, 컬러 어태치먼트로 사용됩니다. 후처리(post-processing)와 같은 작업을 수행하기 위해 먼저 별도의 이미지에 렌더링할 수도 있습니다. 그 경우 `VK_IMAGE_USAGE_TRANSFER_DST_BIT`와 같은 값을 사용하고 메모리 연산을 사용하여 렌더링된 이미지를 스왑 체인 이미지로 전송할 수 있습니다.

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
uint32_t queueFamilyIndices[] = {indices.graphicsFamily.value(), indices.presentFamily.value()};

if (indices.graphicsFamily != indices.presentFamily) {
    createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
    createInfo.queueFamilyIndexCount = 2;
    createInfo.pQueueFamilyIndices = queueFamilyIndices;
} else {
    createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
    createInfo.queueFamilyIndexCount = 0; // Optional
    createInfo.pQueueFamilyIndices = nullptr; // Optional
}
```

다음으로, 여러 큐 패밀리에 걸쳐 사용될 스왑 체인 이미지를 처리하는 방법을 지정해야 합니다. 우리 애플리케이션에서는 그래픽 큐 패밀리가 프레젠테이션 큐와 다른 경우에 해당됩니다. 우리는 그래픽 큐에서 스왑 체인의 이미지에 그리고, 그 다음 프레젠테이션 큐에 제출할 것입니다. 여러 큐에서 접근하는 이미지를 처리하는 두 가지 방법이 있습니다.

*   `VK_SHARING_MODE_EXCLUSIVE`: 이미지는 한 번에 하나의 큐 패밀리에 의해 소유되며, 다른 큐 패밀리에서 사용하기 전에 소유권은 명시적으로 이전되어야 합니다. 이 옵션은 최고의 성능을 제공합니다.
*   `VK_SHARING_MODE_CONCURRENT`: 이미지는 명시적인 소유권 이전 없이 여러 큐 패밀리에서 사용될 수 있습니다.

큐 패밀리가 다른 경우, 이 튜토리얼에서는 동시 모드를 사용할 것입니다. 이는 나중에 설명하는 것이 더 좋은 개념들을 포함하는 소유권 관련 챕터를 다루지 않기 위함입니다. 동시 모드는 `queueFamilyIndexCount`와 `pQueueFamilyIndices` 매개변수를 사용하여 어느 큐 패밀리 간에 소유권이 공유될지 미리 지정해야 합니다. 대부분의 하드웨어에서처럼 그래픽 큐 패밀리와 프레젠테이션 큐 패밀리가 같다면, 배타적 모드를 고수해야 합니다. 동시 모드는 최소 두 개의 서로 다른 큐 패밀리를 지정해야 하기 때문입니다.

```c++
createInfo.preTransform = swapChainSupport.capabilities.currentTransform;
```

`capabilities`의 `supportedTransforms`에서 지원된다면, 시계 방향 90도 회전이나 수평 뒤집기 같은 특정 변환을 스왑 체인의 이미지에 적용하도록 지정할 수 있습니다. 변환을 원하지 않는다고 지정하려면, 단순히 현재 변환을 지정하세요.

```c++
createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
```

`compositeAlpha` 필드는 알파 채널을 윈도우 시스템의 다른 윈도우와 블렌딩하는 데 사용할지 여부를 지정합니다. 거의 항상 알파 채널을 단순히 무시하고 싶을 것이므로, `VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR`를 사용합니다.

```c++
createInfo.presentMode = presentMode;
createInfo.clipped = VK_TRUE;
```

`presentMode` 멤버는 그 자체로 명확합니다. `clipped` 멤버가 `VK_TRUE`로 설정되면, 다른 윈도우가 그 앞에 있는 경우처럼 가려진 픽셀의 색상에는 신경 쓰지 않는다는 의미입니다. 이 픽셀들을 다시 읽어서 예측 가능한 결과를 얻어야 하는 경우가 아니라면, 클리핑을 활성화하여 최고의 성능을 얻을 수 있습니다.

```c++
createInfo.oldSwapchain = VK_NULL_HANDLE;
```

이제 마지막 필드인 `oldSwapchain`이 남았습니다. Vulkan에서는 애플리케이션이 실행되는 동안 윈도우 크기가 조절된 경우처럼 스왑 체인이 유효하지 않게 되거나 최적화되지 않은 상태가 될 수 있습니다. 이 경우 스왑 체인을 처음부터 다시 생성해야 하며, 이전 스왑 체인에 대한 참조가 이 필드에 지정되어야 합니다. 이것은 [추후 챕터](!en/Drawing_a_triangle/Swap_chain_recreation)에서 더 배울 복잡한 주제입니다. 지금은 단 하나의 스왑 체인만 생성한다고 가정할 것입니다.

이제 `VkSwapchainKHR` 객체를 저장할 클래스 멤버를 추가합니다.

```c++
VkSwapchainKHR swapChain;
```

이제 스왑 체인을 생성하는 것은 `vkCreateSwapchainKHR`를 호출하는 것만큼 간단합니다.

```c++
if (vkCreateSwapchainKHR(device, &createInfo, nullptr, &swapChain) != VK_SUCCESS) {
    throw std::runtime_error("failed to create swap chain!");
}
```

매개변수는 논리 장치, 스왑 체인 생성 정보, 선택적 커스텀 할당자, 그리고 핸들을 저장할 변수에 대한 포인터입니다. 놀랄 것은 없습니다. 이는 장치보다 먼저 `vkDestroySwapchainKHR`를 사용하여 정리해야 합니다.

```c++
void cleanup() {
    vkDestroySwapchainKHR(device, swapChain, nullptr);
    ...
}
```

이제 애플리케이션을 실행하여 스왑 체인이 성공적으로 생성되는지 확인하세요! 이 시점에서 `vkCreateSwapchainKHR`에서 접근 위반 오류가 발생하거나 `Failed to find 'vkGetInstanceProcAddress' in layer SteamOverlayVulkanLayer.dll`와 같은 메시지가 보이면, Steam 오버레이 레이어에 대한 [FAQ 항목](!en/FAQ)을 참조하세요.

유효성 검사 레이어가 활성화된 상태에서 `createInfo.imageExtent = extent;` 줄을 제거해 보세요. 유효성 검사 레이어 중 하나가 즉시 실수를 잡아내고 유용한 메시지가 출력되는 것을 볼 수 있습니다.

![](/images/swap_chain_validation_layer.png)

## 스왑 체인 이미지 가져오기

이제 스왑 체인이 생성되었으므로, 남은 것은 그 안의 `VkImage` 핸들을 가져오는 것뿐입니다. 우리는 이후 챕터의 렌더링 작업 중에 이것들을 참조할 것입니다. 핸들을 저장할 클래스 멤버를 추가하세요.

```c++
std::vector<VkImage> swapChainImages;
```

이미지들은 스왑 체인을 위해 구현체에 의해 생성되었으며, 스왑 체인이 파괴되면 자동으로 정리될 것이므로, 우리는 어떤 정리 코드도 추가할 필요가 없습니다.

저는 핸들을 가져오는 코드를 `createSwapChain` 함수의 끝, `vkCreateSwapchainKHR` 호출 바로 뒤에 추가하고 있습니다. 핸들을 가져오는 것은 Vulkan에서 객체 배열을 가져왔던 다른 경우들과 매우 유사합니다. 우리는 스왑 체인에 최소 이미지 수만 지정했으므로, 구현체는 더 많은 이미지를 가진 스왑 체인을 생성할 수 있다는 점을 기억하세요. 그래서 우리는 먼저 `vkGetSwapchainImagesKHR`로 최종 이미지 개수를 쿼리한 다음, 컨테이너의 크기를 조절하고, 마지막으로 핸들을 가져오기 위해 다시 호출할 것입니다.

```c++
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, nullptr);
swapChainImages.resize(imageCount);
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, swapChainImages.data());
```

마지막으로 한 가지, 스왑 체인 이미지에 대해 선택한 포맷과 익스텐트를 멤버 변수에 저장하세요. 이후 챕터에서 필요할 것입니다.

```c++
VkSwapchainKHR swapChain;
std::vector<VkImage> swapChainImages;
VkFormat swapChainImageFormat;
VkExtent2D swapChainExtent;

...

swapChainImageFormat = surfaceFormat.format;
swapChainExtent = extent;
```

이제 우리는 그릴 수 있고 윈도우에 표시될 수 있는 이미지 집합을 갖게 되었습니다. 다음 챕터에서는 이미지를 렌더 타겟으로 설정하는 방법을 다루기 시작하고, 그 다음에는 실제 그래픽 파이프라인과 그리기 커맨드에 대해 알아볼 것입니다!

[C++ 코드](/code/06_swap_chain_creation.cpp)