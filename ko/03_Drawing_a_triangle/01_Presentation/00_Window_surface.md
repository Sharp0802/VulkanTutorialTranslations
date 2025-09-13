Vulkan은 플랫폼에 독립적인 API이므로, 자체적으로는 윈도우 시스템과 직접 상호작용할 수 없습니다. Vulkan과 윈도우 시스템을 연결하여 화면에 결과를 표시하려면 WSI(Window System Integration, 창 시스템 통합) 확장(extension)을 사용해야 합니다. 이번 장에서는 그 첫 번째인 `VK_KHR_surface`에 대해 알아보겠습니다. 이 확장은 렌더링된 이미지를 표시하기 위한 추상적인 표면 타입을 나타내는 `VkSurfaceKHR` 객체를 제공합니다. 우리 프로그램의 서피스(surface)는 이미 GLFW로 열어 둔 창을 기반으로 할 것입니다.

`VK_KHR_surface` 확장은 인스턴스 수준 확장이며, `glfwGetRequiredInstanceExtensions`가 반환하는 목록에 포함되어 있으므로 우리는 사실 이미 활성화했습니다. 이 목록에는 다음 몇 개의 장에서 사용할 다른 WSI 확장들도 포함되어 있습니다.

윈도우 서피스는 물리 장치(physical device) 선택에 영향을 줄 수 있으므로 인스턴스 생성 직후에 만들어야 합니다. 이 작업을 미룬 이유는 윈도우 서피스가 렌더 타겟 및 프레젠테이션(presentation)이라는 더 큰 주제의 일부이며, 이를 설명하면 기본 설정 과정이 복잡해졌을 것이기 때문입니다. 또한 윈도우 서피스는 Vulkan에서 전적으로 선택적인 구성 요소라는 점도 알아두어야 합니다. 오프스크린 렌더링만 필요하다면, Vulkan은 (OpenGL에서 필요했던) 보이지 않는 창을 만드는 것과 같은 꼼수 없이도 이를 가능하게 해줍니다.

## 윈도우 서피스 생성

먼저 디버그 콜백 바로 아래에 `surface` 클래스 멤버를 추가합니다.

```c++
VkSurfaceKHR surface;
```

`VkSurfaceKHR` 객체와 그 사용법은 플랫폼에 독립적이지만, 생성 과정은 윈도우 시스템 세부 정보에 의존하기 때문에 그렇지 않습니다. 예를 들어, Windows에서는 `HWND`와 `HMODULE` 핸들이 필요합니다. 따라서 이 확장에는 플랫폼별 추가 기능이 있으며, Windows에서는 `VK_KHR_win32_surface`라고 불립니다. 이 또한 `glfwGetRequiredInstanceExtensions`가 반환하는 목록에 자동으로 포함됩니다.

이 플랫폼별 확장을 사용하여 Windows에서 서피스를 생성하는 방법을 보여드리겠지만, 이 튜토리얼에서 실제로 사용하지는 않을 것입니다. GLFW와 같은 라이브러리를 사용하면서 플랫폼별 코드를 사용하는 것은 이치에 맞지 않기 때문입니다. GLFW에는 우리를 대신해 플랫폼별 차이점을 처리해주는 `glfwCreateWindowSurface` 함수가 있습니다. 그럼에도 불구하고, 우리가 이 함수에 의존하기 전에 내부적으로 어떤 일이 일어나는지 알아두는 것은 좋습니다.

네이티브 플랫폼 함수에 접근하려면, 상단의 include 구문을 업데이트해야 합니다:

```c++
#define VK_USE_PLATFORM_WIN32_KHR
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
#define GLFW_EXPOSE_NATIVE_WIN32
#include <GLFW/glfw3native.h>
```

윈도우 서피스는 Vulkan 객체이므로, 채워야 할 `VkWin32SurfaceCreateInfoKHR` 구조체가 함께 제공됩니다. 여기에는 `hwnd`와 `hinstance`라는 두 가지 중요한 파라미터가 있습니다. 이것들은 각각 창과 프로세스에 대한 핸들입니다.

```c++
VkWin32SurfaceCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
createInfo.hwnd = glfwGetWin32Window(window);
createInfo.hinstance = GetModuleHandle(nullptr);
```

`glfwGetWin32Window` 함수는 GLFW 창 객체로부터 원시(raw) `HWND`를 얻는 데 사용됩니다. `GetModuleHandle` 호출은 현재 프로세스의 `HINSTANCE` 핸들을 반환합니다.

그 후 `vkCreateWin32SurfaceKHR`로 서피스를 생성할 수 있습니다. 이 함수는 인스턴스, 서피스 생성 정보, 사용자 지정 할당자(custom allocator), 그리고 서피스 핸들이 저장될 변수를 파라미터로 받습니다. 기술적으로 이것은 WSI 확장 함수이지만, 매우 흔하게 사용되므로 표준 Vulkan 로더에 포함되어 있습니다. 따라서 다른 확장들과 달리 명시적으로 함수를 로드할 필요가 없습니다.

```c++
if (vkCreateWin32SurfaceKHR(instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```

이 과정은 다른 플랫폼에서도 유사합니다. 예를 들어 Linux에서는 X11과 함께 `vkCreateXcbSurfaceKHR` 함수가 XCB 연결(connection)과 창을 생성 정보로 받습니다.

`glfwCreateWindowSurface` 함수는 각 플랫폼에 대해 다른 구현으로 정확히 이 작업을 수행합니다. 이제 이 함수를 우리 프로그램에 통합해 보겠습니다. `initVulkan` 함수에서 인스턴스 생성과 `setupDebugMessenger` 호출 직후에 호출될 `createSurface` 함수를 추가하세요.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createSurface() {

}
```

GLFW 호출은 구조체 대신 간단한 파라미터들을 받으므로 함수 구현이 매우 간단합니다:

```c++
void createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("failed to create window surface!");
    }
}
```

파라미터는 `VkInstance`, GLFW 창 포인터, 사용자 지정 할당자, 그리고 `VkSurfaceKHR` 변수를 가리키는 포인터입니다. 이 함수는 해당 플랫폼의 네이티브 호출에서 반환된 `VkResult`를 그대로 전달합니다. GLFW는 서피스를 파괴하는 특별한 함수를 제공하지 않지만, 원래의 API를 통해 쉽게 할 수 있습니다:

```c++
void cleanup() {
        ...
        vkDestroySurfaceKHR(instance, surface, nullptr);
        vkDestroyInstance(instance, nullptr);
        ...
    }
```

서피스가 인스턴스보다 먼저 파괴되도록 해야 합니다.

## 프레젠테이션 지원 여부 질의

Vulkan 구현이 윈도우 시스템 통합을 지원하더라도, 시스템의 모든 장치가 이를 지원한다는 의미는 아닙니다. 따라서 `isDeviceSuitable`을 확장하여 장치가 우리가 만든 서피스에 이미지를 표시(present)할 수 있는지 확인해야 합니다. 프레젠테이션은 큐에 특화된 기능이므로, 이 문제는 사실 우리가 만든 서피스로의 프레젠테이션을 지원하는 큐 패밀리(queue family)를 찾는 것입니다.

실제로 그리기(drawing) 명령을 지원하는 큐 패밀리와 프레젠테이션을 지원하는 큐 패밀리가 겹치지 않을 수도 있습니다. 따라서 별개의 프레젠테이션 큐가 있을 수 있다는 점을 고려하여 `QueueFamilyIndices` 구조체를 수정해야 합니다:

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```

다음으로, `findQueueFamilies` 함수를 수정하여 우리의 윈도우 서피스에 프레젠테이션할 수 있는 기능을 가진 큐 패밀리를 찾도록 합니다. 이를 확인하는 함수는 `vkGetPhysicalDeviceSurfaceSupportKHR`이며, 물리 장치, 큐 패밀리 인덱스, 서피스를 파라미터로 받습니다. `VK_QUEUE_GRAPHICS_BIT`를 확인하는 루프 안에 이 함수 호출을 추가합니다:

```c++
VkBool32 presentSupport = false;
vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
```

그런 다음 boolean 값을 확인하고 프레젠테이션 큐 패밀리의 인덱스를 저장하면 됩니다:

```c++
if (presentSupport) {
    indices.presentFamily = i;
}
```

결국 이 둘은 동일한 큐 패밀리가 될 가능성이 매우 높지만, 프로그램 전반에 걸쳐 일관된 접근을 위해 별개의 큐인 것처럼 다룰 것입니다. 그럼에도 불구하고, 성능 향상을 위해 그리기와 프레젠테이션을 동일한 큐에서 지원하는 물리 장치를 명시적으로 선호하는 로직을 추가할 수도 있습니다.

## 프레젠테이션 큐 생성

남은 한 가지는 논리 장치(logical device) 생성 절차를 수정하여 프레젠테이션 큐를 만들고 `VkQueue` 핸들을 가져오는 것입니다. 핸들을 위한 멤버 변수를 추가하세요:

```c++
VkQueue presentQueue;
```

다음으로, 두 큐 패밀리로부터 큐를 생성하기 위해 여러 개의 `VkDeviceQueueCreateInfo` 구조체가 필요합니다. 이를 우아하게 처리하는 방법은 필요한 큐들에 대해 고유한 큐 패밀리들의 집합(set)을 만드는 것입니다:

```c++
#include <set>

...

QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo{};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}
```

그리고 `VkDeviceCreateInfo`가 이 벡터를 가리키도록 수정합니다:

```c++
createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();
```

만약 큐 패밀리들이 같다면, 인덱스를 한 번만 전달하면 됩니다. 마지막으로, 큐 핸들을 가져오는 호출을 추가합니다:

```c++
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```

만약 큐 패밀리가 같다면, 두 핸들은 아마도 같은 값을 가질 것입니다. 다음 장에서는 스왑 체인(swap chain)과 스왑 체인이 어떻게 우리에게 서피스에 이미지를 표시할 능력을 주는지에 대해 알아보겠습니다.

[C++ 코드](/code/05_window_surface.cpp)