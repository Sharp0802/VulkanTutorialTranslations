## 소개

우리는 튜토리얼의 유니폼 버퍼 부분에서 디스크립터를 처음 살펴보았습니다. 이번 장에서는 새로운 유형의 디스크립터인 *결합 이미지 샘플러(combined image sampler)*에 대해 알아보겠습니다. 이 디스크립터는 셰이더가 이전 장에서 만든 것과 같은 샘플러 객체를 통해 이미지 리소스에 접근할 수 있게 해줍니다.

먼저 디스크립터 셋 레이아웃, 디스크립터 풀, 디스크립터 셋을 수정하여 이러한 결합 이미지 샘플러 디스크립터를 포함하도록 하겠습니다. 그 후에는 `Vertex`에 텍스처 좌표를 추가하고, 프래그먼트 셰이더를 수정하여 단순히 정점 색상을 보간하는 대신 텍스처에서 색상을 읽어오도록 할 것입니다.

## 디스크립터 업데이트

`createDescriptorSetLayout` 함수로 이동하여 결합 이미지 샘플러 디스크립터를 위한 `VkDescriptorSetLayoutBinding`을 추가합니다. 간단하게 유니폼 버퍼 다음 바인딩에 배치하겠습니다:

```c++
VkDescriptorSetLayoutBinding samplerLayoutBinding{};
samplerLayoutBinding.binding = 1;
samplerLayoutBinding.descriptorCount = 1;
samplerLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
samplerLayoutBinding.pImmutableSamplers = nullptr;
samplerLayoutBinding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;

std::array<VkDescriptorSetLayoutBinding, 2> bindings = {uboLayoutBinding, samplerLayoutBinding};
VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = static_cast<uint32_t>(bindings.size());
layoutInfo.pBindings = bindings.data();
```

`stageFlags`를 설정하여 우리가 프래그먼트 셰이더에서 결합 이미지 샘플러 디스크립터를 사용하려는 의도를 나타내도록 하세요. 프래그먼트의 색상이 결정되는 곳이 바로 거기입니다. [높이맵(heightmap)](https://en.wikipedia.org/wiki/Heightmap)을 사용하여 정점 그리드를 동적으로 변형하는 것처럼, 정점 셰이더에서 텍스처 샘플링을 사용하는 것도 가능합니다.

또한 `VkDescriptorPoolCreateInfo`에 `VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER` 타입의 `VkPoolSize`를 추가하여, 결합 이미지 샘플러 할당을 위한 공간을 만들도록 더 큰 디스크립터 풀을 생성해야 합니다. `createDescriptorPool` 함수로 가서 이 디스크립터를 위한 `VkDescriptorPoolSize`를 포함하도록 수정하세요:

```c++
std::array<VkDescriptorPoolSize, 2> poolSizes{};
poolSizes[0].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
poolSizes[0].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
poolSizes[1].type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
poolSizes[1].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);

VkDescriptorPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
poolInfo.poolSizeCount = static_cast<uint32_t>(poolSizes.size());
poolInfo.pPoolSizes = poolSizes.data();
poolInfo.maxSets = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHT);
```

부적절한 디스크립터 풀은 유효성 검사 레이어가 잡아내지 못하는 문제의 좋은 예입니다: Vulkan 1.1부터, 풀이 충분히 크지 않으면 `vkAllocateDescriptorSets`가 `VK_ERROR_POOL_OUT_OF_MEMORY` 오류 코드로 실패할 수 있지만, 드라이버가 내부적으로 문제를 해결하려고 시도할 수도 있습니다. 이는 때때로 (하드웨어, 풀 크기, 할당 크기에 따라) 드라이버가 디스크립터 풀의 한계를 초과하는 할당을 허용할 수도 있다는 것을 의미합니다. 다른 경우에는 `vkAllocateDescriptorSets`가 실패하고 `VK_ERROR_POOL_OUT_OF_MEMORY`를 반환할 것입니다. 일부 기기에서는 할당이 성공하지만 다른 기기에서는 실패할 경우 특히 좌절감을 느낄 수 있습니다.

Vulkan은 할당에 대한 책임을 드라이버에 넘기기 때문에, 디스크립터 풀 생성 시 해당 `descriptorCount` 멤버에 지정된 만큼만 특정 타입(`VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER` 등)의 디스크립터를 할당해야 한다는 엄격한 요구사항은 더 이상 없습니다. 하지만, 그렇게 하는 것이 여전히 모범 사례이며, 앞으로 [모범 사례 유효성 검사(Best Practice Validation)](https://vulkan.lunarg.com/doc/view/1.4.304.0/linux/best_practices.html)를 활성화하면 `VK_LAYER_KHRONOS_validation`이 이런 종류의 문제에 대해 경고할 것입니다.

마지막 단계는 실제 이미지와 샘플러 리소스를 디스크립터 셋의 디스크립터에 바인딩하는 것입니다. `createDescriptorSets` 함수로 이동하세요.

```c++
for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    VkDescriptorBufferInfo bufferInfo{};
    bufferInfo.buffer = uniformBuffers[i];
    bufferInfo.offset = 0;
    bufferInfo.range = sizeof(UniformBufferObject);

    VkDescriptorImageInfo imageInfo{};
    imageInfo.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    imageInfo.imageView = textureImageView;
    imageInfo.sampler = textureSampler;

    ...
}
```

결합 이미지 샘플러 구조체의 리소스는, 유니폼 버퍼 디스크립터의 버퍼 리소스가 `VkDescriptorBufferInfo` 구조체에 지정되는 것과 마찬가지로, `VkDescriptorImageInfo` 구조체에 지정되어야 합니다. 이곳에서 이전 장의 객체들이 함께 사용됩니다.

```c++
std::array<VkWriteDescriptorSet, 2> descriptorWrites{};

descriptorWrites[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[0].dstSet = descriptorSets[i];
descriptorWrites[0].dstBinding = 0;
descriptorWrites[0].dstArrayElement = 0;
descriptorWrites[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorWrites[0].descriptorCount = 1;
descriptorWrites[0].pBufferInfo = &bufferInfo;

descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[1].dstSet = descriptorSets[i];
descriptorWrites[1].dstBinding = 1;
descriptorWrites[1].dstArrayElement = 0;
descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
descriptorWrites[1].descriptorCount = 1;
descriptorWrites[1].pImageInfo = &imageInfo;

vkUpdateDescriptorSets(device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
```

버퍼와 마찬가지로 디스크립터도 이 이미지 정보로 업데이트되어야 합니다. 이번에는 `pBufferInfo` 대신 `pImageInfo` 배열을 사용합니다. 이제 디스크립터는 셰이더에서 사용할 준비가 되었습니다!

## 텍스처 좌표

텍스처 매핑에 필요한 중요한 요소 중 아직 빠진 것이 하나 있는데, 바로 각 정점의 실제 텍스처 좌표입니다. 텍스처 좌표는 이미지가 지오메트리에 실제로 어떻게 매핑될지를 결정합니다.

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
    glm::vec2 texCoord;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};
        bindingDescription.binding = 0;
        bindingDescription.stride = sizeof(Vertex);
        bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

        return bindingDescription;
    }

    static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        attributeDescriptions[1].binding = 0;
        attributeDescriptions[1].location = 1;
        attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[1].offset = offsetof(Vertex, color);

        attributeDescriptions[2].binding = 0;
        attributeDescriptions[2].location = 2;
        attributeDescriptions[2].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[2].offset = offsetof(Vertex, texCoord);

        return attributeDescriptions;
    }
};
```

`Vertex` 구조체를 수정하여 텍스처 좌표를 위한 `vec2`를 포함시키세요. 또한 정점 셰이더에서 텍스처 좌표를 입력으로 접근할 수 있도록 `VkVertexInputAttributeDescription`을 추가해야 합니다. 이는 사각형 표면에 걸쳐 보간하기 위해 프래그먼트 셰이더로 전달할 수 있으려면 필수적입니다.

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}, {0.0f, 1.0f}},
    {{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}, {1.0f, 1.0f}}
};
```

이 튜토리얼에서는 왼쪽 위 모서리의 `0, 0`부터 오른쪽 아래 모서리의 `1, 1`까지의 좌표를 사용하여 간단하게 사각형을 텍스처로 채울 것입니다. 자유롭게 다른 좌표로 실험해 보세요. `0` 미만 또는 `1` 초과의 좌표를 사용하여 주소 지정 모드가 어떻게 동작하는지 확인해 보세요!

## 셰이더

마지막 단계는 텍스처에서 색상을 샘플링하도록 셰이더를 수정하는 것입니다. 먼저 정점 셰이더를 수정하여 텍스처 좌표를 프래그먼트 셰이더로 전달해야 합니다:

```glsl
layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;
layout(location = 2) in vec2 inTexCoord;

layout(location = 0) out vec3 fragColor;
layout(location = 1) out vec2 fragTexCoord;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

정점별 색상과 마찬가지로 `fragTexCoord` 값은 래스터라이저에 의해 사각형 영역에 걸쳐 부드럽게 보간됩니다. 프래그먼트 셰이더가 텍스처 좌표를 색상으로 출력하게 하여 이를 시각화할 수 있습니다:

```glsl
#version 450

layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragTexCoord, 0.0, 1.0);
}
```

아래 이미지와 같은 것을 보게 될 것입니다. 셰이더를 다시 컴파일하는 것을 잊지 마세요!

![](/images/texcoord_visualization.png)

녹색 채널은 수평 좌표를, 적색 채널은 수직 좌표를 나타냅니다. 검은색과 노란색 모서리는 텍스처 좌표가 사각형에 걸쳐 `0, 0`에서 `1, 1`까지 올바르게 보간되었음을 확인시켜 줍니다. 색상을 사용하여 데이터를 시각화하는 것은 더 나은 대안이 없을 때 사용하는 셰이더 프로그래밍의 `printf` 디버깅과 같습니다!

결합 이미지 샘플러 디스크립터는 GLSL에서 샘플러 유니폼으로 표현됩니다. 프래그먼트 셰이더에 이에 대한 참조를 추가하세요:

```glsl
layout(binding = 1) uniform sampler2D texSampler;
```

다른 유형의 이미지에 대해서는 동등한 `sampler1D`와 `sampler3D` 타입이 있습니다. 여기서 올바른 바인딩을 사용하도록 하세요.

```glsl
void main() {
    outColor = texture(texSampler, fragTexCoord);
}
```

텍스처는 내장 함수인 `texture`를 사용하여 샘플링됩니다. 이 함수는 `sampler`와 좌표를 인자로 받습니다. 샘플러는 백그라운드에서 필터링과 변환을 자동으로 처리합니다. 이제 애플리케이션을 실행하면 사각형 위에 텍스처가 표시될 것입니다:

![](/images/texture_on_square.png)

텍스처 좌표를 `1`보다 큰 값으로 스케일링하여 주소 지정 모드를 실험해 보세요. 예를 들어, 다음 프래그먼트 셰이더는 `VK_SAMPLER_ADDRESS_MODE_REPEAT`를 사용할 때 아래 이미지와 같은 결과를 생성합니다:

```glsl
void main() {
    outColor = texture(texSampler, fragTexCoord * 2.0);
}
```

![](/images/texture_on_square_repeated.png)

정점 색상을 사용하여 텍스처 색상을 조작할 수도 있습니다:

```glsl
void main() {
    outColor = vec4(fragColor * texture(texSampler, fragTexCoord).rgb, 1.0);
}
```

알파 채널은 스케일링하지 않기 위해 여기서는 RGB 채널과 알파 채널을 분리했습니다.

![](/images/texture_on_square_colorized.png)

이제 셰이더에서 이미지에 접근하는 방법을 알게 되었습니다! 이는 프레임버퍼에도 기록되는 이미지와 결합될 때 매우 강력한 기술입니다. 이러한 이미지를 입력으로 사용하여 3D 월드 내에서 후처리나 카메라 디스플레이 같은 멋진 효과를 구현할 수 있습니다.

[C++ 코드](/code/26_texture_mapping.cpp) /
[정점 셰이더](/code/26_shader_textures.vert) /
[프래그먼트 셰이더](/code/26_shader_textures.frag)