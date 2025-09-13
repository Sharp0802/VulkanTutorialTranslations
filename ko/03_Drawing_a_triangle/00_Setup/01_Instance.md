## 인스턴스 생성하기

가장 먼저 해야 할 일은 *인스턴스*를 생성하여 Vulkan 라이브러리를 초기화하는 것입니다. 인스턴스는 여러분의 애플리케이션과 Vulkan 라이브러리 간의 연결고리이며, 인스턴스를 생성하는 과정에서 드라이버에 애플리케이션에 대한 몇 가지 세부 정보를 지정하게 됩니다.

`createInstance` 함수를 추가하고 `initVulkan` 함수에서 호출하는 것으로 시작합시다.

```c++
void initVulkan() {
    createInstance();
}
```

또한, 인스턴스 핸들을 저장할 데이터 멤버를 추가합니다:

```c++
private:
VkInstance instance;
```

이제 인스턴스를 생성하기 위해, 먼저 애플리케이션에 대한 정보를 담는 구조체를 채워야 합니다. 이 데이터는 기술적으로는 선택 사항이지만, 드라이버가 특정 애플리케이션을 최적화하는 데 유용한 정보를 제공할 수 있습니다 (예: 특정 특별한 동작을 가진 잘 알려진 그래픽 엔진을 사용하는 경우). 이 구조체는 `VkApplicationInfo`라고 합니다:

```c++
void createInstance() {
    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = "Hello Triangle";
    appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.apiVersion = VK_API_VERSION_1_0;
}
```

앞서 언급했듯이, Vulkan의 많은 구조체는 `sType` 멤버에 타입을 명시적으로 지정해야 합니다. 이 구조체 또한 앞으로 확장(extension) 정보를 가리킬 수 있는 `pNext` 멤버를 가진 많은 구조체 중 하나입니다. 여기서는 값 초기화(value initialization)를 사용하여 `nullptr`로 남겨둡니다.

Vulkan의 많은 정보는 함수 매개변수 대신 구조체를 통해 전달되며, 인스턴스를 생성하기에 충분한 정보를 제공하기 위해 구조체를 하나 더 채워야 합니다. 다음 구조체는 선택 사항이 아니며, 우리가 사용하고자 하는 전역 확장(global extension)과 유효성 검사 계층(validation layer)을 Vulkan 드라이버에 알려줍니다. 여기서 전역(Global)이란 특정 장치가 아닌 프로그램 전체에 적용된다는 의미이며, 이는 다음 몇 개의 챕터에서 명확해질 것입니다.

```c++
VkInstanceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
createInfo.pApplicationInfo = &appInfo;
```

첫 두 매개변수는 직관적입니다. 다음 두 멤버는 원하는 전역 확장을 지정합니다. 개요 챕터에서 언급했듯이, Vulkan은 플랫폼에 구애받지 않는(platform agnostic) API이므로, 윈도우 시스템과 인터페이스하려면 확장이 필요합니다. GLFW에는 이를 위해 필요한 확장을 반환하는 편리한 내장 함수가 있으며, 이 결과를 구조체에 전달할 수 있습니다:

```c++
uint32_t glfwExtensionCount = 0;
const char** glfwExtensions;

glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

createInfo.enabledExtensionCount = glfwExtensionCount;
createInfo.ppEnabledExtensionNames = glfwExtensions;
```

구조체의 마지막 두 멤버는 활성화할 전역 유효성 검사 계층을 결정합니다. 이에 대해서는 다음 챕터에서 더 자세히 다룰 것이므로, 지금은 비워두겠습니다.

```c++
createInfo.enabledLayerCount = 0;
```

이제 Vulkan이 인스턴스를 생성하는 데 필요한 모든 것을 지정했으므로, 마침내 `vkCreateInstance` 호출을 실행할 수 있습니다:

```c++
VkResult result = vkCreateInstance(&createInfo, nullptr, &instance);
```

보시다시피, Vulkan에서 객체 생성 함수의 매개변수가 따르는 일반적인 패턴은 다음과 같습니다:

*   생성 정보를 담은 구조체 포인터
*   사용자 정의 할당자 콜백 포인터, 이 튜토리얼에서는 항상 `nullptr`
*   새 객체의 핸들을 저장할 변수 포인터

모든 것이 순조롭게 진행되었다면, 인스턴스 핸들은 `VkInstance` 클래스 멤버에 저장되었을 것입니다. 거의 모든 Vulkan 함수는 `VK_SUCCESS` 또는 에러 코드인 `VkResult` 타입의 값을 반환합니다. 인스턴스가 성공적으로 생성되었는지 확인하기 위해, 결과를 저장할 필요 없이 성공 값을 바로 확인할 수 있습니다:

```c++
if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```

이제 프로그램을 실행하여 인스턴스가 성공적으로 생성되는지 확인하세요.

## `VK_ERROR_INCOMPATIBLE_DRIVER` 오류 발생 시:
최신 MoltenVK SDK와 함께 MacOS를 사용하는 경우 `vkCreateInstance`에서 `VK_ERROR_INCOMPATIBLE_DRIVER` 오류가 반환될 수 있습니다. [시작 노트](https://vulkan.lunarg.com/doc/sdk/1.3.216.0/mac/getting_started.html)에 따르면, 1.3.216 Vulkan SDK부터 `VK_KHR_PORTABILITY_subset` 확장이 필수입니다.

이 오류를 해결하려면, 먼저 `VkInstanceCreateInfo` 구조체의 `flags`에 `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` 비트를 추가하고, 그 다음 인스턴스 활성화 확장 목록에 `VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME`을 추가해야 합니다.

일반적으로 코드는 다음과 같을 수 있습니다:
```c++
...

std::vector<const char*> requiredExtensions;

for(uint32_t i = 0; i < glfwExtensionCount; i++) {
    requiredExtensions.emplace_back(glfwExtensions[i]);
}

requiredExtensions.emplace_back(VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME);

createInfo.flags |= VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR;

createInfo.enabledExtensionCount = (uint32_t) requiredExtensions.size();
createInfo.ppEnabledExtensionNames = requiredExtensions.data();

if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```

## 확장 지원 확인하기

`vkCreateInstance` 문서를 보면, 가능한 에러 코드 중 하나가 `VK_ERROR_EXTENSION_NOT_PRESENT`임을 알 수 있습니다. 필요한 확장을 지정하고 해당 에러 코드가 반환되면 프로그램을 종료할 수도 있습니다. 윈도우 시스템 인터페이스와 같은 필수적인 확장의 경우에는 이 방법이 타당하지만, 선택적인 기능을 확인하고 싶다면 어떻게 해야 할까요?

인스턴스를 생성하기 전에 지원되는 확장 목록을 가져오기 위해 `vkEnumerateInstanceExtensionProperties` 함수가 있습니다. 이 함수는 확장의 수를 저장하는 변수 포인터와 확장의 세부 정보를 저장할 `VkExtensionProperties` 배열을 인자로 받습니다. 또한 특정 유효성 검사 계층으로 확장을 필터링할 수 있는 선택적 첫 번째 매개변수도 있지만, 지금은 무시하겠습니다.

확장 세부 정보를 담을 배열을 할당하려면 먼저 몇 개가 있는지 알아야 합니다. 후자의 매개변수를 비워두면 확장의 개수만 요청할 수 있습니다:

```c++
uint32_t extensionCount = 0;
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
```

이제 확장 세부 정보를 담을 배열을 할당합니다 (`<vector>`를 포함하세요):

```c++
std::vector<VkExtensionProperties> extensions(extensionCount);
```

마지막으로 확장 세부 정보를 질의할 수 있습니다:

```c++
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, extensions.data());
```

각 `VkExtensionProperties` 구조체에는 확장의 이름과 버전이 포함되어 있습니다. 간단한 for 루프를 사용하여 목록을 출력할 수 있습니다 (`\t`는 들여쓰기를 위한 탭입니다):

```c++
std::cout << "available extensions:\n";

for (const auto& extension : extensions) {
    std::cout << '\t' << extension.extensionName << '\n';
}
```

Vulkan 지원에 대한 세부 정보를 제공하고 싶다면 이 코드를 `createInstance` 함수에 추가할 수 있습니다. 도전 과제로, `glfwGetRequiredInstanceExtensions`가 반환하는 모든 확장이 지원되는 확장 목록에 포함되어 있는지 확인하는 함수를 만들어 보세요.

## 정리하기

`VkInstance`는 프로그램이 종료되기 직전에만 파괴되어야 합니다. `vkDestroyInstance` 함수를 사용하여 `cleanup`에서 파괴할 수 있습니다:

```c++
void cleanup() {
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

`vkDestroyInstance` 함수의 매개변수는 간단합니다. 이전 챕터에서 언급했듯이, Vulkan의 할당 및 해제 함수에는 선택적인 할당자 콜백이 있으며, 우리는 `nullptr`를 전달하여 이를 무시할 것입니다. 이후 챕터에서 생성할 다른 모든 Vulkan 리소스는 인스턴스가 파괴되기 전에 정리되어야 합니다.

인스턴스 생성 이후의 더 복잡한 단계로 넘어가기 전에, [유효성 검사 계층](!ko/Drawing_a_triangle/Setup/Validation_layers)을 확인하여 디버깅 옵션을 평가할 시간입니다.

[C++ 코드](/code/01_instance_creation.cpp)