이 페이지는 Vulkan 애플리케이션을 개발하면서 마주칠 수 있는 일반적인 문제들에 대한 해결책을 나열합니다.

## 코어 유효성 검사 레이어에서 액세스 위반 오류가 발생합니다

MSI Afterburner / RivaTuner Statistics Server가 실행 중이 아닌지 확인하세요. 이 프로그램들은 Vulkan과 호환성 문제가 있습니다.

## 유효성 검사 레이어에서 메시지가 표시되지 않거나 / 유효성 검사 레이어를 사용할 수 없습니다

먼저, 프로그램이 종료된 후에도 터미널을 열어 두어 유효성 검사 레이어가 오류를 출력할 기회를 갖도록 하세요. Visual Studio에서는 F5 대신 Ctrl-F5로 프로그램을 실행하고, Linux에서는 터미널 창에서 프로그램을 실행하여 이렇게 할 수 있습니다. 그래도 메시지가 표시되지 않고 유효성 검사 레이어가 켜져 있다고 확신한다면, [이 페이지](https://vulkan.lunarg.com/doc/view/1.2.135.0/windows/getting_started.html)의 "설치 확인" 지침에 따라 Vulkan SDK가 올바르게 설치되었는지 확인해야 합니다. 또한, `VK_LAYER_KHRONOS_validation` 레이어를 지원하려면 SDK 버전이 최소 1.1.106.0 이상인지 확인하세요.

## `vkCreateSwapchainKHR`가 `SteamOverlayVulkanLayer64.dll`에서 오류를 발생시킵니다

이것은 Steam 클라이언트 베타의 호환성 문제로 보입니다. 다음과 같은 몇 가지 해결 방법이 있습니다:
    * Steam 베타 프로그램 참여를 중단하세요.
    * `DISABLE_VK_LAYER_VALVE_steam_overlay_1` 환경 변수를 `1`로 설정하세요.
    * `HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ImplicitLayers` 아래 레지스트리에서 Steam 오버레이 Vulkan 레이어 항목을 삭제하세요.

예제:

![](/images/steam_layers_env.png)

## `vkCreateInstance`가 `VK_ERROR_INCOMPATIBLE_DRIVER`와 함께 실패합니다

최신 MoltenVK SDK와 함께 MacOS를 사용하고 있다면 `vkCreateInstance`가 `VK_ERROR_INCOMPATIBLE_DRIVER` 오류를 반환할 수 있습니다. 이는 [Vulkan SDK 버전 1.3.216 이상](https://vulkan.lunarg.com/doc/sdk/1.3.216.0/mac/getting_started.html)에서 MoltenVK를 사용하려면 `VK_KHR_PORTABILITY_subset` 확장을 활성화해야 하기 때문입니다. MoltenVK는 현재 완전히 표준을 준수하지 않기 때문입니다.

`VkInstanceCreateInfo`에 `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR` 플래그를 추가하고, 인스턴스 확장 목록에 `VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME`을 추가해야 합니다.

코드 예제:

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