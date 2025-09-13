## 유효성 검사 레이어란?

Vulkan API는 최소한의 드라이버 오버헤드라는 아이디어를 중심으로 설계되었으며, 그 목표의 한 가지 표현은 API에 기본적으로 오류 검사가 매우 제한적으로 포함된다는 점입니다. 열거형을 잘못된 값으로 설정하거나 필수 매개변수에 null 포인터를 전달하는 것과 같은 간단한 실수조차도 일반적으로 명시적으로 처리되지 않으며, 단순히 충돌이나 정의되지 않은 동작으로 이어질 뿐입니다.
Vulkan은 여러분이 하는 모든 일에 대해 매우 명시적일 것을 요구하기 때문에, 새로운 GPU 기능을 사용하면서 논리 장치(logical device) 생성 시에 이를 요청하는 것을 잊는 것과 같은 많은 작은 실수를 하기 쉽습니다.

하지만 이러한 검사를 API에 추가할 수 없다는 의미는 아닙니다. Vulkan은 이를 위해 *유효성 검사 레이어(validation layers)*라고 알려진 멋진 시스템을 도입했습니다. 유효성 검사 레이어는 추가적인 작업을 적용하기 위해 Vulkan 함수 호출에 연결되는 선택적 구성 요소입니다. 유효성 검사 레이어의 일반적인 작업은 다음과 같습니다.

*   사양과 비교하여 매개변수 값을 확인하여 오용을 감지
*   객체의 생성 및 소멸을 추적하여 리소스 누수(resource leak)를 찾음
*   호출이 시작된 스레드를 추적하여 스레드 안전성(thread safety)을 확인
*   모든 호출과 그 매개변수를 표준 출력에 로깅
*   프로파일링 및 리플레이를 위해 Vulkan 호출을 추적

진단용 유효성 검사 레이어에서 함수 구현이 어떤 모습일지에 대한 예는 다음과 같습니다.

```c++
VkResult vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* instance) {

    if (pCreateInfo == nullptr || instance == nullptr) {
        log("필수 매개변수에 null 포인터가 전달되었습니다!");
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    return real_vkCreateInstance(pCreateInfo, pAllocator, instance);
}
```

이러한 유효성 검사 레이어는 관심 있는 모든 디버깅 기능을 포함하도록 자유롭게 쌓을 수 있습니다. 디버그 빌드에서는 유효성 검사 레이어를 활성화하고 릴리스 빌드에서는 완전히 비활성화할 수 있어, 양쪽의 장점을 모두 취할 수 있습니다!

Vulkan에는 내장된 유효성 검사 레이어가 없지만, LunarG Vulkan SDK는 일반적인 오류를 확인하는 훌륭한 레이어 세트를 제공합니다. 또한 이들은 완전히 [오픈 소스](https://github.com/KhronosGroup/Vulkan-ValidationLayers)이므로, 어떤 종류의 실수를 확인하는지 살펴보고 기여할 수 있습니다. 유효성 검사 레이어를 사용하는 것은 실수로 정의되지 않은 동작에 의존하여 애플리케이션이 다른 드라이버에서 깨지는 것을 방지하는 가장 좋은 방법입니다.

유효성 검사 레이어는 시스템에 설치된 경우에만 사용할 수 있습니다. 예를 들어, LunarG 유효성 검사 레이어는 Vulkan SDK가 설치된 PC에서만 사용할 수 있습니다.

이전에는 Vulkan에 인스턴스(instance)와 장치 특정(device specific)이라는 두 가지 다른 유형의 유효성 검사 레이어가 있었습니다. 인스턴스 레이어는 인스턴스와 같은 전역 Vulkan 객체와 관련된 호출만 확인하고, 장치 특정 레이어는 특정 GPU와 관련된 호출만 확인한다는 아이디어였습니다. 장치 특정 레이어는 이제 사용 중단되었으며, 이는 인스턴스 유효성 검사 레이어가 모든 Vulkan 호출에 적용된다는 것을 의미합니다. 사양 문서는 여전히 일부 구현에서 요구되는 호환성을 위해 장치 수준에서도 유효성 검사 레이어를 활성화할 것을 권장합니다. 우리는 논리 장치 수준에서 인스턴스와 동일한 레이어를 지정할 것이며, 이는 [나중에](!ko/Drawing_a_triangle/Setup/Logical_device_and_queues) 보게 될 것입니다.

## 유효성 검사 레이어 사용하기

이 섹션에서는 Vulkan SDK에서 제공하는 표준 진단 레이어를 활성화하는 방법을 살펴보겠습니다. 확장(extension)과 마찬가지로, 유효성 검사 레이어는 이름을 지정하여 활성화해야 합니다. 모든 유용한 표준 유효성 검사는 `VK_LAYER_KHRONOS_validation`으로 알려진 SDK에 포함된 레이어로 묶여 있습니다.

먼저 프로그램에 활성화할 레이어와 활성화 여부를 지정하는 두 개의 구성 변수를 추가합시다. 저는 프로그램이 디버그 모드에서 컴파일되는지 여부에 따라 그 값을 결정하기로 했습니다. `NDEBUG` 매크로는 C++ 표준의 일부이며 "not debug"를 의미합니다.

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;

const std::vector<const char*> validationLayers = {
    "VK_LAYER_KHRONOS_validation"
};

#ifdef NDEBUG
    const bool enableValidationLayers = false;
#else
    const bool enableValidationLayers = true;
#endif
```

요청된 모든 레이어를 사용할 수 있는지 확인하는 새로운 함수 `checkValidationLayerSupport`를 추가할 것입니다. 먼저 `vkEnumerateInstanceLayerProperties` 함수를 사용하여 사용 가능한 모든 레이어를 나열합니다. 그 사용법은 인스턴스 생성 챕터에서 논의된 `vkEnumerateInstanceExtensionProperties`의 사용법과 동일합니다.

```c++
bool checkValidationLayerSupport() {
    uint32_t layerCount;
    vkEnumerateInstanceLayerProperties(&layerCount, nullptr);

    std::vector<VkLayerProperties> availableLayers(layerCount);
    vkEnumerateInstanceLayerProperties(&layerCount, availableLayers.data());

    return false;
}
```

다음으로, `validationLayers`의 모든 레이어가 `availableLayers` 목록에 있는지 확인합니다. `strcmp`를 위해 `<cstring>`을 포함해야 할 수도 있습니다.

```c++
for (const char* layerName : validationLayers) {
    bool layerFound = false;

    for (const auto& layerProperties : availableLayers) {
        if (strcmp(layerName, layerProperties.layerName) == 0) {
            layerFound = true;
            break;
        }
    }

    if (!layerFound) {
        return false;
    }
}

return true;
```

이제 `createInstance`에서 이 함수를 사용할 수 있습니다.

```c++
void createInstance() {
    if (enableValidationLayers && !checkValidationLayerSupport()) {
        throw std::runtime_error("유효성 검사 레이어를 요청했지만, 사용할 수 없습니다!");
    }

    ...
}
```

이제 프로그램을 디버그 모드로 실행하고 오류가 발생하지 않는지 확인하세요. 만약 발생한다면 FAQ를 살펴보세요.

마지막으로, `VkInstanceCreateInfo` 구조체 인스턴스화를 수정하여 유효성 검사 레이어가 활성화된 경우 해당 레이어 이름을 포함하도록 합니다.

```c++
if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

확인이 성공적이었다면 `vkCreateInstance`는 절대로 `VK_ERROR_LAYER_NOT_PRESENT` 오류를 반환해서는 안 되지만, 프로그램을 실행하여 확실히 해두어야 합니다.

## 메시지 콜백

유효성 검사 레이어는 기본적으로 디버그 메시지를 표준 출력으로 인쇄하지만, 우리 프로그램에서 명시적인 콜백을 제공하여 직접 처리할 수도 있습니다. 이를 통해 어떤 종류의 메시지를 보고 싶은지 결정할 수도 있습니다. 모든 메시지가 반드시 (치명적인) 오류는 아니기 때문입니다. 지금 당장 이 작업을 하고 싶지 않다면, 이 챕터의 마지막 섹션으로 건너뛰어도 좋습니다.

메시지와 관련 세부 정보를 처리하기 위해 프로그램에 콜백을 설정하려면, `VK_EXT_debug_utils` 확장을 사용하여 콜백이 있는 디버그 메신저를 설정해야 합니다.

먼저 유효성 검사 레이어의 활성화 여부에 따라 필요한 확장 목록을 반환하는 `getRequiredExtensions` 함수를 만들 것입니다.

```c++
std::vector<const char*> getRequiredExtensions() {
    uint32_t glfwExtensionCount = 0;
    const char** glfwExtensions;
    glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

    std::vector<const char*> extensions(glfwExtensions, glfwExtensions + glfwExtensionCount);

    if (enableValidationLayers) {
        extensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
    }

    return extensions;
}
```

GLFW가 지정한 확장은 항상 필요하지만, 디버그 메신저 확장은 조건부로 추가됩니다. 여기서 저는 리터럴 문자열 "VK_EXT_debug_utils"와 동일한 `VK_EXT_DEBUG_UTILS_EXTENSION_NAME` 매크로를 사용했습니다. 이 매크로를 사용하면 오타를 피할 수 있습니다.

이제 `createInstance`에서 이 함수를 사용할 수 있습니다.

```c++
auto extensions = getRequiredExtensions();
createInfo.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
createInfo.ppEnabledExtensionNames = extensions.data();
```

프로그램을 실행하여 `VK_ERROR_EXTENSION_NOT_PRESENT` 오류가 발생하지 않는지 확인하세요. 이 확장의 존재를 굳이 확인할 필요는 없습니다. 유효성 검사 레이어를 사용할 수 있다는 것은 이 확장을 사용할 수 있다는 것을 암시하기 때문입니다.

이제 디버그 콜백 함수가 어떻게 생겼는지 봅시다. `PFN_vkDebugUtilsMessengerCallbackEXT` 프로토타입을 가진 `debugCallback`이라는 새로운 정적 멤버 함수를 추가하세요. `VKAPI_ATTR`과 `VKAPI_CALL`은 함수가 Vulkan이 호출할 수 있는 올바른 시그니처를 갖도록 보장합니다.

```c++
static VKAPI_ATTR VkBool32 VKAPI_CALL debugCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
    VkDebugUtilsMessageTypeFlagsEXT messageType,
    const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
    void* pUserData) {

    std::cerr << "유효성 검사 레이어: " << pCallbackData->pMessage << std::endl;

    return VK_FALSE;
}
```

첫 번째 매개변수는 메시지의 심각도(severity)를 지정하며, 다음 플래그 중 하나입니다.

*   `VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT`: 진단 메시지
*   `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`: 리소스 생성과 같은 정보성 메시지
*   `VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT`: 반드시 오류는 아니지만 애플리케이션에 버그일 가능성이 매우 높은 동작에 대한 메시지
*   `VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT`: 유효하지 않으며 충돌을 일으킬 수 있는 동작에 대한 메시지

이 열거형의 값은 비교 연산을 사용하여 메시지가 특정 심각도 수준과 같거나 더 나쁜지 확인할 수 있도록 설정되어 있습니다. 예를 들어:

```c++
if (messageSeverity >= VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT) {
    // 메시지가 표시할 만큼 중요함
}
```

`messageType` 매개변수는 다음 값을 가질 수 있습니다.

*   `VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT`: 사양이나 성능과 관련 없는 어떤 이벤트가 발생함
*   `VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT`: 사양을 위반하거나 가능한 실수를 나타내는 일이 발생함
*   `VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT`: Vulkan의 잠재적인 비최적 사용

`pCallbackData` 매개변수는 메시지 자체의 세부 정보를 포함하는 `VkDebugUtilsMessengerCallbackDataEXT` 구조체를 참조하며, 가장 중요한 멤버는 다음과 같습니다.

*   `pMessage`: null로 끝나는 문자열 형태의 디버그 메시지
*   `pObjects`: 메시지와 관련된 Vulkan 객체 핸들 배열
*   `objectCount`: 배열에 있는 객체의 수

마지막으로, `pUserData` 매개변수는 콜백 설정 중에 지정된 포인터를 포함하며, 이를 통해 자신의 데이터를 콜백에 전달할 수 있습니다.

콜백은 유효성 검사 레이어 메시지를 유발한 Vulkan 호출을 중단해야 하는지를 나타내는 불리언 값을 반환합니다. 콜백이 true를 반환하면, 호출은 `VK_ERROR_VALIDATION_FAILED_EXT` 오류와 함께 중단됩니다. 이는 보통 유효성 검사 레이어 자체를 테스트하는 데만 사용되므로, 항상 `VK_FALSE`를 반환해야 합니다.

이제 남은 것은 Vulkan에 콜백 함수에 대해 알려주는 것입니다. 다소 놀랍게도, Vulkan의 디버그 콜백조차도 명시적으로 생성하고 파괴해야 하는 핸들로 관리됩니다. 이러한 콜백은 *디버그 메신저*의 일부이며, 원하는 만큼 많이 가질 수 있습니다. `instance` 바로 아래에 이 핸들을 위한 클래스 멤버를 추가하세요.

```c++
VkDebugUtilsMessengerEXT debugMessenger;
```

이제 `initVulkan`에서 `createInstance` 바로 다음에 호출될 `setupDebugMessenger` 함수를 추가하세요.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
}

void setupDebugMessenger() {
    if (!enableValidationLayers) return;

}
```

메신저와 그 콜백에 대한 세부 정보로 구조체를 채워야 합니다.

```c++
VkDebugUtilsMessengerCreateInfoEXT createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
createInfo.pfnUserCallback = debugCallback;
createInfo.pUserData = nullptr; // Optional
```

`messageSeverity` 필드를 사용하면 콜백이 호출되기를 원하는 모든 심각도 유형을 지정할 수 있습니다. 저는 여기서 `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`를 제외한 모든 유형을 지정하여, 상세한 일반 디버그 정보는 제외하고 가능한 문제에 대한 알림을 받도록 했습니다.

마찬가지로 `messageType` 필드를 사용하면 콜백이 알림을 받을 메시지 유형을 필터링할 수 있습니다. 저는 여기서 간단히 모든 유형을 활성화했습니다. 만약 유용하지 않다면 언제든지 일부를 비활성화할 수 있습니다.

마지막으로 `pfnUserCallback` 필드는 콜백 함수에 대한 포인터를 지정합니다. 선택적으로 `pUserData` 필드에 포인터를 전달할 수 있으며, 이는 `pUserData` 매개변수를 통해 콜백 함수로 전달됩니다. 예를 들어, 이를 사용하여 `HelloTriangleApplication` 클래스에 대한 포인터를 전달할 수 있습니다.

유효성 검사 레이어 메시지와 디버그 콜백을 구성하는 방법은 이 외에도 많지만, 이 튜토리얼을 시작하기에는 이 설정이 좋습니다. 가능한 옵션에 대한 자세한 정보는 [확장 사양](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap50.html#VK_EXT_debug_utils)을 참조하세요.

이 구조체는 `VkDebugUtilsMessengerEXT` 객체를 생성하기 위해 `vkCreateDebugUtilsMessengerEXT` 함수에 전달되어야 합니다. 불행히도, 이 함수는 확장 함수이기 때문에 자동으로 로드되지 않습니다. 우리는 `vkGetInstanceProcAddr`를 사용하여 그 주소를 직접 찾아야 합니다. 이를 백그라운드에서 처리하는 우리 자신의 프록시 함수를 만들 것입니다. 저는 그것을 `HelloTriangleApplication` 클래스 정의 바로 위에 추가했습니다.

```c++
VkResult CreateDebugUtilsMessengerEXT(VkInstance instance, const VkDebugUtilsMessengerCreateInfoEXT* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkDebugUtilsMessengerEXT* pDebugMessenger) {
    auto func = (PFN_vkCreateDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkCreateDebugUtilsMessengerEXT");
    if (func != nullptr) {
        return func(instance, pCreateInfo, pAllocator, pDebugMessenger);
    } else {
        return VK_ERROR_EXTENSION_NOT_PRESENT;
    }
}
```

`vkGetInstanceProcAddr` 함수는 함수를 로드할 수 없는 경우 `nullptr`를 반환합니다. 이제 이 함수를 호출하여 확장 객체를 생성할 수 있습니다(사용 가능한 경우).

```c++
if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS) {
    throw std::runtime_error("디버그 메신저를 설정하는 데 실패했습니다!");
}
```

뒤에서 두 번째 매개변수는 다시 선택적 할당자 콜백으로, 우리는 `nullptr`로 설정했습니다. 그 외의 매개변수들은 상당히 직관적입니다. 디버그 메신저는 우리의 Vulkan 인스턴스와 그 레이어에 특정적이므로, 첫 번째 인자로 명시적으로 지정해야 합니다. 나중에 다른 *자식* 객체에서도 이 패턴을 보게 될 것입니다.

`VkDebugUtilsMessengerEXT` 객체는 `vkDestroyDebugUtilsMessengerEXT` 호출로 정리해야 합니다. `vkCreateDebugUtilsMessengerEXT`와 유사하게, 이 함수도 명시적으로 로드해야 합니다.

`CreateDebugUtilsMessengerEXT` 바로 아래에 또 다른 프록시 함수를 만드세요.

```c++
void DestroyDebugUtilsMessengerEXT(VkInstance instance, VkDebugUtilsMessengerEXT debugMessenger, const VkAllocationCallbacks* pAllocator) {
    auto func = (PFN_vkDestroyDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkDestroyDebugUtilsMessengerEXT");
    if (func != nullptr) {
        func(instance, debugMessenger, pAllocator);
    }
}
```

이 함수가 정적 클래스 함수이거나 클래스 외부의 함수인지 확인하세요. 그러면 `cleanup` 함수에서 호출할 수 있습니다.

```c++
void cleanup() {
    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }

    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

## 인스턴스 생성 및 소멸 디버깅

이제 유효성 검사 레이어를 사용하여 프로그램에 디버깅을 추가했지만 아직 모든 것을 다루지는 않았습니다. `vkCreateDebugUtilsMessengerEXT` 호출은 유효한 인스턴스가 생성되어야 하고, `vkDestroyDebugUtilsMessengerEXT`는 인스턴스가 파괴되기 전에 호출되어야 합니다. 이로 인해 현재로서는 `vkCreateInstance`와 `vkDestroyInstance` 호출의 문제를 디버깅할 수 없습니다.

하지만 [확장 문서](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_debug_utils.adoc#examples)를 자세히 읽어보면, 이 두 함수 호출을 위해 특별히 별도의 디버그 유틸리티 메신저를 생성하는 방법이 있다는 것을 알 수 있습니다. `VkInstanceCreateInfo`의 `pNext` 확장 필드에 `VkDebugUtilsMessengerCreateInfoEXT` 구조체에 대한 포인터를 전달하기만 하면 됩니다. 먼저 메신저 생성 정보 채우기를 별도의 함수로 추출합시다.

```c++
void populateDebugMessengerCreateInfo(VkDebugUtilsMessengerCreateInfoEXT& createInfo) {
    createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
    createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
    createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
    createInfo.pfnUserCallback = debugCallback;
}

...

void setupDebugMessenger() {
    if (!enableValidationLayers) return;

    VkDebugUtilsMessengerCreateInfoEXT createInfo;
    populateDebugMessengerCreateInfo(createInfo);

    if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS) {
        throw std::runtime_error("디버그 메신저를 설정하는 데 실패했습니다!");
    }
}
```

이제 `createInstance` 함수에서 이것을 재사용할 수 있습니다.

```c++
void createInstance() {
    ...

    VkInstanceCreateInfo createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    createInfo.pApplicationInfo = &appInfo;

    ...

    VkDebugUtilsMessengerCreateInfoEXT debugCreateInfo{};
    if (enableValidationLayers) {
        createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
        createInfo.ppEnabledLayerNames = validationLayers.data();

        populateDebugMessengerCreateInfo(debugCreateInfo);
        createInfo.pNext = (VkDebugUtilsMessengerCreateInfoEXT*) &debugCreateInfo;
    } else {
        createInfo.enabledLayerCount = 0;

        createInfo.pNext = nullptr;
    }

    if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
        throw std::runtime_error("인스턴스를 생성하는 데 실패했습니다!");
    }
}
```

`debugCreateInfo` 변수는 `vkCreateInstance` 호출 전에 파괴되지 않도록 if문 외부에 배치됩니다. 이런 식으로 추가 디버그 메신저를 생성하면 `vkCreateInstance`와 `vkDestroyInstance` 동안 자동으로 사용되고 그 이후에 정리됩니다.

## 테스트

이제 의도적으로 실수를 만들어 유효성 검사 레이어가 작동하는 것을 확인해 봅시다. `cleanup` 함수에서 `DestroyDebugUtilsMessengerEXT` 호출을 일시적으로 제거하고 프로그램을 실행하세요. 프로그램이 종료되면 다음과 같은 것을 보게 될 것입니다.

![](/images/validation_layer_test.png)

>만약 아무 메시지도 보이지 않는다면 [설치를 확인](https://vulkan.lunarg.com/doc/view/1.2.131.1/windows/getting_started.html#user-content-verify-the-installation)하세요.

어떤 호출이 메시지를 유발했는지 확인하고 싶다면, 메시지 콜백에 중단점(breakpoint)을 설정하고 스택 트레이스(stack trace)를 살펴보세요.

## 구성

유효성 검사 레이어의 동작에 대한 설정은 `VkDebugUtilsMessengerCreateInfoEXT` 구조체에 지정된 플래그 외에도 훨씬 많습니다. Vulkan SDK로 이동하여 `Config` 디렉터리로 가세요. 거기에서 레이어를 구성하는 방법을 설명하는 `vk_layer_settings.txt` 파일을 찾을 수 있습니다.

자신의 애플리케이션에 대한 레이어 설정을 구성하려면, 이 파일을 프로젝트의 `Debug` 및 `Release` 디렉터리에 복사하고 지침에 따라 원하는 동작을 설정하세요. 그러나 이 튜토리얼의 나머지 부분에서는 기본 설정을 사용한다고 가정하겠습니다.

이 튜토리얼 전반에 걸쳐, 유효성 검사 레이어가 실수를 잡는 데 얼마나 도움이 되는지 보여주고 Vulkan으로 무엇을 하고 있는지 정확히 아는 것이 얼마나 중요한지 가르쳐 드리기 위해 몇 가지 의도적인 실수를 할 것입니다. 이제 [시스템의 Vulkan 장치](!ko/Drawing_a_triangle/Setup/Physical_devices_and_queue_families)에 대해 알아볼 시간입니다.

[C++ 코드](/code/02_validation_layers.cpp)