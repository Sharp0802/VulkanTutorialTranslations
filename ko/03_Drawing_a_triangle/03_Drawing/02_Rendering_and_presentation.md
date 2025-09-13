이번 챕터에서는 모든 것을 하나로 합칠 것입니다. 우리는 메인 루프에서 호출되어 삼각형을 화면에 표시할 `drawFrame` 함수를 작성할 것입니다. 먼저 함수를 만들고 `mainLoop`에서 호출해 봅시다:

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }
}

...

void drawFrame() {

}
```

## 프레임의 개요

개괄적으로 Vulkan에서 프레임을 렌더링하는 것은 다음과 같은 공통된 단계들로 구성됩니다:

*   이전 프레임이 끝날 때까지 기다립니다
*   스왑 체인에서 이미지를 가져옵니다
*   해당 이미지에 장면을 그리는 커맨드 버퍼를 기록합니다
*   기록된 커맨드 버퍼를 제출합니다
*   스왑 체인 이미지를 화면에 표시(present)합니다

이후 챕터에서 그리기 함수를 확장하겠지만, 지금은 이것이 우리 렌더 루프의 핵심입니다.

<!-- 프레임의 개요를 보여주는 이미지 추가 -->

## 동기화

<!-- 동기화를 보여주는 이미지 추가 -->

Vulkan의 핵심 설계 철학 중 하나는 GPU에서의 실행 동기화가 명시적이라는 것입니다. 작업 순서는 우리가 다양한 동기화 프리미티브를 사용하여 정의해야 하며, 이 프리미티브는 드라이버에게 우리가 원하는 실행 순서를 알려줍니다. 이는 GPU에서 작업을 실행하기 시작하는 많은 Vulkan API 호출이 비동기적이라는 것을 의미합니다. 즉, 함수는 작업이 완료되기 전에 반환됩니다.

이번 챕터에서는 GPU에서 발생하기 때문에 명시적으로 순서를 정해야 하는 여러 이벤트가 있습니다. 예를 들면 다음과 같습니다:

*   스왑 체인에서 이미지를 가져오기
*   가져온 이미지에 그리는 커맨드를 실행하기
*   해당 이미지를 화면에 표시(presentation)하고 스왑 체인에 반환하기

이러한 각 이벤트는 단일 함수 호출로 시작되지만 모두 비동기적으로 실행됩니다. 함수 호출은 실제 작업이 끝나기 전에 반환되며 실행 순서 또한 정의되지 않습니다. 이는 안타까운 일입니다. 왜냐하면 각 작업은 이전 작업이 완료되는 것에 의존하기 때문입니다. 따라서 원하는 순서를 달성하기 위해 어떤 프리미티브를 사용할 수 있는지 알아봐야 합니다.

### 세마포어(Semaphores)

세마포어는 큐 작업 간에 순서를 추가하는 데 사용됩니다. 큐 작업이란 커맨드 버퍼에서든, 나중에 보게 될 함수 내에서든 우리가 큐에 제출하는 작업을 의미합니다. 큐의 예로는 그래픽스 큐와 프레젠테이션 큐가 있습니다. 세마포어는 동일한 큐 내의 작업 순서를 정하는 데에도, 서로 다른 큐 간의 작업 순서를 정하는 데에도 사용됩니다.

Vulkan에는 바이너리(binary)와 타임라인(timeline) 두 종류의 세마포어가 있습니다. 이 튜토리얼에서는 바이너리 세마포어만 사용할 것이므로 타임라인 세마포어는 다루지 않겠습니다. 앞으로 '세마포어'라는 용어는 바이너리 세마포어만을 지칭합니다.

세마포어는 시그널되지 않았거나(unsignaled) 시그널된(signaled) 상태 중 하나입니다. 처음에는 시그널되지 않은 상태로 시작합니다. 큐 작업의 순서를 정하기 위해 세마포어를 사용하는 방법은, 한 큐 작업에서는 '시그널' 세마포어로, 다른 큐 작업에서는 '대기' 세마포어로 동일한 세마포어를 제공하는 것입니다. 예를 들어, 순서대로 실행하고 싶은 세마포어 S와 큐 작업 A, B가 있다고 가정해 봅시다. 우리가 Vulkan에 알려주는 것은, 작업 A가 실행을 마치면 세마포어 S에 '시그널'을 보내고, 작업 B는 실행을 시작하기 전에 세마포어 S를 '대기'하라는 것입니다. 작업 A가 끝나면 세마포어 S는 시그널되고, 작업 B는 S가 시그널될 때까지 시작되지 않습니다. 작업 B가 실행을 시작하면, 세마포어 S는 자동으로 시그널되지 않은 상태로 재설정되어 다시 사용할 수 있게 됩니다.

방금 설명한 내용의 의사 코드(pseudo-code)입니다:
```
VkCommandBuffer A, B = ... // 커맨드 버퍼 기록
VkSemaphore S = ... // 세마포어 생성

// A를 큐에 넣고, 끝나면 S에 시그널 - 즉시 실행 시작
vkQueueSubmit(work: A, signal: S, wait: None)

// B를 큐에 넣고, 시작하기 전에 S를 기다림
vkQueueSubmit(work: B, signal: None, wait: S)
```

이 코드 조각에서 `vkQueueSubmit()` 호출은 둘 다 즉시 반환됩니다. 대기는 GPU에서만 발생합니다. CPU는 블록되지 않고 계속 실행됩니다. CPU를 대기시키려면 다른 동기화 프리미티브가 필요한데, 이제 그것을 설명하겠습니다.

### 펜스(Fences)

펜스는 실행을 동기화하는 데 사용된다는 점에서 비슷한 목적을 가지지만, CPU, 즉 호스트(host)에서의 실행 순서를 정하기 위한 것입니다. 간단히 말해, 호스트가 GPU가 무언가를 마쳤는지 알아야 할 때 펜스를 사용합니다.

세마포어와 마찬가지로 펜스도 시그널되거나 시그널되지 않은 상태 중 하나입니다. 실행할 작업을 제출할 때마다 해당 작업에 펜스를 붙일 수 있습니다. 작업이 완료되면 펜스가 시그널됩니다. 그러면 우리는 호스트가 펜스가 시그널될 때까지 기다리게 할 수 있으며, 이를 통해 호스트가 계속 진행하기 전에 작업이 완료되었음을 보장할 수 있습니다.

구체적인 예로 스크린샷 찍기를 들 수 있습니다. GPU에서 필요한 작업을 이미 마쳤다고 가정합시다. 이제 이미지를 GPU에서 호스트로 전송한 다음 메모리를 파일에 저장해야 합니다. 전송을 실행하는 커맨드 버퍼 A와 펜스 F가 있습니다. 우리는 펜스 F와 함께 커맨드 버퍼 A를 제출한 다음, 즉시 호스트에게 F가 시그널될 때까지 기다리라고 지시합니다. 이로 인해 호스트는 커맨드 버퍼 A의 실행이 끝날 때까지 블록됩니다. 따라서 메모리 전송이 완료되었으므로 호스트가 파일을 디스크에 안전하게 저장할 수 있습니다.

설명한 내용의 의사 코드입니다:
```
VkCommandBuffer A = ... // 전송이 포함된 커맨드 버퍼 기록
VkFence F = ... // 펜스 생성

// A를 큐에 넣고, 즉시 작업 시작, 끝나면 F에 시그널
vkQueueSubmit(work: A, fence: F)

vkWaitForFence(F) // A의 실행이 끝날 때까지 실행을 블록함

save_screenshot_to_disk() // 전송이 끝나기 전까지 실행할 수 없음
```

세마포어 예제와 달리, 이 예제는 호스트 실행을 *블록*합니다. 이는 호스트가 실행이 끝날 때까지 기다리는 것 외에는 아무것도 하지 않는다는 것을 의미합니다. 이 경우, 스크린샷을 디스크에 저장하기 전에 전송이 완료되었는지 확인해야 했습니다.

일반적으로, 필요하지 않은 한 호스트를 블록하지 않는 것이 좋습니다. 우리는 GPU와 호스트에 유용한 작업을 계속 제공하고 싶습니다. 펜스가 시그널되기를 기다리는 것은 유용한 작업이 아닙니다. 따라서 작업을 동기화하기 위해 세마포어나 아직 다루지 않은 다른 동기화 프리미티브를 선호합니다.

펜스는 수동으로 재설정하여 시그널되지 않은 상태로 되돌려야 합니다. 이는 펜스가 호스트의 실행을 제어하는 데 사용되므로, 호스트가 펜스를 언제 재설정할지 결정하기 때문입니다. 이는 호스트의 개입 없이 GPU의 작업 순서를 정하는 데 사용되는 세마포어와 대조됩니다.

요약하자면, 세마포어는 GPU에서의 작업 실행 순서를 지정하는 데 사용되고, 펜스는 CPU와 GPU를 서로 동기화 상태로 유지하는 데 사용됩니다.

### 무엇을 선택해야 할까요?

우리는 사용할 두 개의 동기화 프리미티브와 동기화를 적용할 두 곳이 있습니다: 스왑 체인 작업과 이전 프레임이 끝나기를 기다리는 것입니다. 스왑 체인 작업은 GPU에서 발생하므로 가능하다면 호스트가 기다리게 하고 싶지 않기 때문에 세마포어를 사용하고 싶습니다. 이전 프레임이 끝나기를 기다리는 것에는 반대 이유로 펜스를 사용하고 싶습니다. 왜냐하면 호스트가 기다려야 하기 때문입니다. 이는 한 번에 한 프레임 이상을 그리지 않도록 하기 위함입니다. 우리는 매 프레임마다 커맨드 버퍼를 다시 기록하기 때문에, 현재 프레임의 실행이 끝나기 전까지는 다음 프레임의 작업을 커맨드 버퍼에 기록할 수 없습니다. GPU가 사용 중인 커맨드 버퍼의 현재 내용을 덮어쓰고 싶지 않기 때문입니다.

## 동기화 객체 생성하기

이미지가 스왑 체인에서 획득되어 렌더링 준비가 되었음을 알리는 세마포어 하나, 렌더링이 끝나 프레젠테이션이 가능함을 알리는 세마포어 하나, 그리고 한 번에 한 프레임만 렌더링되도록 보장하는 펜스 하나가 필요합니다.

이 세마포어 객체들과 펜스 객체를 저장하기 위해 세 개의 클래스 멤버를 만듭니다:

```c++
VkSemaphore imageAvailableSemaphore;
VkSemaphore renderFinishedSemaphore;
VkFence inFlightFence;
```

세마포어를 생성하기 위해, 이 튜토리얼의 이 부분에 대한 마지막 `create` 함수인 `createSyncObjects`를 추가할 것입니다:

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandPool();
    createCommandBuffer();
    createSyncObjects();
}

...

void createSyncObjects() {

}
```

세마포어를 생성하려면 `VkSemaphoreCreateInfo`를 채워야 하지만, 현재 API 버전에서는 `sType` 외에는 실제로 필수 필드가 없습니다:

```c++
void createSyncObjects() {
    VkSemaphoreCreateInfo semaphoreInfo{};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAphore_CREATE_INFO;
}
```

향후 Vulkan API 버전이나 확장 기능에서 다른 구조체들처럼 `flags`와 `pNext` 파라미터에 기능이 추가될 수 있습니다.

펜스를 생성하려면 `VkFenceCreateInfo`를 채워야 합니다:

```c++
VkFenceCreateInfo fenceInfo{};
fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
```

세마포어와 펜스 생성은 `vkCreateSemaphore`와 `vkCreateFence`를 사용하는 익숙한 패턴을 따릅니다:

```c++
if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphore) != VK_SUCCESS ||
    vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphore) != VK_SUCCESS ||
    vkCreateFence(device, &fenceInfo, nullptr, &inFlightFence) != VK_SUCCESS) {
    throw std::runtime_error("failed to create semaphores!");
}
```

세마포어와 펜스는 프로그램이 끝날 때, 모든 커맨드가 완료되고 더 이상 동기화가 필요하지 않을 때 정리되어야 합니다:

```c++
void cleanup() {
    vkDestroySemaphore(device, imageAvailableSemaphore, nullptr);
    vkDestroySemaphore(device, renderFinishedSemaphore, nullptr);
    vkDestroyFence(device, inFlightFence, nullptr);
```

이제 메인 그리기 함수로 넘어갑시다!

## 이전 프레임 기다리기

프레임의 시작에서, 우리는 이전 프레임이 완료될 때까지 기다려서 커맨드 버퍼와 세마포어를 사용할 수 있도록 해야 합니다. 이를 위해 `vkWaitForFences`를 호출합니다:

```c++
void drawFrame() {
    vkWaitForFences(device, 1, &inFlightFence, VK_TRUE, UINT64_MAX);
}
```

`vkWaitForFences` 함수는 펜스 배열을 받아 호스트에서 하나 또는 모든 펜스가 시그널될 때까지 기다린 후 반환합니다. 여기서 전달하는 `VK_TRUE`는 모든 펜스를 기다리겠다는 것을 나타내지만, 펜스가 하나일 경우에는 상관없습니다. 이 함수는 또한 타임아웃 파라미터를 가지는데, 64비트 부호 없는 정수의 최댓값인 `UINT64_MAX`로 설정하여 사실상 타임아웃을 비활성화합니다.

기다린 후에는 `vkResetFences` 호출로 펜스를 수동으로 시그널되지 않은 상태로 재설정해야 합니다:
```c++
    vkResetFences(device, 1, &inFlightFence);
```

진행하기 전에 우리 설계에 약간의 문제가 있습니다. 첫 프레임에서 `drawFrame()`을 호출하면 즉시 `inFlightFence`가 시그널되기를 기다립니다. `inFlightFence`는 프레임 렌더링이 완료된 후에만 시그널되는데, 지금은 첫 프레임이므로 펜스를 시그널할 이전 프레임이 없습니다! 따라서 `vkWaitForFences()`는 결코 일어나지 않을 일을 기다리며 무기한 블록됩니다.

이 딜레마에 대한 많은 해결책 중, API에 내장된 영리한 해결 방법이 있습니다. 펜스를 시그널된 상태로 생성하여, 첫 번째 `vkWaitForFences()` 호출이 펜스가 이미 시그널되었기 때문에 즉시 반환되도록 하는 것입니다.

이를 위해 `VkFenceCreateInfo`에 `VK_FENCE_CREATE_SIGNALED_BIT` 플래그를 추가합니다:

```c++
void createSyncObjects() {
    ...

    VkFenceCreateInfo fenceInfo{};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
    fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

    ...
}
```

## 스왑 체인에서 이미지 획득하기

`drawFrame` 함수에서 다음에 해야 할 일은 스왑 체인에서 이미지를 획득하는 것입니다. 스왑 체인은 확장 기능이라는 것을 기억하세요. 따라서 `vk*KHR` 명명 규칙을 가진 함수를 사용해야 합니다:

```c++
void drawFrame() {
    ...

    uint32_t imageIndex;
    vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphore, VK_NULL_HANDLE, &imageIndex);
}
```

`vkAcquireNextImageKHR`의 처음 두 파라미터는 논리 장치와 이미지를 획득하려는 스왑 체인입니다. 세 번째 파라미터는 이미지를 사용할 수 있게 될 때까지의 타임아웃을 나노초 단위로 지정합니다. 64비트 부호 없는 정수의 최댓값을 사용하면 사실상 타임아웃을 비활성화하는 것입니다.

다음 두 파라미터는 프레젠테이션 엔진이 이미지 사용을 마쳤을 때 시그널될 동기화 객체를 지정합니다. 그 시점이 우리가 이미지에 그리기를 시작할 수 있는 때입니다. 세마포어, 펜스 또는 둘 다 지정할 수 있습니다. 여기서는 `imageAvailableSemaphore`를 그 목적으로 사용할 것입니다.

마지막 파라미터는 사용 가능해진 스왑 체인 이미지의 인덱스를 출력할 변수를 지정합니다. 이 인덱스는 우리의 `swapChainImages` 배열에 있는 `VkImage`를 가리킵니다. 우리는 이 인덱스를 사용하여 `VkFrameBuffer`를 선택할 것입니다.

## 커맨드 버퍼 기록하기

사용할 스왑 체인 이미지를 지정하는 imageIndex를 손에 넣었으니, 이제 커맨드 버퍼를 기록할 수 있습니다. 먼저, 커맨드 버퍼가 기록될 수 있는 상태인지 확인하기 위해 `vkResetCommandBuffer`를 호출합니다.

```c++
vkResetCommandBuffer(commandBuffer, 0);
```

`vkResetCommandBuffer`의 두 번째 파라미터는 `VkCommandBufferResetFlagBits` 플래그입니다. 특별한 작업을 할 것이 아니므로 0으로 둡니다.

이제 `recordCommandBuffer` 함수를 호출하여 원하는 커맨드를 기록합니다.

```c++
recordCommandBuffer(commandBuffer, imageIndex);
```

완전히 기록된 커맨드 버퍼가 있으니 이제 제출할 수 있습니다.

## 커맨드 버퍼 제출하기

큐 제출 및 동기화는 `VkSubmitInfo` 구조체의 파라미터를 통해 구성됩니다.

```c++
VkSubmitInfo submitInfo{};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

VkSemaphore waitSemaphores[] = {imageAvailableSemaphore};
VkPipelineStageFlags waitStages[] = {VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT};
submitInfo.waitSemaphoreCount = 1;
submitInfo.pWaitSemaphores = waitSemaphores;
submitInfo.pWaitDstStageMask = waitStages;
```

처음 세 파라미터는 실행이 시작되기 전에 기다릴 세마포어와 파이프라인의 어느 단계에서 기다릴지를 지정합니다. 우리는 이미지를 사용할 수 있을 때까지 이미지에 색상을 쓰는 것을 기다리고 싶으므로, 색상 첨부파일에 쓰는 그래픽스 파이프라인 단계를 지정합니다. 이는 이론적으로 구현이 이미지가 아직 사용 가능하지 않은 동안에도 우리의 정점 셰이더 등을 이미 실행 시작할 수 있음을 의미합니다. `waitStages` 배열의 각 항목은 `pWaitSemaphores`에서 동일한 인덱스를 가진 세마포어에 해당합니다.

```c++
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;
```

다음 두 파라미터는 실제로 실행을 위해 제출할 커맨드 버퍼를 지정합니다. 우리는 가지고 있는 단일 커맨드 버퍼를 간단히 제출합니다.

```c++
VkSemaphore signalSemaphores[] = {renderFinishedSemaphore};
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = signalSemaphores;
```

`signalSemaphoreCount`와 `pSignalSemaphores` 파라미터는 커맨드 버퍼(들)의 실행이 완료되면 시그널할 세마포어를 지정합니다. 우리 경우에는 `renderFinishedSemaphore`를 그 목적으로 사용합니다.

```c++
if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFence) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

이제 `vkQueueSubmit`을 사용하여 그래픽스 큐에 커맨드 버퍼를 제출할 수 있습니다. 이 함수는 작업량이 훨씬 클 때 효율성을 위해 `VkSubmitInfo` 구조체 배열을 인수로 받습니다. 마지막 파라미터는 커맨드 버퍼 실행이 완료될 때 시그널될 선택적 펜스를 참조합니다. 이를 통해 커맨드 버퍼를 재사용해도 안전한 시점을 알 수 있으므로 `inFlightFence`를 전달하고 싶습니다. 이제 다음 프레임에서 CPU는 이 커맨드 버퍼의 실행이 완료될 때까지 기다린 후 새로운 커맨드를 기록할 것입니다.

## 서브패스 종속성

렌더 패스의 서브패스들은 이미지 레이아웃 전환을 자동으로 처리한다는 것을 기억하세요. 이러한 전환은 서브패스 간의 메모리 및 실행 종속성을 지정하는 *서브패스 종속성(subpass dependencies)*에 의해 제어됩니다. 지금은 단 하나의 서브패스만 있지만, 이 서브패스 바로 전후의 작업들도 암시적인 "서브패스"로 간주됩니다.

렌더 패스 시작과 끝에서 전환을 처리하는 두 개의 내장 종속성이 있지만, 전자는 올바른 시점에 발생하지 않습니다. 그것은 전환이 파이프라인의 시작에서 발생한다고 가정하지만, 우리는 그 시점에 아직 이미지를 획득하지 않았습니다! 이 문제를 해결하는 두 가지 방법이 있습니다. `imageAvailableSemaphore`에 대한 `waitStages`를 `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`으로 변경하여 이미지를 사용할 수 있을 때까지 렌더 패스가 시작되지 않도록 하거나, 렌더 패스가 `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT` 단계를 기다리게 할 수 있습니다. 저는 여기서 두 번째 옵션을 선택하기로 했습니다. 왜냐하면 서브패스 종속성과 그 작동 방식을 살펴볼 좋은 기회이기 때문입니다.

서브패스 종속성은 `VkSubpassDependency` 구조체에 지정됩니다. `createRenderPass` 함수로 가서 하나를 추가하세요:

```c++
VkSubpassDependency dependency{};
dependency.srcSubpass = VK_SUBPASS_EXTERNAL;
dependency.dstSubpass = 0;
```

처음 두 필드는 종속성과 종속된 서브패스의 인덱스를 지정합니다. 특별한 값 `VK_SUBPASS_EXTERNAL`은 `srcSubpass` 또는 `dstSubpass`에 지정되었는지에 따라 렌더 패스 전후의 암시적 서브패스를 나타냅니다. 인덱스 `0`은 우리의 첫 번째이자 유일한 서브패스를 나타냅니다. `dstSubpass`는 종속성 그래프에서 순환을 방지하기 위해 항상 `srcSubpass`보다 높아야 합니다(서브패스 중 하나가 `VK_SUBPASS_EXTERNAL`이 아닌 한).

```c++
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.srcAccessMask = 0;
```

다음 두 필드는 기다릴 작업과 이러한 작업이 발생하는 단계를 지정합니다. 우리가 이미지에 접근하기 전에 스왑 체인이 이미지에서 읽기를 마칠 때까지 기다려야 합니다. 이는 색상 첨부파일 출력 단계 자체를 기다림으로써 달성할 수 있습니다.

```c++
dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
```

이것을 기다려야 하는 작업들은 색상 첨부파일 단계에 있으며 색상 첨부파일의 쓰기와 관련이 있습니다. 이러한 설정은 실제로 필요하고 허용될 때까지(즉, 우리가 색상을 쓰기 시작하고 싶을 때까지) 전환이 일어나지 않도록 막을 것입니다.

```c++
renderPassInfo.dependencyCount = 1;
renderPassInfo.pDependencies = &dependency;
```

`VkRenderPassCreateInfo` 구조체에는 종속성 배열을 지정하는 두 개의 필드가 있습니다.

## 프레젠테이션

프레임 그리기의 마지막 단계는 결과를 스왑 체인에 다시 제출하여最终적으로 화면에 표시되도록 하는 것입니다. 프레젠테이션은 `drawFrame` 함수 끝에서 `VkPresentInfoKHR` 구조체를 통해 구성됩니다.

```c++
VkPresentInfoKHR presentInfo{};
presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;

presentInfo.waitSemaphoreCount = 1;
presentInfo.pWaitSemaphores = signalSemaphores;
```

처음 두 파라미터는 `VkSubmitInfo`와 마찬가지로 프레젠테이션이 일어나기 전에 기다릴 세마포어를 지정합니다. 커맨드 버퍼의 실행이 끝나기를, 즉 삼각형이 그려지기를 기다리고 싶으므로, 시그널될 세마포어를 가져와서 그것들을 기다립니다. 따라서 `signalSemaphores`를 사용합니다.

```c++
VkSwapchainKHR swapChains[] = {swapChain};
presentInfo.swapchainCount = 1;
presentInfo.pSwapchains = swapChains;
presentInfo.pImageIndices = &imageIndex;
```

다음 두 파라미터는 이미지를 표시할 스왑 체인과 각 스왑 체인에 대한 이미지의 인덱스를 지정합니다. 이것은 거의 항상 하나일 것입니다.

```c++
presentInfo.pResults = nullptr; // Optional
```

`pResults`라는 마지막 선택적 파라미터가 하나 있습니다. 이것은 프레젠테이션이 성공했는지 모든 개별 스왑 체인에 대해 확인할 `VkResult` 값의 배열을 지정할 수 있게 해줍니다. 단일 스왑 체인만 사용하는 경우에는 필요하지 않습니다. 왜냐하면 프레젠테이션 함수의 반환 값을 간단히 사용할 수 있기 때문입니다.

```c++
vkQueuePresentKHR(presentQueue, &presentInfo);
```

`vkQueuePresentKHR` 함수는 이미지를 스왑 체인에 표시하라는 요청을 제출합니다. 다음 챕터에서 `vkAcquireNextImageKHR`와 `vkQueuePresentKHR` 모두에 대한 오류 처리를 추가할 것입니다. 왜냐하면 이들의 실패가 지금까지 본 함수들과 달리 반드시 프로그램이 종료되어야 함을 의미하지는 않기 때문입니다.

여기까지 모든 것을 올바르게 했다면, 프로그램을 실행했을 때 다음과 유사한 것을 보게 될 것입니다:

![](/images/triangle.png)

> 이 색칠된 삼각형은 그래픽스 튜토리얼에서 흔히 보던 것과 약간 다르게 보일 수 있습니다. 그 이유는 이 튜토리얼에서는 셰이더가 선형 색 공간에서 보간하고 나중에 sRGB 색 공간으로 변환하기 때문입니다. 차이점에 대한 논의는 [이 블로그 포스트](https://medium.com/@heypete/hello-triangle-meet-swift-and-wide-color-6f9e246616d9)를 참조하세요.

만세! 안타깝게도, 유효성 검사 레이어를 활성화하면 프로그램을 닫자마자 충돌하는 것을 보게 될 것입니다. `debugCallback`에서 터미널에 출력된 메시지가 그 이유를 알려줍니다:

![](/images/semaphore_in_use.png)

`drawFrame`의 모든 작업은 비동기적이라는 것을 기억하세요. 즉, `mainLoop`의 루프를 빠져나갈 때, 그리기 및 프레젠테이션 작업이 여전히 진행 중일 수 있습니다. 그런 일이 일어나는 동안 리소스를 정리하는 것은 좋은 생각이 아닙니다.

이 문제를 해결하려면 `mainLoop`를 종료하고 창을 파괴하기 전에 논리 장치가 작업을 마칠 때까지 기다려야 합니다:

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }

    vkDeviceWaitIdle(device);
}
```

`vkQueueWaitIdle`을 사용하여 특정 커맨드 큐의 작업이 완료될 때까지 기다릴 수도 있습니다. 이 함수들은 매우 초보적인 동기화 수행 방법으로 사용될 수 있습니다. 이제 창을 닫을 때 프로그램이 문제없이 종료되는 것을 볼 수 있을 것입니다.

## 결론

900줄이 조금 넘는 코드를 작성한 끝에, 마침내 화면에 무언가 나타나는 것을 볼 수 있게 되었습니다! Vulkan 프로그램을 부트스트래핑하는 것은 분명 많은 작업이지만, 핵심은 Vulkan이 명시성을 통해 엄청난 제어권을 제공한다는 것입니다. 지금 시간을 내어 코드를 다시 읽어보고, 프로그램에 있는 모든 Vulkan 객체의 목적과 서로 어떻게 관련되는지에 대한 정신적 모델을 구축하는 것을 추천합니다. 우리는 이 지식을 바탕으로 지금부터 프로그램의 기능을 확장해 나갈 것입니다.

다음 챕터에서는 여러 프레임을 동시에 처리하도록 렌더 루프를 확장할 것입니다.

[C++ 코드](/code/15_hello_triangle.cpp) /
[정점 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)