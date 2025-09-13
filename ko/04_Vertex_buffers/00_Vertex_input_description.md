## 서론

앞으로 몇 개의 챕터에서, 우리는 정점 셰이더에 하드코딩된 정점 데이터를 메모리의 정점 버퍼로 교체할 것입니다. 가장 쉬운 접근법인 CPU에서 접근 가능한 버퍼를 만들고 `memcpy`를 사용하여 정점 데이터를 직접 복사하는 것으로 시작하겠습니다. 그 후에는 스테이징 버퍼를 사용해 정점 데이터를 고성능 메모리로 복사하는 방법을 알아볼 것입니다.

## 정점 셰이더

먼저 정점 셰이더 코드가 더 이상 정점 데이터를 포함하지 않도록 변경합니다. 정점 셰이더는 `in` 키워드를 사용해 정점 버퍼로부터 입력을 받습니다.

```glsl
#version 450

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

`inPosition`과 `inColor` 변수는 *정점 어트리뷰트(vertex attribute)*입니다. 이것들은 마치 우리가 두 배열을 사용해 정점별 위치와 색상을 수동으로 지정했던 것처럼, 정점 버퍼에서 정점별로 지정되는 속성입니다. 정점 셰이더를 다시 컴파일하는 것을 잊지 마세요!

`fragColor`와 마찬가지로, `layout(location = x)` 어노테이션은 나중에 우리가 참조할 수 있는 인덱스를 입력에 할당합니다. `dvec3` 64비트 벡터와 같은 일부 타입은 여러 *슬롯*을 사용한다는 점을 아는 것이 중요합니다. 즉, 그 다음의 인덱스는 최소 2 이상 높아야 합니다:

```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

레이아웃 한정자(layout qualifier)에 대한 더 많은 정보는 [OpenGL 위키](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL))에서 찾을 수 있습니다.

## 정점 데이터

셰이더 코드에 있던 정점 데이터를 우리 프로그램 코드의 배열로 옮길 것입니다. 먼저 벡터 및 행렬과 같은 선형대수 관련 타입을 제공하는 GLM 라이브러리를 포함하는 것으로 시작합니다. 우리는 이 타입들을 사용해 위치와 색상 벡터를 지정할 것입니다.

```c++
#include <glm/glm.hpp>
```

정점 셰이더에서 사용할 두 개의 어트리뷰트를 포함하는 `Vertex`라는 새 구조체를 만듭니다:

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
};
```

GLM은 편리하게도 셰이더 언어에서 사용되는 벡터 타입과 정확히 일치하는 C++ 타입을 제공합니다.

```c++
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

이제 `Vertex` 구조체를 사용하여 정점 데이터의 배열을 지정합니다. 이전과 정확히 같은 위치와 색상 값을 사용하지만, 이제는 하나의 정점 배열로 결합되었습니다. 이를 정점 어트리뷰트 *인터리빙(interleaving)*이라고 합니다.

## 바인딩 서술 (Binding descriptions)

다음 단계는 이 데이터 형식이 GPU 메모리에 업로드된 후 정점 셰이더에 어떻게 전달되는지 Vulkan에 알려주는 것입니다. 이 정보를 전달하기 위해서는 두 가지 유형의 구조체가 필요합니다.

첫 번째 구조체는 `VkVertexInputBindingDescription`이며, `Vertex` 구조체에 멤버 함수를 추가하여 이 구조체를 올바른 데이터로 채울 것입니다.

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};

        return bindingDescription;
    }
};
```

정점 바인딩은 정점들을 순회하며 메모리로부터 데이터를 어떤 속도로 로드할지 서술합니다. 이는 데이터 항목 사이의 바이트 수와 각 정점 또는 각 인스턴스 후에 다음 데이터 항목으로 이동할지 여부를 지정합니다.

```c++
VkVertexInputBindingDescription bindingDescription{};
bindingDescription.binding = 0;
bindingDescription.stride = sizeof(Vertex);
bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
```

우리의 모든 정점별 데이터는 하나의 배열에 함께 묶여 있으므로, 바인딩은 하나만 가질 것입니다. `binding` 매개변수는 바인딩 배열 내에서 바인딩의 인덱스를 지정합니다. `stride` 매개변수는 한 항목에서 다음 항목까지의 바이트 수를 지정하며, `inputRate` 매개변수는 다음 값 중 하나를 가질 수 있습니다:

*   `VK_VERTEX_INPUT_RATE_VERTEX`: 각 정점 후에 다음 데이터 항목으로 이동
*   `VK_VERTEX_INPUT_RATE_INSTANCE`: 각 인스턴스 후에 다음 데이터 항목으로 이동

우리는 인스턴스 렌더링을 사용하지 않을 것이므로, 정점별 데이터를 고수할 것입니다.

## 어트리뷰트 서술 (Attribute descriptions)

정점 입력을 처리하는 방법을 서술하는 두 번째 구조체는 `VkVertexInputAttributeDescription`입니다. 이 구조체들을 채우기 위해 `Vertex`에 또 다른 헬퍼 함수를 추가할 것입니다.

```c++
#include <array>

...

static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
    std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

    return attributeDescriptions;
}
```

함수 프로토타입이 나타내듯이, 이 구조체는 두 개가 될 것입니다. 어트리뷰트 서술 구조체는 바인딩 서술로부터 온 정점 데이터 청크에서 정점 어트리뷰트를 어떻게 추출할지 서술합니다. 우리는 위치와 색상, 두 개의 어트리뷰트를 가지고 있으므로 두 개의 어트리뷰트 서술 구조체가 필요합니다.

```c++
attributeDescriptions[0].binding = 0;
attributeDescriptions[0].location = 0;
attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
attributeDescriptions[0].offset = offsetof(Vertex, pos);
```

`binding` 매개변수는 정점별 데이터가 어느 바인딩에서 오는지 Vulkan에 알려줍니다. `location` 매개변수는 정점 셰이더에 있는 입력의 `location` 지시문을 참조합니다. 정점 셰이더에서 `location 0`인 입력은 위치이며, 이는 두 개의 32비트 부동소수점 컴포넌트를 가집니다.

`format` 매개변수는 어트리뷰트의 데이터 타입을 서술합니다. 약간 혼란스럽게도, 형식은 색상 형식과 동일한 열거형을 사용하여 지정됩니다. 다음 셰이더 타입과 형식은 일반적으로 함께 사용됩니다:

*   `float`: `VK_FORMAT_R32_SFLOAT`
*   `vec2`: `VK_FORMAT_R32G32_SFLOAT`
*   `vec3`: `VK_FORMAT_R32G32B32_SFLOAT`
*   `vec4`: `VK_FORMAT_R32G32B32A32_SFLOAT`

보시다시피, 셰이더 데이터 타입의 컴포넌트 수와 색상 채널 수가 일치하는 형식을 사용해야 합니다. 셰이더의 컴포넌트 수보다 더 많은 채널을 사용하는 것은 허용되지만, 초과분은 조용히 무시됩니다. 채널 수가 컴포넌트 수보다 적으면, BGA 컴포넌트는 `(0, 0, 1)`의 기본값을 사용하게 됩니다. 색상 타입(`SFLOAT`, `UINT`, `SINT`)과 비트 너비 또한 셰이더 입력의 타입과 일치해야 합니다. 다음 예시를 참조하세요:

*   `ivec2`: `VK_FORMAT_R32G32_SINT`, 32비트 부호 있는 정수의 2-컴포넌트 벡터
*   `uvec4`: `VK_FORMAT_R32G32B32A32_UINT`, 32비트 부호 없는 정수의 4-컴포넌트 벡터
*   `double`: `VK_FORMAT_R64_SFLOAT`, 배정밀도(64비트) 부동소수점

`format` 매개변수는 어트리뷰트 데이터의 바이트 크기를 암시적으로 정의하고, `offset` 매개변수는 읽어올 정점별 데이터의 시작 부분으로부터의 바이트 수를 지정합니다. 바인딩은 한 번에 하나의 `Vertex`를 로드하며, 위치 어트리뷰트(`pos`)는 이 구조체의 시작에서 `0` 바이트 오프셋에 있습니다. 이것은 `offsetof` 매크로를 사용하여 자동으로 계산됩니다.

```c++
attributeDescriptions[1].binding = 0;
attributeDescriptions[1].location = 1;
attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
attributeDescriptions[1].offset = offsetof(Vertex, color);
```

색상 어트리뷰트도 거의 같은 방식으로 서술됩니다.

## 파이프라인 정점 입력

이제 `createGraphicsPipeline`에서 구조체들을 참조하여 이 형식의 정점 데이터를 받아들이도록 그래픽 파이프라인을 설정해야 합니다. `vertexInputInfo` 구조체를 찾아 두 서술을 참조하도록 수정합니다:

```c++
auto bindingDescription = Vertex::getBindingDescription();
auto attributeDescriptions = Vertex::getAttributeDescriptions();

vertexInputInfo.vertexBindingDescriptionCount = 1;
vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```

이제 파이프라인은 `vertices` 컨테이너 형식의 정점 데이터를 받아들여 우리의 정점 셰이더로 전달할 준비가 되었습니다. 이제 유효성 검사 레이어를 활성화한 채로 프로그램을 실행하면, 바인딩에 연결된 정점 버퍼가 없다는 오류 메시지를 보게 될 것입니다. 다음 단계는 정점 버퍼를 만들고 정점 데이터를 그곳으로 옮겨 GPU가 접근할 수 있도록 하는 것입니다.

[C++ 코드](/code/18_vertex_input.cpp) /
[정점 셰이더](/code/18_shader_vertexbuffer.vert) /
[프래그먼트 셰이더](/code/18_shader_vertexbuffer.frag)