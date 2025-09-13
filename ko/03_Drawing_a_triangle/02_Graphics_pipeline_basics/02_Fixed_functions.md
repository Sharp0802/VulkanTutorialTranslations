기존의 그래픽 API는 그래픽 파이프라인의 대부분 단계에 대해 기본 상태를 제공했습니다. Vulkan에서는 대부분의 파이프라인 상태를 명시적으로 지정해야 하며, 이 상태는 불변의(immutable) 파이프라인 상태 객체에 구워집니다(baked). 이번 장에서는 이러한 고정 함수(fixed-function) 연산을 구성하기 위한 모든 구조체를 채워 넣을 것입니다.

## 동적 상태(Dynamic state)

*대부분*의 파이프라인 상태는 파이프라인 상태 객체에 구워져야 하지만, 제한된 일부 상태는 드로우 타임(draw time)에 파이프라인을 재생성하지 않고도 *실제로* 변경할 수 있습니다. 예를 들어 뷰포트의 크기, 선 너비, 블렌드 상수 등이 있습니다. 동적 상태를 사용하고 이러한 속성들을 제외하고 싶다면, 다음과 같이 `VkPipelineDynamicStateCreateInfo` 구조체를 채워야 합니다:

```c++
std::vector<VkDynamicState> dynamicStates = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = static_cast<uint32_t>(dynamicStates.size());
dynamicState.pDynamicStates = dynamicStates.data();
```

이렇게 하면 이러한 값들의 설정이 무시되며, 그리기 시점에 해당 데이터를 지정할 수 있게 되고(또한 지정해야만 합니다) 이는 더 유연한 설정을 가능하게 합니다. 뷰포트나 시저 상태와 같이 파이프라인 상태에 구워질 경우 설정이 더 복잡해지는 것들에 대해 매우 흔하게 사용되는 방식입니다.

## 정점 입력(Vertex input)

`VkPipelineVertexInputStateCreateInfo` 구조체는 정점 셰이더로 전달될 정점 데이터의 형식을 설명합니다. 이는 대략 두 가지 방식으로 설명됩니다:

*   바인딩(Bindings): 데이터 간의 간격 및 데이터가 정점 단위(per-vertex)인지 인스턴스 단위(per-instance)인지 여부 (참고: [인스턴싱](https://ko.wikipedia.org/wiki/인스턴싱))
*   속성 서술(Attribute descriptions): 정점 셰이더로 전달되는 속성의 타입, 어떤 바인딩에서 로드할지, 그리고 어느 오프셋에 있는지

우리는 정점 데이터를 정점 셰이더에 직접 하드코딩하고 있으므로, 이 구조체를 채워서 현재 로드할 정점 데이터가 없다고 명시할 것입니다. 이 부분은 정점 버퍼 장에서 다시 다룰 것입니다.

```c++
VkPipelineVertexInputStateCreateInfo vertexInputInfo{};
vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 0;
vertexInputInfo.pVertexBindingDescriptions = nullptr; // Optional
vertexInputInfo.vertexAttributeDescriptionCount = 0;
vertexInputInfo.pVertexAttributeDescriptions = nullptr; // Optional
```

`pVertexBindingDescriptions`와 `pVertexAttributeDescriptions` 멤버는 정점 데이터를 로드하기 위한 앞서 언급된 세부 사항들을 설명하는 구조체 배열을 가리킵니다. 이 구조체를 `createGraphicsPipeline` 함수의 `shaderStages` 배열 바로 뒤에 추가하세요.

## 입력 조립(Input assembly)

`VkPipelineInputAssemblyStateCreateInfo` 구조체는 두 가지를 설명합니다: 정점들로부터 어떤 종류의 지오메트리(geometry)가 그려질 것인지, 그리고 프리미티브 재시작(primitive restart)을 활성화할 것인지 여부입니다. 전자는 `topology` 멤버에 지정되며 다음과 같은 값을 가질 수 있습니다:

*   `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`: 정점들로부터 점을 그림
*   `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`: 재사용 없이 2개의 정점마다 선을 그림
*   `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP`: 각 선의 끝 정점이 다음 선의 시작 정점으로 사용됨
*   `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`: 재사용 없이 3개의 정점마다 삼각형을 그림
*   `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP `: 각 삼각형의 두 번째와 세 번째 정점이 다음 삼각형의 첫 두 정점으로 사용됨

일반적으로 정점은 정점 버퍼에서 인덱스 순서대로 로드되지만, *엘리먼트 버퍼(element buffer)*를 사용하면 사용할 인덱스를 직접 지정할 수 있습니다. 이를 통해 정점을 재사용하는 등의 최적화를 수행할 수 있습니다. `primitiveRestartEnable` 멤버를 `VK_TRUE`로 설정하면, `0xFFFF` 또는 `0xFFFFFFFF`와 같은 특수 인덱스를 사용하여 `_STRIP` 토폴로지 모드에서 선과 삼각형을 분리할 수 있습니다.

이 튜토리얼에서는 계속 삼각형을 그릴 예정이므로, 구조체에 다음과 같은 데이터를 사용할 것입니다:

```c++
VkPipelineInputAssemblyStateCreateInfo inputAssembly{};
inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
inputAssembly.primitiveRestartEnable = VK_FALSE;
```

## 뷰포트와 시저(Viewports and scissors)

뷰포트는 기본적으로 출력물이 렌더링될 프레임버퍼의 영역을 설명합니다. 이는 거의 항상 `(0, 0)`에서 `(width, height)`까지이며, 이 튜토리얼에서도 마찬가지입니다.

```c++
VkViewport viewport{};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = (float) swapChainExtent.width;
viewport.height = (float) swapChainExtent.height;
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
```

스왑체인과 그 이미지들의 크기는 창의 `WIDTH`와 `HEIGHT`와 다를 수 있다는 점을 기억하세요. 스왑체인 이미지는 나중에 프레임버퍼로 사용될 것이므로, 그 크기를 따라야 합니다.

`minDepth`와 `maxDepth` 값은 프레임버퍼에 사용할 깊이 값의 범위를 지정합니다. 이 값들은 `[0.0f, 1.0f]` 범위 내에 있어야 하지만, `minDepth`가 `maxDepth`보다 클 수도 있습니다. 특별한 작업을 하지 않는다면, 표준값인 `0.0f`와 `1.0f`를 사용하는 것이 좋습니다.

뷰포트가 이미지에서 프레임버퍼로의 변환을 정의하는 반면, 시저 사각형(scissor rectangle)은 픽셀이 실제로 저장될 영역을 정의합니다. 시저 사각형 밖의 모든 픽셀은 래스터라이저에 의해 버려집니다. 이것들은 변환이라기보다는 필터처럼 작동합니다. 차이점은 아래 그림에 설명되어 있습니다. 왼쪽 시저 사각형은 뷰포트보다 크기만 하다면 해당 이미지를 만들어내는 여러 가능성 중 하나일 뿐이라는 점에 유의하세요.

![](/images/viewports_scissors.png)

따라서 프레임버퍼 전체에 그리고 싶다면, 프레임버퍼 전체를 덮는 시저 사각형을 지정하면 됩니다:

```c++
VkRect2D scissor{};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;
```

뷰포트와 시저 사각형은 파이프라인의 정적 부분으로 지정하거나 커맨드 버퍼에 설정되는 [동적 상태](#dynamic-state)로 지정할 수 있습니다. 전자가 다른 상태들과 더 일관성이 있지만, 뷰포트와 시저 상태를 동적으로 만드는 것이 훨씬 더 많은 유연성을 제공하기 때문에 종종 편리합니다. 이는 매우 일반적이며 모든 구현에서 성능 저하 없이 이 동적 상태를 처리할 수 있습니다.

동적 뷰포트와 시저 사각형을 선택하는 경우, 파이프라인에 대해 각각의 동적 상태를 활성화해야 합니다:

```c++
std::vector<VkDynamicState> dynamicStates = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = static_cast<uint32_t>(dynamicStates.size());
dynamicState.pDynamicStates = dynamicStates.data();
```

그리고 파이프라인 생성 시에는 그 개수만 지정하면 됩니다:

```c++
VkPipelineViewportStateCreateInfo viewportState{};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.scissorCount = 1;
```

실제 뷰포트와 시저 사각형은 나중에 그리기 시점에 설정될 것입니다.

동적 상태를 사용하면 단일 커맨드 버퍼 내에서 다른 뷰포트나 시저 사각형을 지정하는 것조차 가능합니다.

동적 상태가 없다면, 뷰포트와 시저 사각형은 `VkPipelineViewportStateCreateInfo` 구조체를 사용하여 파이프라인에 설정되어야 합니다. 이는 이 파이프라인의 뷰포트와 시저 사각형을 불변으로 만듭니다. 이 값들을 변경하려면 새 값으로 새 파이프라인을 생성해야 합니다.

```c++
VkPipelineViewportStateCreateInfo viewportState{};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.pViewports = &viewport;
viewportState.scissorCount = 1;
viewportState.pScissors = &scissor;
```

어떻게 설정하든, 일부 그래픽 카드에서는 여러 개의 뷰포트와 시저 사각형을 사용할 수 있으므로 구조체 멤버는 이들의 배열을 참조합니다. 여러 개를 사용하려면 GPU 기능을 활성화해야 합니다 (논리 장치 생성 참조).

## 래스터라이저(Rasterizer)

래스터라이저는 정점 셰이더에서 정점들에 의해 형성된 지오메트리를 가져와 프래그먼트 셰이더에 의해 색칠될 프래그먼트로 변환합니다. 또한 [깊이 테스팅](https://en.wikipedia.org/wiki/Z-buffering), [면 컬링](https://en.wikipedia.org/wiki/Back-face_culling), 시저 테스트를 수행하며, 폴리곤 전체를 채우는 프래그먼트나 가장자리만(와이어프레임 렌더링) 출력하도록 구성할 수 있습니다. 이 모든 것은 `VkPipelineRasterizationStateCreateInfo` 구조체를 사용하여 구성됩니다.

```c++
VkPipelineRasterizationStateCreateInfo rasterizer{};
rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
rasterizer.depthClampEnable = VK_FALSE;
```

`depthClampEnable`이 `VK_TRUE`로 설정되면, 근평면(near plane)과 원평면(far plane)을 벗어난 프래그먼트는 버려지는 대신 평면에 고정(clamp)됩니다. 이는 그림자 맵과 같은 일부 특수한 경우에 유용합니다. 이를 사용하려면 GPU 기능을 활성화해야 합니다.

```c++
rasterizer.rasterizerDiscardEnable = VK_FALSE;
```

`rasterizerDiscardEnable`이 `VK_TRUE`로 설정되면, 지오메트리는 래스터라이저 단계를 통과하지 않습니다. 이는 기본적으로 프레임버퍼로의 모든 출력을 비활성화합니다.

```c++
rasterizer.polygonMode = VK_POLYGON_MODE_FILL;
```

`polygonMode`는 지오메트리에 대한 프래그먼트 생성 방식을 결정합니다. 사용 가능한 모드는 다음과 같습니다:

*   `VK_POLYGON_MODE_FILL`: 폴리곤의 영역을 프래그먼트로 채움
*   `VK_POLYGON_MODE_LINE`: 폴리곤 가장자리가 선으로 그려짐
*   `VK_POLYGON_MODE_POINT`: 폴리곤 정점이 점으로 그려짐

채우기(fill) 이외의 모드를 사용하려면 GPU 기능을 활성화해야 합니다.

```c++
rasterizer.lineWidth = 1.0f;
```

`lineWidth` 멤버는 간단하며, 프래그먼트 수 측면에서 선의 두께를 설명합니다. 지원되는 최대 선 너비는 하드웨어에 따라 다르며, `1.0f`보다 두꺼운 선을 사용하려면 `wideLines` GPU 기능을 활성화해야 합니다.

```c++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;
```

`cullMode` 변수는 사용할 면 컬링(face culling)의 유형을 결정합니다. 컬링을 비활성화하거나, 앞면(front face)을 컬링하거나, 뒷면(back face)을 컬링하거나, 또는 둘 다 컬링할 수 있습니다. `frontFace` 변수는 앞면으로 간주될 면의 정점 순서를 지정하며, 시계 방향(clockwise) 또는 반시계 방향(counterclockwise)이 될 수 있습니다.

```c++
rasterizer.depthBiasEnable = VK_FALSE;
rasterizer.depthBiasConstantFactor = 0.0f; // Optional
rasterizer.depthBiasClamp = 0.0f; // Optional
rasterizer.depthBiasSlopeFactor = 0.0f; // Optional
```

래스터라이저는 상수 값을 더하거나 프래그먼트의 기울기에 따라 값을 편향시켜 깊이 값을 변경할 수 있습니다. 이는 때때로 그림자 매핑에 사용되지만, 우리는 사용하지 않을 것입니다. `depthBiasEnable`을 `VK_FALSE`로 설정하기만 하면 됩니다.

## 멀티샘플링(Multisampling)

`VkPipelineMultisampleStateCreateInfo` 구조체는 [안티에일리어싱](https://ko.wikipedia.org/wiki/멀티샘플_안티에일리어싱)을 수행하는 방법 중 하나인 멀티샘플링을 구성합니다. 이는 동일한 픽셀로 래스터화되는 여러 폴리곤의 프래그먼트 셰이더 결과를 결합하여 작동합니다. 이는 주로 가장자리에서 발생하며, 가장 눈에 띄는 에일리어싱 아티팩트가 발생하는 곳이기도 합니다. 단 하나의 폴리곤만 픽셀에 매핑되는 경우 프래그먼트 셰이더를 여러 번 실행할 필요가 없기 때문에, 단순히 더 높은 해상도로 렌더링한 후 다운스케일링하는 것보다 훨씬 비용이 적게 듭니다. 이를 활성화하려면 GPU 기능을 활성화해야 합니다.

```c++
VkPipelineMultisampleStateCreateInfo multisampling{};
multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
multisampling.sampleShadingEnable = VK_FALSE;
multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
multisampling.minSampleShading = 1.0f; // Optional
multisampling.pSampleMask = nullptr; // Optional
multisampling.alphaToCoverageEnable = VK_FALSE; // Optional
multisampling.alphaToOneEnable = VK_FALSE; // Optional
```

멀티샘플링은 나중 장에서 다시 다룰 것이며, 지금은 비활성화된 상태로 두겠습니다.

## 깊이 및 스텐실 테스팅(Depth and stencil testing)

깊이 버퍼 및/또는 스텐실 버퍼를 사용하는 경우, `VkPipelineDepthStencilStateCreateInfo`를 사용하여 깊이 및 스텐실 테스트도 구성해야 합니다. 우리는 지금 둘 다 없으므로, 해당 구조체에 대한 포인터 대신 `nullptr`를 전달하면 됩니다. 이 부분은 깊이 버퍼링 장에서 다시 다룰 것입니다.

## 색상 블렌딩(Color blending)

프래그먼트 셰이더가 색상을 반환한 후, 이 색상은 프레임버퍼에 이미 있는 색상과 결합되어야 합니다. 이 변환을 색상 블렌딩(color blending)이라고 하며, 두 가지 방법이 있습니다:

*   이전 값과 새 값을 혼합하여 최종 색상을 생성
*   비트 연산(bitwise operation)을 사용하여 이전 값과 새 값을 결합

색상 블렌딩을 구성하기 위한 두 가지 유형의 구조체가 있습니다. 첫 번째 구조체인 `VkPipelineColorBlendAttachmentState`는 연결된 각 프레임버퍼에 대한 구성을 포함하고, 두 번째 구조체인 `VkPipelineColorBlendStateCreateInfo`는 *전역* 색상 블렌딩 설정을 포함합니다. 우리의 경우 프레임버퍼는 하나뿐입니다:

```c++
VkPipelineColorBlendAttachmentState colorBlendAttachment{};
colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
colorBlendAttachment.blendEnable = VK_FALSE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD; // Optional
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD; // Optional
```

이 프레임버퍼별 구조체를 사용하면 첫 번째 방식의 색상 블렌딩을 구성할 수 있습니다. 수행될 연산은 다음 의사 코드로 가장 잘 설명됩니다:

```c++
if (blendEnable) {
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <alphaBlendOp> (dstAlphaBlendFactor * oldColor.a);
} else {
    finalColor = newColor;
}

finalColor = finalColor & colorWriteMask;
```

`blendEnable`이 `VK_FALSE`로 설정되면, 프래그먼트 셰이더의 새 색상은 수정 없이 그대로 통과합니다. 그렇지 않으면, 두 혼합 연산이 수행되어 새 색상을 계산합니다. 결과 색상은 `colorWriteMask`와 AND 연산되어 실제로 어떤 채널이 통과할지 결정됩니다.

색상 블렌딩을 사용하는 가장 일반적인 방법은 알파 블렌딩을 구현하는 것입니다. 여기서 우리는 새 색상이 불투명도에 따라 이전 색상과 혼합되기를 원합니다. `finalColor`는 다음과 같이 계산되어야 합니다:

```c++
finalColor.rgb = newAlpha * newColor + (1 - newAlpha) * oldColor;
finalColor.a = newAlpha.a;
```

이는 다음 매개변수로 달성할 수 있습니다:

```c++
colorBlendAttachment.blendEnable = VK_TRUE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

가능한 모든 연산은 사양의 `VkBlendFactor` 및 `VkBlendOp` 열거형에서 찾을 수 있습니다.

두 번째 구조체는 모든 프레임버퍼에 대한 구조체 배열을 참조하며, 앞서 언급한 계산에서 블렌드 인자로 사용할 수 있는 블렌드 상수를 설정할 수 있게 해줍니다.

```c++
VkPipelineColorBlendStateCreateInfo colorBlending{};
colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlending.logicOpEnable = VK_FALSE;
colorBlending.logicOp = VK_LOGIC_OP_COPY; // Optional
colorBlending.attachmentCount = 1;
colorBlending.pAttachments = &colorBlendAttachment;
colorBlending.blendConstants[0] = 0.0f; // Optional
colorBlending.blendConstants[1] = 0.0f; // Optional
colorBlending.blendConstants[2] = 0.0f; // Optional
colorBlending.blendConstants[3] = 0.0f; // Optional
```

두 번째 블렌딩 방식(비트 조합)을 사용하려면 `logicOpEnable`을 `VK_TRUE`로 설정해야 합니다. 그러면 `logicOp` 필드에서 비트 연산을 지정할 수 있습니다. 이는 연결된 모든 프레임버퍼에 대해 `blendEnable`을 `VK_FALSE`로 설정한 것처럼 첫 번째 방식을 자동으로 비활성화합니다! `colorWriteMask`는 이 모드에서도 프레임버퍼의 어떤 채널이 실제로 영향을 받을지 결정하는 데 사용됩니다. 우리가 여기서 한 것처럼 두 모드를 모두 비활성화하는 것도 가능하며, 이 경우 프래그먼트 색상은 수정 없이 프레임버퍼에 기록됩니다.

## 파이프라인 레이아웃(Pipeline layout)

셰이더에서 `uniform` 값을 사용할 수 있는데, 이는 동적 상태 변수와 유사한 전역 변수로서 그리기 시점에 변경하여 셰이더를 재생성하지 않고도 동작을 바꿀 수 있습니다. 일반적으로 변환 행렬을 정점 셰이더에 전달하거나 프래그먼트 셰이더에서 텍스처 샘플러를 생성하는 데 사용됩니다.

이러한 유니폼 값은 파이프라인 생성 중에 `VkPipelineLayout` 객체를 생성하여 지정해야 합니다. 이후 장까지 사용하지는 않겠지만, 여전히 빈 파이프라인 레이아웃을 생성해야 합니다.

이 객체를 담을 클래스 멤버를 만드세요. 나중에 다른 함수에서 참조할 것이기 때문입니다:

```c++
VkPipelineLayout pipelineLayout;
```

그리고 `createGraphicsPipeline` 함수에서 객체를 생성합니다:

```c++
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 0; // Optional
pipelineLayoutInfo.pSetLayouts = nullptr; // Optional
pipelineLayoutInfo.pushConstantRangeCount = 0; // Optional
pipelineLayoutInfo.pPushConstantRanges = nullptr; // Optional

if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create pipeline layout!");
}
```

이 구조체는 *푸시 상수(push constants)*도 지정하는데, 이는 셰이더에 동적 값을 전달하는 또 다른 방법으로 이후 장에서 다룰 수 있습니다. 파이프라인 레이아웃은 프로그램의 수명 내내 참조될 것이므로, 마지막에 파괴되어야 합니다:

```c++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

## 결론(Conclusion)

이것으로 모든 고정 함수 상태에 대한 설명이 끝났습니다! 이 모든 것을 처음부터 설정하는 것은 많은 작업이지만, 장점은 이제 그래픽 파이프라인에서 일어나는 모든 일을 거의 완전히 파악하게 되었다는 것입니다! 이는 특정 구성 요소의 기본 상태가 예상과 달라 예기치 않은 동작에 부딪힐 가능성을 줄여줍니다.

그러나 그래픽 파이프라인을 최종적으로 생성하기 전에 만들어야 할 객체가 하나 더 있는데, 바로 [렌더 패스](!/kr/Drawing_a_triangle/Graphics_pipeline_basics/Render_passes)입니다.

[C++ 코드](/code/10_fixed_functions.cpp) /
[정점 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)