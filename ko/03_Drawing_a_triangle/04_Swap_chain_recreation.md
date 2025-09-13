## 들어가며

지금 우리가 만든 애플리케이션은 성공적으로 삼각형을 그리지만, 아직 제대로 처리하지 못하는 몇 가지 상황이 있습니다. 창 서피스가 변경되어 스왑 체인이 더 이상 호환되지 않게 될 수 있습니다. 이러한 현상이 발생하는 원인 중 하나는 창 크기 변경입니다. 우리는 이러한 이벤트를 잡아내어 스왑 체인을 재생성해야 합니다.

## 스왑 체인 재생성하기

`createSwapChain`과, 스왑 체인 또는 창 크기에 의존하는 모든 객체들의 생성 함수를 호출하는 새로운 `recreateSwapChain` 함수를 만드세요.

```c++
void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    createSwapChain();
    createImageViews();
    createFramebuffers();
}
```

지난 장에서와 마찬가지로 아직 사용 중일 수 있는 리소스는 건드리면 안 되므로, 먼저 `vkDeviceWaitIdle`을 호출합니다. 당연히 스왑 체인 자체를 재생성해야 합니다. 이미지 뷰는 스왑 체인 이미지에 직접적으로 기반하므로 재생성해야 합니다. 마지막으로, 프레임버퍼는 스왑 체인 이미지에 직접적으로 의존하므로 마찬가지로 재생성해야 합니다.

이러한 객체들의 이전 버전이 재생성 전에 정리(cleanup)되도록 하기 위해, 일부 정리 코드를 `recreateSwapChain` 함수에서 호출할 수 있는 별도의 함수로 옮겨야 합니다. 이 함수를 `cleanupSwapChain`이라고 부르겠습니다:

```c++
void cleanupSwapChain() {

}

void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createFramebuffers();
}
```

참고로 여기서는 간단히 하기 위해 렌더 패스를 재생성하지 않습니다. 이론적으로는 애플리케이션의 생명 주기 동안 스왑 체인 이미지 포맷이 변경될 수 있습니다. 예를 들어, 표준 범위(standard range) 모니터에서 높은 동적 범위(high dynamic range) 모니터로 창을 옮길 때입니다. 이 경우, 동적 범위 간의 변경이 올바르게 반영되도록 애플리케이션이 렌더 패스를 재생성해야 할 수도 있습니다.

`cleanup`에서 스왑 체인 갱신의 일부로 재생성되는 모든 객체의 정리 코드를 `cleanupSwapChain`으로 옮기겠습니다:

```c++
void cleanupSwapChain() {
    for (auto framebuffer : swapChainFramebuffers) {
        vkDestroyFramebuffer(device, framebuffer, nullptr);
    }

    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    vkDestroySwapchainKHR(device, swapChain, nullptr);
}

void cleanup() {
    cleanupSwapChain();

    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);

    vkDestroyRenderPass(device, renderPass, nullptr);

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    vkDestroyCommandPool(device, commandPool, nullptr);

    vkDestroyDevice(device, nullptr);

    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }

    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

`chooseSwapExtent`에서는 이미 새 창 해상도를 쿼리하여 스왑 체인 이미지의 크기가 (새로운) 올바른 크기를 갖도록 하므로 `chooseSwapExtent`를 수정할 필요는 없습니다(스왑 체인을 만들 때 이미 서피스의 해상도를 픽셀 단위로 얻기 위해 `glfwGetFramebufferSize`를 사용해야 했던 것을 기억하세요).

이것만으로 스왑 체인을 재생성할 수 있습니다! 하지만 이 접근 방식의 단점은 새 스왑 체인을 만들기 전에 모든 렌더링을 중지해야 한다는 것입니다. 이전 스왑 체인의 이미지에 대한 드로잉 커맨드가 아직 처리 중(in-flight)인 동안 새 스왑 체인을 생성하는 것이 가능합니다. `VkSwapchainCreateInfoKHR` 구조체의 `oldSwapChain` 필드에 이전 스왑 체인을 전달하고, 이전 스왑 체인 사용을 마친 직후에 파괴(소멸)해야 합니다.

## 최적이 아니거나 오래된 스왑 체인

이제 스왑 체인 재생성이 언제 필요한지 파악하고 새로운 `recreateSwapChain` 함수를 호출하기만 하면 됩니다. 다행히 Vulkan은 보통 프레젠테이션 중에 스왑 체인이 더 이상 적합하지 않다고 알려줍니다. `vkAcquireNextImageKHR`와 `vkQueuePresentKHR` 함수는 이를 나타내기 위해 다음과 같은 특별한 값을 반환할 수 있습니다.

*   `VK_ERROR_OUT_OF_DATE_KHR`: 스왑 체인이 서피스와 호환되지 않게 되어 더 이상 렌더링에 사용할 수 없습니다. 보통 창 크기 변경 후에 발생합니다.
*   `VK_SUBOPTIMAL_KHR`: 스왑 체인은 여전히 서피스에 성공적으로 프레젠테이션하는 데 사용할 수 있지만, 서피스 속성과 더 이상 정확하게 일치하지 않습니다.

```c++
VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

if (result == VK_ERROR_OUT_OF_DATE_KHR) {
    recreateSwapChain();
    return;
} else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
    throw std::runtime_error("failed to acquire swap chain image!");
}
```

이미지를 획득하려고 할 때 스왑 체인이 오래된(out of date) 것으로 판명되면 더 이상 프레젠테이션할 수 없습니다. 따라서 즉시 스왑 체인을 재생성하고 다음 `drawFrame` 호출에서 다시 시도해야 합니다.

스왑 체인이 최적이 아닌(suboptimal) 경우에도 그렇게 할 수 있지만, 이 경우에는 이미 이미지를 획득했으므로 그대로 진행하기로 했습니다. `VK_SUCCESS`와 `VK_SUBOPTIMAL_KHR`는 모두 "성공" 반환 코드로 간주됩니다.

```c++
result = vkQueuePresentKHR(presentQueue, &presentInfo);

if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR) {
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    throw std::runtime_error("failed to present swap chain image!");
}

currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
```

`vkQueuePresentKHR` 함수도 동일한 의미로 동일한 값을 반환합니다. 이 경우, 최상의 결과를 원하기 때문에 스왑 체인이 최적이 아니더라도 재생성할 것입니다.

## 교착 상태 해결하기

지금 코드를 실행하려고 하면 교착 상태(deadlock)에 빠질 수 있습니다. 코드를 디버깅해보면 애플리케이션이 `vkWaitForFences`에 도달하지만 그 이상 진행하지 않는 것을 발견할 수 있습니다. 이는 `vkAcquireNextImageKHR`가 `VK_ERROR_OUT_OF_DATE_KHR`을 반환할 때 스왑 체인을 재생성한 후 `drawFrame`에서 반환하기 때문입니다. 하지만 그 전에 현재 프레임의 펜스가 대기 후 리셋됩니다. 즉시 반환하기 때문에 실행을 위한 작업이 제출되지 않고 펜스는 절대 신호를 받지 못하게 되어, `vkWaitForFences`가 영원히 멈추게 됩니다.

다행히 간단한 해결책이 있습니다. 펜스를 사용해 작업을 제출할 것이 확실해진 후에 펜스 리셋을 지연시키는 것입니다. 이렇게 하면, 일찍 반환하더라도 펜스는 여전히 신호를 받은 상태이므로 다음에 같은 펜스 객체를 사용할 때 `vkWaitForFences`가 교착 상태에 빠지지 않습니다.

`drawFrame`의 시작 부분은 이제 다음과 같아야 합니다:
```c++
vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

uint32_t imageIndex;
VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

if (result == VK_ERROR_OUT_OF_DATE_KHR) {
    recreateSwapChain();
    return;
} else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
    throw std::runtime_error("failed to acquire swap chain image!");
}

// 작업을 제출하는 경우에만 펜스를 리셋합니다
vkResetFences(device, 1, &inFlightFences[currentFrame]);
```

## 창 크기 변경 명시적으로 처리하기

많은 드라이버와 플랫폼이 창 크기 변경 후 `VK_ERROR_OUT_OF_DATE_KHR`을 자동으로 발생시키지만, 이것이 보장되지는 않습니다. 그래서 크기 변경을 명시적으로 처리하기 위한 추가 코드를 넣을 것입니다. 먼저 크기 변경이 발생했음을 알리는 플래그 역할을 할 새 멤버 변수를 추가합니다:

```c++
std::vector<VkFence> inFlightFences;

bool framebufferResized = false;
```

그런 다음 `drawFrame` 함수가 이 플래그도 확인하도록 수정해야 합니다:

```c++
if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized) {
    framebufferResized = false;
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    ...
}
```

세마포어가 일관된 상태를 유지하도록 `vkQueuePresentKHR` 이후에 이 작업을 수행하는 것이 중요합니다. 그렇지 않으면 신호를 받은 세마포어가 제대로 대기되지 않을 수 있습니다. 이제 실제로 크기 변경을 감지하기 위해 GLFW 프레임워크의 `glfwSetFramebufferSizeCallback` 함수를 사용하여 콜백을 설정할 수 있습니다:

```c++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
    glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
}

static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {

}
```

콜백으로 `static` 함수를 만드는 이유는 GLFW가 `HelloTriangleApplication` 인스턴스를 가리키는 올바른 `this` 포인터로 멤버 함수를 어떻게 호출해야 하는지 모르기 때문입니다.

하지만 콜백에서 `GLFWwindow`에 대한 참조를 얻을 수 있으며, 임의의 포인터를 저장할 수 있는 또 다른 GLFW 함수가 있습니다: `glfwSetWindowUserPointer`:

```c++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
glfwSetWindowUserPointer(window, this);
glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
```

이제 `glfwGetWindowUserPointer`를 사용하여 콜백 내에서 이 값을 가져와 플래그를 올바르게 설정할 수 있습니다:

```c++
static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {
    auto app = reinterpret_cast<HelloTriangleApplication*>(glfwGetWindowUserPointer(window));
    app->framebufferResized = true;
}
```

이제 프로그램을 실행하고 창 크기를 조절하여 프레임버퍼가 창과 함께 제대로 크기가 조절되는지 확인해보세요.

## 창 최소화 처리하기

스왑 체인이 오래된 상태(out of date)가 될 수 있는 또 다른 경우는 특별한 종류의 창 크기 변경인 창 최소화입니다. 이 경우는 프레임버퍼 크기가 `0`이 되기 때문에 특별합니다. 이 튜토리얼에서는 `recreateSwapChain` 함수를 확장하여 창이 다시 포그라운드에 올 때까지 일시 중지하는 방식으로 처리하겠습니다:

```c++
void recreateSwapChain() {
    int width = 0, height = 0;
    glfwGetFramebufferSize(window, &width, &height);
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    ...
}
```

`glfwGetFramebufferSize`에 대한 초기 호출은 크기가 이미 올바르고 `glfwWaitEvents`가 기다릴 것이 없는 경우를 처리합니다.

축하합니다, 여러분은 이제 처음으로 잘 동작하는 Vulkan 프로그램을 완성했습니다! 다음 장에서는 정점 셰이더에 하드코딩된 정점들을 제거하고 실제로 정점 버퍼를 사용할 것입니다.

[C++ 코드](/code/17_swap_chain_recreation.cpp) /
[정점 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)