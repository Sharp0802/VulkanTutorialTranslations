이제 이전 챕터들에서 다룬 모든 구조체와 객체를 조합하여 그래픽 파이프라인을 생성할 수 있습니다! 빠른 복습을 위해, 현재 우리가 가지고 있는 객체 유형은 다음과 같습니다:

*   셰이더 스테이지: 그래픽 파이프라인의 프로그래밍 가능한 스테이지의 기능을 정의하는 셰이더 모듈
*   고정 함수 상태: 입력 어셈블리, 래스터라이저, 뷰포트, 컬러 블렌딩과 같이 파이프라인의 고정 함수 스테이지를 정의하는 모든 구조체
*   파이프라인 레이아웃: 셰이더에서 참조하며 드로우 타임에 업데이트할 수 있는 유니폼 및 푸시 값
*   렌더 패스: 파이프라인 스테이지에서 참조하는 어태치먼트와 그 사용 방식

이 모든 것을 조합하면 그래픽 파이프라인의 기능이 완벽하게 정의되므로, 이제 `createGraphicsPipeline` 함수의 끝에서 `VkGraphicsPipelineCreateInfo` 구조체를 채우기 시작할 수 있습니다. 단, 셰이더 모듈은 파이프라인 생성 중에 여전히 사용해야 하므로 `vkDestroyShaderModule` 호출보다는 앞에 있어야 합니다.

```c++
VkGraphicsPipelineCreateInfo pipelineInfo{};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
pipelineInfo.stageCount = 2;
pipelineInfo.pStages = shaderStages;
```

`VkPipelineShaderStageCreateInfo` 구조체 배열을 참조하는 것으로 시작합니다.

```c++
pipelineInfo.pVertexInputState = &vertexInputInfo;
pipelineInfo.pInputAssemblyState = &inputAssembly;
pipelineInfo.pViewportState = &viewportState;
pipelineInfo.pRasterizationState = &rasterizer;
pipelineInfo.pMultisampleState = &multisampling;
pipelineInfo.pDepthStencilState = nullptr; // Optional
pipelineInfo.pColorBlendState = &colorBlending;
pipelineInfo.pDynamicState = &dynamicState;
```

그다음, 고정 함수 스테이지를 설명하는 모든 구조체를 참조합니다.

```c++
pipelineInfo.layout = pipelineLayout;
```

그다음은 파이프라인 레이아웃인데, 이것은 구조체 포인터가 아닌 Vulkan 핸들입니다.

```c++
pipelineInfo.renderPass = renderPass;
pipelineInfo.subpass = 0;
```

그리고 마지막으로 렌더 패스에 대한 참조와 이 그래픽 파이프라인이 사용될 서브패스의 인덱스가 있습니다. 이 특정 인스턴스 대신 다른 렌더 패스를 이 파이프라인과 함께 사용하는 것도 가능하지만, 그러려면 해당 렌더 패스가 `renderPass`와 *호환 가능*해야 합니다. 호환성 요구 사항은 [여기](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap8.html#renderpass-compatibility)에 설명되어 있지만, 이 튜토리얼에서는 해당 기능을 사용하지 않을 것입니다.

```c++
pipelineInfo.basePipelineHandle = VK_NULL_HANDLE; // Optional
pipelineInfo.basePipelineIndex = -1; // Optional
```

실제로는 `basePipelineHandle`과 `basePipelineIndex`라는 두 개의 매개변수가 더 있습니다. Vulkan에서는 기존 파이프라인에서 파생하여 새로운 그래픽 파이프라인을 생성할 수 있습니다. 파이프라인 파생의 개념은, 기존 파이프라인과 많은 기능을 공유하는 파이프라인을 설정하는 비용이 더 저렴하고, 동일한 부모에서 파생된 파이프라인 간의 전환도 더 빠를 수 있다는 것입니다. `basePipelineHandle`로 기존 파이프라인의 핸들을 지정하거나, `basePipelineIndex`로 곧 생성될 다른 파이프라인을 인덱스로 참조할 수 있습니다. 지금은 파이프라인이 하나뿐이므로, 단순히 null 핸들과 유효하지 않은 인덱스를 지정하겠습니다. 이 값들은 `VkGraphicsPipelineCreateInfo`의 `flags` 필드에 `VK_PIPELINE_CREATE_DERIVATIVE_BIT` 플래그가 지정된 경우에만 사용됩니다.

이제 마지막 단계를 준비하기 위해 `VkPipeline` 객체를 담을 클래스 멤버를 생성합니다:

```c++
VkPipeline graphicsPipeline;
```

그리고 마침내 그래픽 파이프라인을 생성합니다:

```c++
if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &graphicsPipeline) != VK_SUCCESS) {
    throw std::runtime_error("failed to create graphics pipeline!");
}
```

`vkCreateGraphicsPipelines` 함수는 사실 Vulkan의 일반적인 객체 생성 함수보다 매개변수가 더 많습니다. 이 함수는 여러 개의 `VkGraphicsPipelineCreateInfo` 객체를 받아 단일 호출로 여러 `VkPipeline` 객체를 생성하도록 설계되었습니다.

두 번째 매개변수는 `VK_NULL_HANDLE` 인자를 전달한 곳으로, 선택적인 `VkPipelineCache` 객체를 참조합니다. 파이프라인 캐시는 `vkCreateGraphicsPipelines`의 여러 호출에 걸쳐 파이프라인 생성 관련 데이터를 저장하고 재사용하는 데 사용할 수 있으며, 캐시를 파일에 저장하면 프로그램 실행 간에도 재사용할 수 있습니다. 이를 통해 나중에 파이프라인 생성 속도를 크게 높일 수 있습니다. 이것에 대해서는 파이프라인 캐시 챕터에서 다룰 것입니다.

그래픽 파이프라인은 모든 일반적인 그리기 작업에 필요하므로, 프로그램이 끝날 때 파괴되어야 합니다:

```c++
void cleanup() {
    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

이제 프로그램을 실행하여 이 모든 힘든 작업이 성공적인 파이프라인 생성으로 이어졌는지 확인하세요! 화면에 무언가 나타나는 것을 볼 날이 머지않았습니다. 다음 몇 챕터에서는 스왑 체인 이미지로부터 실제 프레임버퍼를 설정하고 그리기 커맨드를 준비할 것입니다.

[C++ 코드](/code/12_graphics_pipeline_complete.cpp) /
[버텍스 셰이더](/code/09_shader_base.vert) /
[프래그먼트 셰이더](/code/09_shader_base.frag)