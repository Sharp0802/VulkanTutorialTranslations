## 전체 구조

이전 장에서는 적절한 설정을 모두 마친 Vulkan 프로젝트를 만들고 샘플 코드로 테스트해 보았습니다. 이번 장에서는 아래 코드를 가지고 처음부터 시작해 보겠습니다.

```c++
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <cstdlib>

class HelloTriangleApplication {
public:
    void run() {
        initVulkan();
        mainLoop();
        cleanup();
    }

private:
    void initVulkan() {

    }

    void mainLoop() {

    }

    void cleanup() {

    }
};

int main() {
    HelloTriangleApplication app;

    try {
        app.run();
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

먼저 LunarG SDK의 Vulkan 헤더를 포함합니다. 이 헤더는 Vulkan의 함수, 구조체, 열거형을 제공합니다. `stdexcept`와 `iostream` 헤더는 오류를 보고하고 전파하는 데 사용됩니다. `cstdlib` 헤더는 `EXIT_SUCCESS`와 `EXIT_FAILURE` 매크로를 제공합니다.

프로그램 자체는 클래스로 감싸여 있습니다. 여기서 Vulkan 객체들을 private 클래스 멤버로 저장하고, 각 객체를 초기화하는 함수를 추가하여 `initVulkan` 함수에서 호출할 것입니다. 모든 준비가 끝나면 메인 루프에 진입하여 프레임 렌더링을 시작합니다. 잠시 후에 `mainLoop` 함수를 채워 창이 닫힐 때까지 반복하는 루프를 포함시킬 것입니다. 창이 닫히고 `mainLoop`가 반환되면, `cleanup` 함수에서 사용했던 리소스를 확실히 해제할 것입니다.

실행 중 치명적인 오류가 발생하면, 설명적인 메시지와 함께 `std::runtime_error` 예외를 던질 것입니다. 이 예외는 `main` 함수로 다시 전파되어 명령 프롬프트에 출력됩니다. 다양한 표준 예외 타입도 처리하기 위해, 더 일반적인 `std::exception`을 catch합니다. 곧 다루게 될 오류의 한 예는 특정 필수 익스텐션(extension)이 지원되지 않는 것을 발견하는 경우입니다.

이 장 이후의 거의 모든 장에서는 `initVulkan`에서 호출될 새로운 함수 하나와, 마지막에 `cleanup`에서 해제해야 할 하나 이상의 새로운 Vulkan 객체를 private 클래스 멤버에 추가하게 될 것입니다.

## 리소스 관리

`malloc`으로 할당된 모든 메모리 덩어리에 `free` 호출이 필요한 것처럼, 우리가 생성하는 모든 Vulkan 객체는 더 이상 필요하지 않을 때 명시적으로 파괴되어야 합니다. C++에서는 `<memory>` 헤더에 제공되는 [RAII](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)나 스마트 포인터를 사용하여 자동 리소스 관리를 수행할 수 있습니다. 하지만, 이 튜토리얼에서는 Vulkan 객체의 할당과 해제를 명시적으로 다루기로 했습니다. 결국, Vulkan의 특징은 실수를 피하기 위해 모든 작업을 명시적으로 처리하는 것이므로, API가 어떻게 작동하는지 배우기 위해 객체의 생명주기를 명시적으로 다루는 것이 좋습니다.

이 튜토리얼을 마친 후에는, 생성자에서 Vulkan 객체를 획득하고 소멸자에서 해제하는 C++ 클래스를 작성하거나, 소유권 요구사항에 따라 `std::unique_ptr` 또는 `std::shared_ptr`에 사용자 정의 삭제자(deleter)를 제공하는 방식으로 자동 리소스 관리를 구현할 수 있습니다. RAII는 대규모 Vulkan 프로그램에 권장되는 모델이지만, 학습 목적으로는 배후에서 어떤 일이 일어나는지 아는 것이 항상 좋습니다.

Vulkan 객체는 `vkCreateXXX`와 같은 함수로 직접 생성되거나, `vkAllocateXXX`와 같은 함수를 통해 다른 객체를 통해 할당됩니다. 객체가 더 이상 어디에서도 사용되지 않는 것을 확인한 후에는, 대응하는 `vkDestroyXXX` 및 `vkFreeXXX`로 파괴해야 합니다. 이 함수들의 매개변수는 일반적으로 객체 유형에 따라 다르지만, 모두가 공유하는 하나의 매개변수가 있습니다: `pAllocator`입니다. 이는 사용자 정의 메모리 할당자를 위한 콜백을 지정할 수 있게 해주는 선택적 매개변수입니다. 이 튜토리얼에서는 이 매개변수를 무시하고 항상 `nullptr`을 인자로 전달할 것입니다.

## GLFW 연동하기

오프스크린 렌더링에 Vulkan을 사용하려는 경우 창을 만들지 않아도 완벽하게 작동하지만, 실제로 무언가를 보여주는 것이 훨씬 더 흥미롭습니다! 먼저 `#include <vulkan/vulkan.h>` 줄을 다음과 같이 바꾸세요.

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```

이렇게 하면 GLFW가 자체 정의를 포함하고 Vulkan 헤더도 자동으로 불러옵니다. `initWindow` 함수를 추가하고, `run` 함수에서 다른 호출들보다 먼저 이 함수를 호출하도록 추가하세요. 이 함수를 사용하여 GLFW를 초기화하고 창을 만들 것입니다.

```c++
void run() {
    initWindow();
    initVulkan();
    mainLoop();
    cleanup();
}

private:
    void initWindow() {

    }
```

`initWindow`에서의 첫 번째 호출은 `glfwInit()`이어야 합니다. 이 함수는 GLFW 라이브러리를 초기화합니다. GLFW는 원래 OpenGL 컨텍스트를 생성하도록 설계되었기 때문에, 후속 호출로 OpenGL 컨텍스트를 생성하지 않도록 알려줘야 합니다.

```c++
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```

창 크기 조정을 처리하는 것은 나중에 다룰 특별한 주의가 필요하므로, 지금은 다른 윈도우 힌트 호출로 비활성화합니다.

```c++
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
```

이제 실제 창을 생성하는 것만 남았습니다. `GLFWwindow* window;` private 클래스 멤버를 추가하여 창에 대한 참조를 저장하고, 다음과 같이 창을 초기화하세요.

```c++
window = glfwCreateWindow(800, 600, "Vulkan", nullptr, nullptr);
```

처음 세 매개변수는 창의 너비, 높이, 제목을 지정합니다. 네 번째 매개변수를 사용하면 창을 열 모니터를 선택적으로 지정할 수 있고, 마지막 매개변수는 OpenGL에만 관련이 있습니다.

앞으로 이 값들을 여러 번 참조하게 될 것이므로, 하드코딩된 너비와 높이 숫자 대신 상수를 사용하는 것이 좋습니다. 저는 `HelloTriangleApplication` 클래스 정의 위에 다음 줄들을 추가했습니다.

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;
```

그리고 창 생성 호출을 다음과 같이 변경했습니다.

```c++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
```

이제 다음과 같은 `initWindow` 함수가 완성되었을 것입니다.

```c++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
}
```

오류가 발생하거나 창이 닫힐 때까지 애플리케이션을 계속 실행하려면, `mainLoop` 함수에 다음과 같이 이벤트 루프를 추가해야 합니다.

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
}
```

이 코드는 상당히 직관적일 것입니다. 사용자가 창을 닫을 때까지 루프를 돌며 X 버튼 누르기와 같은 이벤트를 확인합니다. 이 루프는 나중에 단일 프레임을 렌더링하는 함수를 호출할 곳이기도 합니다.

창이 닫히면, 창을 파괴하고 GLFW 자체를 종료하여 리소스를 정리해야 합니다. 이것이 우리의 첫 `cleanup` 코드가 될 것입니다.

```c++
void cleanup() {
    glfwDestroyWindow(window);

    glfwTerminate();
}
```

이제 프로그램을 실행하면 `Vulkan`이라는 제목의 창이 나타나고, 창을 닫아 애플리케이션을 종료할 때까지 유지되는 것을 볼 수 있습니다. 이제 Vulkan 애플리케이션의 기본 골격을 갖추었으니, [첫 번째 Vulkan 객체를 생성해 봅시다](!/kr/Drawing_a_triangle/Setup/Instance)!

[C++ 코드](/code/00_base_code.cpp)