## 소개

사용할 물리 장치를 선택한 후에는, 그것과 인터페이스할 *논리 장치*(logical device)를 설정해야 합니다. 논리 장치 생성 과정은 인스턴스 생성 과정과 유사하며 우리가 사용하고자 하는 기능들을 기술합니다. 또한 어떤 큐 패밀리가 사용 가능한지 질의(query)했으므로 이제 어떤 큐를 생성할지 명시해야 합니다. 요구 사항이 다양하다면 같은 물리 장치에서 여러 개의 논리 장치를 생성할 수도 있습니다.

먼저 논리 장치 핸들을 저장할 새로운 클래스 멤버를 추가하는 것으로 시작하겠습니다.

```c++
VkDevice device;
```

다음으로, `initVulkan`에서 호출되는 `createLogicalDevice` 함수를 추가합니다.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createLogicalDevice() {

}
```

## 생성할 큐 명시하기

논리 장치를 생성하려면 또다시 구조체에 여러 세부 정보를 명시해야 하는데, 그 첫 번째는 `VkDeviceQueueCreateInfo`입니다. 이 구조체는 단일 큐 패밀리에 대해 우리가 원하는 큐의 개수를 기술합니다. 지금은 그래픽스 기능이 있는 큐에만 관심이 있습니다.

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

VkDeviceQueueCreateInfo queueCreateInfo{};
queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
queueCreateInfo.queueCount = 1;
```

현재 사용 가능한 드라이버들은 각 큐 패밀리마다 적은 수의 큐만 생성하도록 허용하며, 사실 한 개보다 더 많이 필요하지는 않습니다. 왜냐하면 모든 커맨드 버퍼를 여러 스레드에서 생성한 다음, 메인 스레드에서 오버헤드가 적은 단일 호출로 한 번에 제출할 수 있기 때문입니다.

Vulkan에서는 `0.0`과 `1.0` 사이의 부동 소수점 숫자를 사용하여 큐에 우선순위를 할당함으로써 커맨드 버퍼 실행 스케줄링에 영향을 줄 수 있습니다. 이는 큐가 단 하나만 있는 경우에도 필수적입니다.

```c++
float queuePriority = 1.0f;
queueCreateInfo.pQueuePriorities = &queuePriority;
```

## 사용할 장치 기능 명시하기

다음으로 명시할 정보는 우리가 사용할 장치 기능들의 집합입니다. 이것들은 이전 장에서 `vkGetPhysicalDeviceFeatures`로 지원 여부를 질의했던 기능들로, 지오메트리 셰이더 같은 것들입니다. 지금은 특별한 기능이 필요 없으므로, 그냥 정의한 뒤 모든 것을 `VK_FALSE`로 두면 됩니다. 이 구조체는 나중에 Vulkan으로 더 흥미로운 작업을 시작할 때 다시 다루겠습니다.

```c++
VkPhysicalDeviceFeatures deviceFeatures{};
```

## 논리 장치 생성하기

앞의 두 구조체가 준비되었으니, 이제 메인 `VkDeviceCreateInfo` 구조체를 채워 넣기 시작할 수 있습니다.

```c++
VkDeviceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
```

먼저 큐 생성 정보와 장치 기능 구조체를 가리키는 포인터를 추가합니다.

```c++
createInfo.pQueueCreateInfos = &queueCreateInfo;
createInfo.queueCreateInfoCount = 1;

createInfo.pEnabledFeatures = &deviceFeatures;
```

나머지 정보는 `VkInstanceCreateInfo` 구조체와 유사하며, 확장 기능과 유효성 검사 레이어를 명시해야 합니다. 차이점은 이번에는 이것들이 장치에 한정된다는 점입니다.

장치 한정 확장의 예시로는 `VK_KHR_swapchain`이 있으며, 이는 해당 장치에서 렌더링된 이미지를 창에 표시(present)할 수 있게 해줍니다. 시스템에 Vulkan 장치가 있지만 이 기능이 없을 수도 있는데, 예를 들어 컴퓨트 연산만 지원하는 경우가 그렇습니다. 이 확장은 스왑 체인 장에서 다시 다룰 것입니다.

이전 Vulkan 구현에서는 인스턴스와 장치 한정 유효성 검사 레이어를 구분했지만, [이제는 그렇지 않습니다](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap40.html#extendingvulkan-layers-devicelayerdeprecation). 이는 최신 구현에서는 `VkDeviceCreateInfo`의 `enabledLayerCount`와 `ppEnabledLayerNames` 필드가 무시된다는 것을 의미합니다. 하지만 구형 구현과의 호환성을 위해 이 값들을 설정해 두는 것이 여전히 좋은 생각입니다.

```c++
createInfo.enabledExtensionCount = 0;

if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

지금은 장치 한정 확장이 필요하지 않습니다.

이제 다 됐습니다. 적절한 이름의 `vkCreateDevice` 함수를 호출하여 논리 장치를 인스턴스화할 준비가 되었습니다.

```c++
if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS) {
    throw std::runtime_error("failed to create logical device!");
}
```

매개변수는 인터페이스할 물리 장치, 방금 명시한 큐 및 사용 정보, 선택적인 할당 콜백 포인터, 그리고 논리 장치 핸들을 저장할 변수를 가리키는 포인터입니다. 인스턴스 생성 함수와 마찬가지로, 이 호출은 존재하지 않는 확장을 활성화하거나 지원되지 않는 기능의 사용을 명시하는 경우 오류를 반환할 수 있습니다.

`cleanup`에서 `vkDestroyDevice` 함수로 장치를 파괴해야 합니다.

```c++
void cleanup() {
    vkDestroyDevice(device, nullptr);
    ...
}
```

논리 장치는 인스턴스와 직접 상호작용하지 않기 때문에, 매개변수로 포함되지 않습니다.

## 큐 핸들 가져오기

큐는 논리 장치와 함께 자동으로 생성되지만, 아직 그것들과 인터페이스할 핸들이 없습니다. 먼저 그래픽스 큐에 대한 핸들을 저장할 클래스 멤버를 추가합니다.

```c++
VkQueue graphicsQueue;
```

장치 큐는 장치가 파괴될 때 암시적으로 정리되므로 `cleanup`에서 아무것도 할 필요가 없습니다.

`vkGetDeviceQueue` 함수를 사용하여 각 큐 패밀리에 대한 큐 핸들을 가져올 수 있습니다. 매개변수는 논리 장치, 큐 패밀리, 큐 인덱스, 그리고 큐 핸들을 저장할 변수를 가리키는 포인터입니다. 이 패밀리에서는 단 하나의 큐만 생성하므로, 인덱스는 간단히 `0`을 사용합니다.

```c++
vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
```

논리 장치와 큐 핸들이 있으니, 이제 실제로 그래픽 카드를 사용하여 무언가를 할 수 있습니다! 다음 몇 개의 장에서는 결과를 창 시스템에 표시(present)하기 위한 리소스를 설정할 것입니다.

[C++ 코드](/code/04_logical_device.cpp)