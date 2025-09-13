## 서론(Introduction)

이전 챕터의 디스크립터 셋 레이아웃은 바인딩될 수 있는 디스크립터의 유형을 설명합니다. 이번 챕터에서는 각 `VkBuffer` 리소스에 대한 디스크립터 셋을 생성하여 유니폼 버퍼 디스크립터에 바인딩할 것입니다.

## 디스크립터 풀(Descriptor pool)

디스크립터 셋은 직접 생성할 수 없으며, 커맨드 버퍼처럼 풀에서 할당되어야 합니다. 디스크립터 셋에 해당하는 것은 놀랍지 않게도 *디스크립터 풀*이라고 불립니다. 이를 설정하기 위해 새로운 함수 `createDescriptorPool`을 작성할 것입니다.

```c++
void initVulkan() {
    ...
    createUniformBuffers();
    createDescriptorPool();
    ...
}

...

void createDescriptorPool() {

}
```

먼저, `VkDescriptorPoolSize` 구조체를 사용하여 우리의 디스크립터 셋이 어떤 디스크립터 유형을 포함할 것이며, 각각 몇 개나 포함할 것인지 설명해야 합니다.

```c++
VkDescriptorPoolSize poolSize{};
poolSize.type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
poolSize.descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
```

우리는 모든 프레임에 대해 이 디스크립터 중 하나를 할당할 것입니다. 이 풀 크기 구조체는 메인 `VkDescriptorPoolCreateInfo`에 의해 참조됩니다.

```c++
VkDescriptorPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
poolInfo.poolSizeCount = 1;
poolInfo.pPoolSizes = &poolSize;
```

사용 가능한 개별 디스크립터의 최대 개수 외에도, 할당될 수 있는 디스크립터 셋의 최대 개수도 지정해야 합니다.

```c++
poolInfo.maxSets = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
```

이 구조체는 커맨드 풀과 유사하게 개별 디스크립터 셋의 해제 가능 여부를 결정하는 선택적 플래그를 가지고 있습니다: `VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT`. 우리는 디스크립터 셋을 생성한 후에는 건드리지 않을 것이므로 이 플래그는 필요 없습니다. `flags`는 기본값인 `0`으로 둘 수 있습니다.

```c++
VkDescriptorPool descriptorPool;

...

if (vkCreateDescriptorPool(device, &poolInfo, nullptr, &descriptorPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create descriptor pool!");
}
```

디스크립터 풀의 핸들을 저장하기 위해 새로운 클래스 멤버를 추가하고, `vkCreateDescriptorPool`을 호출하여 생성합니다.

## 디스크립터 셋(Descriptor set)

이제 디스크립터 셋 자체를 할당할 수 있습니다. 이를 위해 `createDescriptorSets` 함수를 추가합시다.

```c++
void initVulkan() {
    ...
    createDescriptorPool();
    createDescriptorSets();
    ...
}

...

void createDescriptorSets() {

}
```

디스크립터 셋 할당은 `VkDescriptorSetAllocateInfo` 구조체로 설명됩니다. 할당할 디스크립터 풀, 할당할 디스크립터 셋의 수, 그리고 기반으로 할 디스크립터 셋 레이아웃을 지정해야 합니다.

```c++
std::vector<VkDescriptorSetLayout> layouts(MAX_FRAMES_IN_FLIGHT, descriptorSetLayout);
VkDescriptorSetAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
allocInfo.descriptorPool = descriptorPool;
allocInfo.descriptorSetCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
allocInfo.pSetLayouts = layouts.data();
```

우리의 경우, 현재 렌더링 중인 각 프레임에 대해 하나의 디스크립터 셋을 생성하며, 모두 동일한 레이아웃을 사용합니다. 안타깝게도 다음 함수가 셋의 수와 일치하는 배열을 예상하기 때문에 레이아웃의 모든 복사본이 필요합니다.

디스크립터 셋 핸들을 담을 클래스 멤버를 추가하고 `vkAllocateDescriptorSets`로 할당합니다.

```c++
VkDescriptorPool descriptorPool;
std::vector<VkDescriptorSet> descriptorSets;

...

descriptorSets.resize(MAX_FRAMES_IN_FLIGHT);
if (vkAllocateDescriptorSets(device, &allocInfo, descriptorSets.data()) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate descriptor sets!");
}
```

디스크립터 셋은 디스크립터 풀이 파괴될 때 자동으로 해제되므로 명시적으로 정리할 필요가 없습니다. `vkAllocateDescriptorSets` 호출은 각각 하나의 유니폼 버퍼 디스크립터를 가진 디스크립터 셋들을 할당할 것입니다.

```c++
void cleanup() {
    ...
    vkDestroyDescriptorPool(device, descriptorPool, nullptr);

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);
    ...
}
```

디스크립터 셋은 이제 할당되었지만, 그 안의 디스크립터들은 여전히 구성이 필요합니다. 이제 모든 디스크립터를 채우기 위한 루프를 추가할 것입니다.

```c++
for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {

}
```

우리의 유니폼 버퍼 디스크립터처럼 버퍼를 참조하는 디스크립터는 `VkDescriptorBufferInfo` 구조체로 구성됩니다. 이 구조체는 버퍼와 디스크립터의 데이터가 포함된 버퍼 내의 영역을 지정합니다.

```c++
for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    VkDescriptorBufferInfo bufferInfo{};
    bufferInfo.buffer = uniformBuffers[i];
    bufferInfo.offset = 0;
    bufferInfo.range = sizeof(UniformBufferObject);
}
```

이 경우처럼 버퍼 전체를 덮어쓴다면, range에 `VK_WHOLE_SIZE` 값을 사용하는 것도 가능합니다. 디스크립터의 구성은 `vkUpdateDescriptorSets` 함수를 사용하여 업데이트되며, 이 함수는 `VkWriteDescriptorSet` 구조체의 배열을 매개변수로 받습니다.

```c++
VkWriteDescriptorSet descriptorWrite{};
descriptorWrite.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrite.dstSet = descriptorSets[i];
descriptorWrite.dstBinding = 0;
descriptorWrite.dstArrayElement = 0;
```

처음 두 필드는 업데이트할 디스크립터 셋과 바인딩을 지정합니다. 우리는 유니폼 버퍼 바인딩에 인덱스 `0`을 부여했습니다. 디스크립터는 배열이 될 수 있다는 점을 기억하세요. 따라서 업데이트하려는 배열의 첫 번째 인덱스도 지정해야 합니다. 우리는 배열을 사용하지 않으므로 인덱스는 그냥 `0`입니다.

```c++
descriptorWrite.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorWrite.descriptorCount = 1;
```

디스크립터의 유형을 다시 지정해야 합니다. 인덱스 `dstArrayElement`에서 시작하여 배열 내의 여러 디스크립터를 한 번에 업데이트하는 것이 가능합니다. `descriptorCount` 필드는 업데이트하려는 배열 요소의 수를 지정합니다.

```c++
descriptorWrite.pBufferInfo = &bufferInfo;
descriptorWrite.pImageInfo = nullptr; // Optional
descriptorWrite.pTexelBufferView = nullptr; // Optional
```

마지막 필드는 실제로 디스크립터를 구성하는 `descriptorCount` 개의 구조체를 가진 배열을 참조합니다. 세 가지 중 어떤 것을 사용해야 하는지는 디스크립터의 유형에 따라 다릅니다. `pBufferInfo` 필드는 버퍼 데이터를 참조하는 디스크립터에 사용되고, `pImageInfo`는 이미지 데이터를 참조하는 디스크립터에, `pTexelBufferView`는 버퍼 뷰를 참조하는 디스크립터에 사용됩니다. 우리의 디스크립터는 버퍼 기반이므로 `pBufferInfo`를 사용합니다.

```c++
vkUpdateDescriptorSets(device, 1, &descriptorWrite, 0, nullptr);
```

업데이트는 `vkUpdateDescriptorSets`를 사용하여 적용됩니다. 이 함수는 두 종류의 배열을 매개변수로 받습니다: `VkWriteDescriptorSet` 배열과 `VkCopyDescriptorSet` 배열입니다. 후자는 이름에서 알 수 있듯이 디스크립터를 서로 복사하는 데 사용할 수 있습니다.

## 디스크립터 셋 사용하기(Using descriptor sets)

이제 `recordCommandBuffer` 함수를 업데이트하여 각 프레임에 맞는 디스크립터 셋을 셰이더의 디스크립터에 `vkCmdBindDescriptorSets`로 실제로 바인딩해야 합니다. 이는 `vkCmdDrawIndexed` 호출 전에 수행되어야 합니다.

```c++
vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, 1, &descriptorSets[currentFrame], 0, nullptr);
vkCmdDrawIndexed(commandBuffer, static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

정점 및 인덱스 버퍼와 달리, 디스크립터 셋은 그래픽스 파이프라인에만 국한되지 않습니다. 따라서 디스크립터 셋을 그래픽스 파이프라인에 바인딩할지, 컴퓨트 파이프라인에 바인딩할지 지정해야 합니다. 다음 매개변수는 디스크립터가 기반으로 하는 레이아웃입니다. 다음 세 개의 매개변수는 첫 번째 디스크립터 셋의 인덱스, 바인딩할 셋의 수, 그리고 바인딩할 셋의 배열을 지정합니다. 이 부분은 잠시 후에 다시 살펴보겠습니다. 마지막 두 매개변수는 동적 디스크립터에 사용되는 오프셋 배열을 지정합니다. 이는 나중 챕터에서 다룰 것입니다.

지금 프로그램을 실행하면 안타깝게도 아무것도 보이지 않는 것을 알 수 있습니다. 문제는 투영 행렬에서 수행한 Y-축 뒤집기 때문에, 정점들이 이제 시계 방향 대신 반시계 방향으로 그려지고 있다는 것입니다. 이로 인해 후면 컬링(backface culling)이 작동하여 어떠한 지오메트리도 그려지지 않게 됩니다. `createGraphicsPipeline` 함수로 가서 `VkPipelineRasterizationStateCreateInfo`의 `frontFace`를 수정하여 이를 바로잡습니다.

```c++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE;
```

프로그램을 다시 실행하면 이제 다음을 볼 수 있습니다:

![](/images/spinning_quad.png)

투영 행렬이 이제 종횡비를 보정하기 때문에 직사각형이 정사각형으로 바뀌었습니다. `updateUniformBuffer`가 화면 크기 조정을 처리하므로 `recreateSwapChain`에서 디스크립터 셋을 다시 생성할 필요가 없습니다.

## 정렬 요구사항(Alignment requirements)

지금까지 우리가 간과했던 한 가지는 C++ 구조체의 데이터가 셰이더의 유니폼 정의와 정확히 어떻게 일치해야 하는가입니다. 단순히 양쪽에서 같은 타입을 사용하는 것이 당연해 보입니다.

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

하지만, 이것이 전부는 아닙니다. 예를 들어, 구조체와 셰이더를 다음과 같이 수정해 보세요.

```c++
struct UniformBufferObject {
    glm::vec2 foo;
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    vec2 foo;
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

셰이더와 프로그램을 다시 컴파일하고 실행하면, 지금까지 작업했던 다채로운 사각형이 사라진 것을 발견할 것입니다! 이는 우리가 *정렬 요구사항*을 고려하지 않았기 때문입니다.

Vulkan은 구조체의 데이터가 메모리에서 특정 방식으로 정렬되기를 기대합니다. 예를 들면:

*   스칼라는 N (= 32비트 float의 경우 4바이트)에 의해 정렬되어야 합니다.
*   `vec2`는 2N (= 8바이트)에 의해 정렬되어야 합니다.
*   `vec3` 또는 `vec4`는 4N (= 16바이트)에 의해 정렬되어야 합니다.
*   중첩된 구조체는 멤버들의 기본 정렬 값을 16의 배수로 올림한 값에 의해 정렬되어야 합니다.
*   `mat4` 행렬은 `vec4`와 동일한 정렬을 가져야 합니다.

정렬 요구사항의 전체 목록은 [사양서](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap15.html#interfaces-resources-layout)에서 찾을 수 있습니다.

세 개의 `mat4` 필드만 있었던 우리의 원래 셰이더는 이미 정렬 요구사항을 충족했습니다. 각 `mat4`는 4 x 4 x 4 = 64바이트 크기이므로, `model`의 오프셋은 `0`, `view`의 오프셋은 64, `proj`의 오프셋은 128입니다. 이들은 모두 16의 배수이므로 문제가 없었습니다.

새로운 구조체는 크기가 8바이트뿐인 `vec2`로 시작하며, 이 때문에 모든 오프셋이 틀어집니다. 이제 `model`의 오프셋은 `8`, `view`는 `72`, `proj`는 `136`이 되며, 어느 것도 16의 배수가 아닙니다. 이 문제를 해결하기 위해 C++11에 도입된 [`alignas`](https://en.cppreference.com/w/cpp/language/alignas) 지정자를 사용할 수 있습니다.

```c++
struct UniformBufferObject {
    glm::vec2 foo;
    alignas(16) glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

이제 프로그램을 다시 컴파일하고 실행하면 셰이더가 행렬 값을 다시 올바르게 받는 것을 볼 수 있습니다.

다행히도 *대부분의* 경우 이러한 정렬 요구사항에 대해 생각하지 않아도 되는 방법이 있습니다. GLM을 포함하기 직전에 `GLM_FORCE_DEFAULT_ALIGNED_GENTYPES`를 정의할 수 있습니다.

```c++
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEFAULT_ALIGNED_GENTYPES
#include <glm/glm.hpp>
```

이렇게 하면 GLM이 정렬 요구사항이 이미 지정된 버전의 `vec2`와 `mat4`를 사용하도록 강제합니다. 이 정의를 추가하면 `alignas` 지정자를 제거해도 프로그램이 여전히 작동할 것입니다.

안타깝게도 이 방법은 중첩 구조체를 사용하기 시작하면 제대로 작동하지 않을 수 있습니다. C++ 코드에서 다음과 같은 정의를 생각해 보세요.

```c++
struct Foo {
    glm::vec2 v;
};

struct UniformBufferObject {
    Foo f1;
    Foo f2;
};
```

그리고 다음 셰이더 정의:

```c++
struct Foo {
    vec2 v;
};

layout(binding = 0) uniform UniformBufferObject {
    Foo f1;
    Foo f2;
} ubo;
```

이 경우 `f2`는 중첩 구조체이므로 오프셋이 `16`이어야 하지만, 실제로는 `8`의 오프셋을 갖게 됩니다. 이런 경우에는 정렬을 직접 지정해야 합니다.

```c++
struct UniformBufferObject {
    Foo f1;
    alignas(16) Foo f2;
};
```

이러한 함정들은 항상 정렬에 대해 명시적으로 처리하는 것이 좋은 이유입니다. 그렇게 하면 정렬 오류의 이상한 증상에 당황하지 않을 것입니다.

```c++
struct UniformBufferObject {
    alignas(16) glm::mat4 model;
    alignas(16) glm::mat4 view;
    alignas(16) glm::mat4 proj;
};
```

`foo` 필드를 제거한 후 셰이더를 다시 컴파일하는 것을 잊지 마세요.

## 다중 디스크립터 셋(Multiple descriptor sets)

일부 구조체와 함수 호출에서 암시했듯이, 실제로는 여러 디스크립터 셋을 동시에 바인딩하는 것이 가능합니다. 파이프라인 레이아웃을 생성할 때 각 디스크립터 셋에 대한 디스크립터 셋 레이아웃을 지정해야 합니다. 그러면 셰이더는 다음과 같이 특정 디스크립터 셋을 참조할 수 있습니다.

```c++
layout(set = 0, binding = 0) uniform UniformBufferObject { ... }
```

이 기능을 사용하여 객체별로 다른 디스크립터와 공유되는 디스크립터를 별도의 디스크립터 셋에 넣을 수 있습니다. 이 경우 드로우 콜 간에 대부분의 디스크립터를 다시 바인딩하는 것을 피하게 되어 잠재적으로 더 효율적일 수 있습니다.

[C++ 코드](/code/23_descriptor_sets.cpp) /
[정점 셰이더](/code/22_shader_ubo.vert) /
[프래그먼트 셰이더](/code/22_shader_ubo.frag)