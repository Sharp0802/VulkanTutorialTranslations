Vulkan에서는 그리기 작업이나 메모리 전송과 같은 커맨드를 함수 호출로 직접 실행하지 않습니다. 대신, 수행하려는 모든 작업을 커맨드 버퍼 객체에 기록해야 합니다. 이렇게 하는 것의 장점은, Vulkan에게 우리가 원하는 작업을 알려줄 준비가 되었을 때 모든 커맨드가 함께 제출된다는 것입니다. Vulkan은 모든 커맨드를 한 번에 사용할 수 있으므로 더 효율적으로 처리할 수 있습니다. 또한, 원한다면 여러 스레드에서 커맨드 기록을 병렬로 수행할 수도 있습니다.

## 커맨드 풀

커맨드 버퍼를 생성하기 전에 먼저 커맨드 풀을 만들어야 합니다. 커맨드 풀은 버퍼를 저장하는 데 사용되는 메모리를 관리하며, 커맨드 버퍼는 커맨드 풀에서 할당됩니다. `VkCommandPool`을 저장할 새 클래스 멤버를 추가합니다:

```c++
VkCommandPool commandPool;
```

그런 다음 `createCommandPool`이라는 새 함수를 만들고, 프레임버퍼가 생성된 후 `initVulkan`에서 호출합니다.

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
}

...

void createCommandPool() {

}
```

커맨드 풀 생성에는 두 개의 매개변수만 필요합니다:

```c++
QueueFamilyIndices queueFamilyIndices = findQueueFamilies(physicalDevice);

VkCommandPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
poolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;
poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value();
```

커맨드 풀에는 두 가지 가능한 플래그가 있습니다:

* `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT`: 커맨드 버퍼가 매우 자주 새로운 커맨드로 다시 기록될 것임을 암시합니다 (메모리 할당 동작이 변경될 수 있습니다).
* `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT`: 커맨드 버퍼를 개별적으로 다시 기록할 수 있도록 허용합니다. 이 플래그가 없으면 모든 커맨드 버퍼를 함께 리셋해야 합니다.

우리는 매 프레임마다 커맨드 버퍼를 기록할 것이므로, 커맨드 버퍼를 리셋하고 다시 기록할 수 있어야 합니다. 따라서 커맨드 풀에 `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT` 플래그 비트를 설정해야 합니다.

커맨드 버퍼는 우리가 가져왔던 그래픽스 및 프레젠테이션 큐와 같은 장치 큐 중 하나에 제출하여 실행됩니다. 각 커맨드 풀은 단일 유형의 큐에 제출되는 커맨드 버퍼만 할당할 수 있습니다. 우리는 그리기를 위한 커맨드를 기록할 것이므로, 그래픽스 큐 패밀리를 선택했습니다.


```c++
if (vkCreateCommandPool(device, &poolInfo, nullptr, &commandPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create command pool!");
}
```

`vkCreateCommandPool` 함수를 사용하여 커맨드 풀 생성을 마칩니다. 이 함수에는 특별한 매개변수가 없습니다. 커맨드는 프로그램 전반에 걸쳐 화면에 무언가를 그리는 데 사용되므로, 풀은 프로그램이 끝날 때만 파괴되어야 합니다:

```c++
void cleanup() {
    vkDestroyCommandPool(device, commandPool, nullptr);

    ...
}
```

## 커맨드 버퍼 할당

이제 커맨드 버퍼 할당을 시작할 수 있습니다.

클래스 멤버로 `VkCommandBuffer` 객체를 생성합니다. 커맨드 버퍼는 커맨드 풀이 파괴될 때 자동으로 해제되므로, 명시적인 정리(cleanup) 코드가 필요하지 않습니다.

```c++
VkCommandBuffer commandBuffer;
```

이제 커맨드 풀에서 단일 커맨드 버퍼를 할당하는 `createCommandBuffer` 함수 작업을 시작하겠습니다.

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
}

...

void createCommandBuffer() {

}
```

커맨드 버퍼는 `vkAllocateCommandBuffers` 함수로 할당되며, 이 함수는 커맨드 풀과 할당할 버퍼 수를 지정하는 `VkCommandBufferAllocateInfo` 구조체를 매개변수로 받습니다:

```c++
VkCommandBufferAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
allocInfo.commandPool = commandPool;
allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
allocInfo.commandBufferCount = 1;

if (vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate command buffers!");
}
```

`level` 매개변수는 할당된 커맨드 버퍼가 주(primary) 커맨드 버퍼인지 보조(secondary) 커맨드 버퍼인지를 지정합니다.

* `VK_COMMAND_BUFFER_LEVEL_PRIMARY`: 실행을 위해 큐에 제출될 수 있지만, 다른 커맨드 버퍼에서 호출될 수는 없습니다.
* `VK_COMMAND_BUFFER_LEVEL_SECONDARY`: 직접 제출될 수는 없지만, 주 커맨드 버퍼에서 호출될 수 있습니다.

여기서는 보조 커맨드 버퍼 기능을 사용하지 않겠지만, 주 커맨드 버퍼에서 공통 작업을 재사용하는 데 유용하다는 것을 상상할 수 있습니다.

우리는 하나의 커맨드 버퍼만 할당하므로, `commandBufferCount` 매개변수는 1입니다.

## 커맨드 버퍼 기록

이제 실행하려는 커맨드를 커맨드 버퍼에 작성하는 `recordCommandBuffer` 함수 작업을 시작하겠습니다. 사용될 `VkCommandBuffer`와 현재 렌더링할 스왑체인 이미지의 인덱스가 매개변수로 전달될 것입니다.

```c++
void recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex) {

}
```

커맨드 버퍼 기록은 항상 `vkBeginCommandBuffer`를 호출하는 것으로 시작합니다. 이 함수는 해당 커맨드 버퍼의 사용법에 대한 세부 정보를 지정하는 작은 `VkCommandBufferBeginInfo` 구조체를 인자로 받습니다.

```c++
VkCommandBufferBeginInfo beginInfo{};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
beginInfo.flags = 0; // Optional
beginInfo.pInheritanceInfo = nullptr; // Optional

if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS) {
    throw std::runtime_error("failed to begin recording command buffer!");
}
```

`flags` 매개변수는 커맨드 버퍼를 어떻게 사용할 것인지 지정합니다. 사용 가능한 값은 다음과 같습니다:

* `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`: 커맨드 버퍼가 한 번 실행된 직후 다시 기록될 것임을 의미합니다.
* `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT`: 이 커맨드 버퍼가 단일 렌더 패스 내에서만 사용되는 보조 커맨드 버퍼임을 의미합니다.
* `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`: 커맨드 버퍼가 이미 실행 대기 중인 상태에서도 다시 제출될 수 있음을 의미합니다.

지금은 이 플래그들 중 어느 것도 우리에게 해당되지 않습니다.

`pInheritanceInfo` 매개변수는 보조 커맨드 버퍼에만 관련이 있습니다. 이 매개변수는 호출하는 주 커맨드 버퍼로부터 어떤 상태를 상속받을지 지정합니다.

만약 커맨드 버퍼가 이미 한 번 기록되었다면, `vkBeginCommandBuffer`를 호출하는 것은 암묵적으로 해당 버퍼를 리셋합니다. 나중에 버퍼에 커맨드를 추가하는 것은 불가능합니다.

## 렌더 패스 시작하기

그리기는 `vkCmdBeginRenderPass`로 렌더 패스를 시작하는 것으로부터 시작됩니다. 렌더 패스는 `VkRenderPassBeginInfo` 구조체의 몇몇 매개변수를 사용하여 구성됩니다.

```c++
VkRenderPassBeginInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
renderPassInfo.renderPass = renderPass;
renderPassInfo.framebuffer = swapChainFramebuffers[imageIndex];
```

첫 번째 매개변수들은 렌더 패스 자체와 바인딩할 어태치먼트입니다. 우리는 각 스왑 체인 이미지에 대해 프레임버퍼를 생성했으며, 각 이미지는 색상 어태치먼트로 지정되었습니다. 따라서 우리가 그리고자 하는 스왑체인 이미지에 대한 프레임버퍼를 바인딩해야 합니다. 전달된 `imageIndex` 매개변수를 사용하여 현재 스왑체인 이미지에 맞는 프레임버퍼를 선택할 수 있습니다.

```c++
renderPassInfo.renderArea.offset = {0, 0};
renderPassInfo.renderArea.extent = swapChainExtent;
```

다음 두 매개변수는 렌더 영역의 크기를 정의합니다. 렌더 영역은 셰이더 로드 및 저장이 일어날 위치를 정의합니다. 이 영역 밖의 픽셀들은 정의되지 않은 값을 갖게 됩니다. 최상의 성능을 위해서는 어태치먼트의 크기와 일치해야 합니다.

```c++
VkClearValue clearColor = {{{0.0f, 0.0f, 0.0f, 1.0f}}};
renderPassInfo.clearValueCount = 1;
renderPassInfo.pClearValues = &clearColor;
```

마지막 두 매개변수는 `VK_ATTACHMENT_LOAD_OP_CLEAR`에 사용할 클리어(clear) 값을 정의합니다. 우리는 이 값을 색상 어태치먼트의 로드 작업으로 사용했습니다. 저는 클리어 색상을 100% 불투명도의 검은색으로 정의했습니다.

```c++
vkCmdBeginRenderPass(commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
```

이제 렌더 패스를 시작할 수 있습니다. 커맨드를 기록하는 모든 함수는 `vkCmd` 접두사로 식별할 수 있습니다. 이 함수들은 모두 `void`를 반환하므로, 기록을 마칠 때까지는 오류 처리가 없습니다.

모든 커맨드의 첫 번째 매개변수는 항상 커맨드를 기록할 커맨드 버퍼입니다. 두 번째 매개변수는 우리가 방금 제공한 렌더 패스의 세부 정보를 지정합니다. 마지막 매개변수는 렌더 패스 내의 그리기 커맨드가 어떻게 제공될지를 제어합니다. 이 매개변수는 다음 두 값 중 하나를 가질 수 있습니다:

* `VK_SUBPASS_CONTENTS_INLINE`: 렌더 패스 커맨드들이 주 커맨드 버퍼 자체에 포함되며, 보조 커맨드 버퍼는 실행되지 않습니다.
* `VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS`: 렌더 패스 커맨드들이 보조 커맨드 버퍼로부터 실행됩니다.

우리는 보조 커맨드 버퍼를 사용하지 않을 것이므로, 첫 번째 옵션을 선택하겠습니다.

## 기본 그리기 커맨드

이제 그래픽스 파이프라인을 바인딩할 수 있습니다:

```c++
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);
```

두 번째 매개변수는 파이프라인 객체가 그래픽스 파이프라인인지 컴퓨트 파이프라인인지를 지정합니다. 이로써 우리는 Vulkan에게 그래픽스 파이프라인에서 어떤 작업을 실행할지, 그리고 프래그먼트 셰이더에서 어떤 어태치먼트를 사용할지를 알려주었습니다.

[고정 함수 챕터](../02_Graphics_pipeline_basics/02_Fixed_functions.md#dynamic-state)에서 언급했듯이, 우리는 이 파이프라인의 뷰포트와 시저 상태를 동적(dynamic)으로 지정했습니다. 따라서 그리기 커맨드를 호출하기 전에 커맨드 버퍼에서 이 값들을 설정해야 합니다:

```c++
VkViewport viewport{};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = static_cast<float>(swapChainExtent.width);
viewport.height = static_cast<float>(swapChainExtent.height);
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
vkCmdSetViewport(commandBuffer, 0, 1, &viewport);

VkRect2D scissor{};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;
vkCmdSetScissor(commandBuffer, 0, 1, &scissor);
```

이제 삼각형에 대한 그리기 커맨드를 호출할 준비가 되었습니다:

```c++
vkCmdDraw(commandBuffer, 3, 1, 0, 0);
```

실제 `vkCmdDraw` 함수는 다소 허무하게 보일 수 있지만, 사전에 모든 정보를 지정했기 때문에 이렇게 간단합니다. 이 함수는 커맨드 버퍼 외에 다음과 같은 매개변수들을 가집니다:

* `vertexCount`: 정점 버퍼가 없더라도, 기술적으로는 그려야 할 3개의 정점이 있습니다.
* `instanceCount`: 인스턴스 렌더링에 사용됩니다. 사용하지 않을 경우 `1`로 설정합니다.
* `firstVertex`: 정점 버퍼로의 오프셋으로 사용되며, `gl_VertexIndex`의 최솟값을 정의합니다.
* `firstInstance`: 인스턴스 렌더링을 위한 오프셋으로 사용되며, `gl_InstanceIndex`의 최솟값을 정의합니다.

## 마무리

이제 렌더 패스를 종료할 수 있습니다:

```c++
vkCmdEndRenderPass(commandBuffer);
```

그리고 커맨드 버퍼 기록을 마칩니다:

```c++
if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS) {
    throw std::runtime_error("failed to record command buffer!");
}
```



다음 챕터에서는 메인 루프 코드를 작성할 것입니다. 메인 루프는 스왑 체인에서 이미지를 가져오고, 커맨드 버퍼를 기록 및 실행한 다음, 완성된 이미지를 다시 스왑 체인으로 반환하는 역할을 합니다.

[C++ 코드](/code/14_command_buffers.cpp) /
[정점 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)