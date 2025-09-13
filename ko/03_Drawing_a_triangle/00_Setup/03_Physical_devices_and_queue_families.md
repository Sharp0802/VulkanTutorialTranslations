## 물리 디바이스 선택하기

`VkInstance`를 통해 Vulkan 라이브러리를 초기화한 후에는, 시스템에서 우리가 필요로 하는 기능을 지원하는 그래픽 카드를 찾아 선택해야 합니다. 사실 여러 개의 그래픽 카드를 선택하여 동시에 사용할 수도 있지만, 이 튜토리얼에서는 우리의 요구에 맞는 첫 번째 그래픽 카드만 사용하도록 하겠습니다.

`pickPhysicalDevice` 함수를 추가하고 `initVulkan` 함수에서 이 함수를 호출하도록 하겠습니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
}

void pickPhysicalDevice() {

}
```

최종적으로 선택할 그래픽 카드는 새로운 클래스 멤버로 추가되는 `VkPhysicalDevice` 핸들에 저장될 것입니다. 이 객체는 `VkInstance`가 소멸될 때 암시적으로 소멸되므로, `cleanup` 함수에서 새로 무언가를 할 필요는 없습니다.

```c++
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
```

그래픽 카드 목록을 가져오는 것은 확장을 나열하는 것과 매우 유사하며, 먼저 개수를 쿼리하는 것부터 시작합니다.

```c++
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
```

Vulkan을 지원하는 디바이스가 0개라면 더 이상 진행할 의미가 없습니다.

```c++
if (deviceCount == 0) {
    throw std::runtime_error("failed to find GPUs with Vulkan support!");
}
```

그렇지 않다면, 이제 모든 `VkPhysicalDevice` 핸들을 담을 배열을 할당할 수 있습니다.

```c++
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
```

이제 각각의 디바이스를 평가하고 우리가 수행하려는 작업에 적합한지 확인해야 합니다. 모든 그래픽 카드가 동일하게 만들어지지는 않았기 때문입니다. 이를 위해 새로운 함수를 도입하겠습니다.

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

그리고 이 함수에 추가할 요구 사항을 충족하는 물리 디바이스가 있는지 확인할 것입니다.

```c++
for (const auto& device : devices) {
    if (isDeviceSuitable(device)) {
        physicalDevice = device;
        break;
    }
}

if (physicalDevice == VK_NULL_HANDLE) {
    throw std::runtime_error("failed to find a suitable GPU!");
}
```

다음 섹션에서는 `isDeviceSuitable` 함수에서 확인할 첫 번째 요구 사항을 소개할 것입니다. 이후 챕터에서 더 많은 Vulkan 기능을 사용하게 되면, 더 많은 검사를 포함하도록 이 함수를 확장할 것입니다.

## 기본적인 디바이스 적합성 검사

디바이스의 적합성을 평가하기 위해 몇 가지 세부 정보를 쿼리하는 것부터 시작할 수 있습니다. 이름, 타입, 지원하는 Vulkan 버전과 같은 기본적인 디바이스 속성은 `vkGetPhysicalDeviceProperties`를 사용하여 쿼리할 수 있습니다.

```c++
VkPhysicalDeviceProperties deviceProperties;
vkGetPhysicalDeviceProperties(device, &deviceProperties);
```

텍스처 압축, 64비트 부동소수점, 멀티 뷰포트 렌더링(VR에 유용)과 같은 선택적 기능에 대한 지원 여부는 `vkGetPhysicalDeviceFeatures`를 사용하여 쿼리할 수 있습니다.

```c++
VkPhysicalDeviceFeatures deviceFeatures;
vkGetPhysicalDeviceFeatures(device, &deviceFeatures);
```

디바이스 메모리 및 큐 패밀리에 관해서는 나중에 논의할 더 많은 세부 정보가 있습니다(다음 섹션 참조).

예를 들어, 우리 애플리케이션이 지오메트리 셰이더를 지원하는 외장 그래픽 카드에서만 사용 가능하다고 가정해 봅시다. 그러면 `isDeviceSuitable` 함수는 다음과 같을 것입니다.

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    VkPhysicalDeviceProperties deviceProperties;
    VkPhysicalDeviceFeatures deviceFeatures;
    vkGetPhysicalDeviceProperties(device, &deviceProperties);
    vkGetPhysicalDeviceFeatures(device, &deviceFeatures);

    return deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU &&
           deviceFeatures.geometryShader;
}
```

단순히 디바이스가 적합한지 아닌지를 확인하고 첫 번째 것을 선택하는 대신, 각 디바이스에 점수를 매겨 가장 높은 점수를 받은 디바이스를 선택할 수도 있습니다. 이런 식으로 외장 그래픽 카드에 더 높은 점수를 주어 선호하되, 사용 가능한 유일한 GPU가 내장 GPU일 경우에는 내장 GPU를 선택하도록 할 수 있습니다. 다음과 같이 구현할 수 있습니다.

```c++
#include <map>

...

void pickPhysicalDevice() {
    ...

    // 순서가 있는 맵을 사용하여 점수 오름차순으로 후보를 자동 정렬
    std::multimap<int, VkPhysicalDevice> candidates;

    for (const auto& device : devices) {
        int score = rateDeviceSuitability(device);
        candidates.insert(std::make_pair(score, device));
    }

    // 최고점 후보가 적합한지 확인
    if (candidates.rbegin()->first > 0) {
        physicalDevice = candidates.rbegin()->second;
    } else {
        throw std::runtime_error("failed to find a suitable GPU!");
    }
}

int rateDeviceSuitability(VkPhysicalDevice device) {
    ...

    int score = 0;

    // 외장 GPU는 상당한 성능 이점이 있음
    if (deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) {
        score += 1000;
    }

    // 텍스처의 최대 가능 크기는 그래픽 품질에 영향을 줌
    score += deviceProperties.limits.maxImageDimension2D;

    // 지오메트리 셰이더 없이는 애플리케이션이 작동할 수 없음
    if (!deviceFeatures.geometryShader) {
        return 0;
    }

    return score;
}
```

이 튜토리얼에서 이 모든 것을 구현할 필요는 없지만, 디바이스 선택 프로세스를 어떻게 설계할 수 있는지에 대한 아이디어를 제공하기 위함입니다. 물론 선택지의 이름을 표시하고 사용자가 선택하게 할 수도 있습니다.

우리는 이제 막 시작하는 단계이므로 Vulkan 지원 여부가 유일하게 필요한 것이며, 따라서 어떤 GPU든 상관없이 만족할 것입니다.

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

다음 섹션에서는 확인해야 할 첫 번째 실제 필수 기능에 대해 논의하겠습니다.

## 큐 패밀리

이전에 잠시 언급했듯이, 그리기부터 텍스처 업로드에 이르기까지 Vulkan의 거의 모든 작업은 커맨드를 큐에 제출해야 합니다. 서로 다른 *큐 패밀리*에서 비롯된 다양한 유형의 큐가 있으며, 각 큐 패밀리는 커맨드의 일부만 허용합니다. 예를 들어, 컴퓨트 커맨드 처리만 허용하는 큐 패밀리나 메모리 전송 관련 커맨드만 허용하는 큐 패밀리가 있을 수 있습니다.

우리는 디바이스가 어떤 큐 패밀리를 지원하는지, 그리고 그중 어떤 것이 우리가 사용하고자 하는 커맨드를 지원하는지 확인해야 합니다. 이를 위해 필요한 모든 큐 패밀리를 찾는 새로운 함수 `findQueueFamilies`를 추가하겠습니다.

지금은 그래픽 커맨드를 지원하는 큐만 찾을 것이므로, 함수는 다음과 같을 수 있습니다.

```c++
uint32_t findQueueFamilies(VkPhysicalDevice device) {
    // 그래픽 큐 패밀리를 찾는 로직
}
```

하지만 다음 챕터 중 하나에서 곧바로 또 다른 큐를 찾게 될 것이므로, 이를 대비하여 인덱스들을 구조체로 묶는 것이 좋습니다.

```c++
struct QueueFamilyIndices {
    uint32_t graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // 구조체를 채우기 위해 큐 패밀리 인덱스를 찾는 로직
    return indices;
}
```

그런데 만약 큐 패밀리를 사용할 수 없다면 어떻게 할까요? `findQueueFamilies`에서 예외를 던질 수도 있지만, 이 함수는 디바이스 적합성에 대한 결정을 내리기에 적절한 곳이 아닙니다. 예를 들어, 우리는 전용 전송 큐 패밀리가 있는 디바이스를 *선호*할 수는 있지만 필수로 요구하지는 않을 수 있습니다. 따라서 특정 큐 패밀리를 찾았는지 여부를 나타낼 방법이 필요합니다.

큐 패밀리의 부재를 나타내기 위해 매직 값을 사용하는 것은 실제로 불가능합니다. `uint32_t`의 어떤 값이든 이론적으로 유효한 큐 패밀리 인덱스가 될 수 있기 때문입니다( `0` 포함). 다행히 C++17에서는 값이 존재하는 경우와 그렇지 않은 경우를 구별할 수 있는 데이터 구조를 도입했습니다.

```c++
#include <optional>

...

std::optional<uint32_t> graphicsFamily;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // false

graphicsFamily = 0;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // true
```

`std::optional`은 무언가를 할당하기 전까지는 값을 포함하지 않는 래퍼입니다. 언제든지 `has_value()` 멤버 함수를 호출하여 값이 포함되어 있는지 여부를 쿼리할 수 있습니다. 즉, 로직을 다음과 같이 변경할 수 있습니다.

```c++
#include <optional>

...

struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // 찾을 수 있었던 큐 패밀리에 인덱스 할당
    return indices;
}
```

이제 실제로 `findQueueFamilies`를 구현할 수 있습니다.

```c++
QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;

    ...

    return indices;
}
```

큐 패밀리 목록을 가져오는 과정은 예상과 정확히 같으며 `vkGetPhysicalDeviceQueueFamilyProperties`를 사용합니다.

```c++
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
```

`VkQueueFamilyProperties` 구조체는 지원되는 작업 유형 및 해당 패밀리를 기반으로 생성할 수 있는 큐의 수를 포함하여 큐 패밀리에 대한 몇 가지 세부 정보를 담고 있습니다. 우리는 `VK_QUEUE_GRAPHICS_BIT`를 지원하는 큐 패밀리를 최소 하나 찾아야 합니다.

```c++
int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if (queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) {
        indices.graphicsFamily = i;
    }

    i++;
}
```

이제 이 멋진 큐 패밀리 조회 함수가 있으므로, `isDeviceSuitable` 함수에서 디바이스가 우리가 사용하려는 커맨드를 처리할 수 있는지 확인하는 검사로 사용할 수 있습니다.

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.graphicsFamily.has_value();
}
```

이를 좀 더 편리하게 만들기 위해, 구조체 자체에 일반적인 검사 기능도 추가하겠습니다.

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;

    bool isComplete() {
        return graphicsFamily.has_value();
    }
};

...

bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.isComplete();
}
```

이제 `findQueueFamilies`에서 조기 탈출하는 데에도 이를 사용할 수 있습니다.

```c++
for (const auto& queueFamily : queueFamilies) {
    ...

    if (indices.isComplete()) {
        break;
    }

    i++;
}
```

좋습니다, 올바른 물리 디바이스를 찾는 데 필요한 것은 이것으로 충분합니다! 다음 단계는 물리 디바이스와 상호작용하기 위해 [논리 디바이스를 생성하는 것](!en/Drawing_a_triangle/Setup/Logical_device_and_queues)입니다.

[C++ 코드](/code/03_physical_device_selection.cpp)