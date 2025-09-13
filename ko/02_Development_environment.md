이번 장에서는 Vulkan 애플리케이션 개발을 위한 환경을 설정하고 몇 가지 유용한 라이브러리를 설치할 것입니다. 컴파일러를 제외하고 우리가 사용할 모든 도구는 Windows, Linux, MacOS와 호환되지만 설치 단계가 약간 다르기 때문에 여기서는 각 운영체제별로 나누어 설명합니다.

## Windows

Windows용으로 개발하는 경우, 코드를 컴파일하기 위해 Visual Studio를 사용한다고 가정하겠습니다. 완전한 C++17 지원을 위해서는 Visual Studio 2017 또는 2019를 사용해야 합니다. 아래에 설명된 단계는 VS 2017을 기준으로 작성되었습니다.

### Vulkan SDK

Vulkan 애플리케이션 개발에 필요한 가장 중요한 구성 요소는 SDK입니다. SDK에는 헤더, 표준 유효성 검사 레이어, 디버깅 도구 및 Vulkan 함수를 위한 로더가 포함되어 있습니다. 로더는 런타임에 드라이버에서 함수를 찾는데, OpenGL의 GLEW에 익숙하다면 이와 유사한 방식이라고 생각하면 됩니다.

[LunarG 웹사이트](https://vulkan.lunarg.com/)에서 페이지 하단의 버튼을 사용하여 SDK를 다운로드할 수 있습니다. 계정을 만들 필요는 없지만, 계정을 만들면 유용할 수 있는 몇 가지 추가 문서에 접근할 수 있습니다.

![](/images/vulkan_sdk_download_buttons.png)

설치를 진행하고 SDK의 설치 위치에 주의하세요. 가장 먼저 할 일은 그래픽 카드와 드라이버가 Vulkan을 제대로 지원하는지 확인하는 것입니다. SDK를 설치한 디렉터리로 이동하여 `Bin` 디렉터리를 열고 `vkcube.exe` 데모를 실행하세요. 다음과 같은 화면이 보여야 합니다:

![](/images/cube_demo.png)

오류 메시지가 나타나면 드라이버가 최신 버전인지, Vulkan 런타임을 포함하고 있는지, 그리고 그래픽 카드가 지원되는지 확인하세요. 주요 공급업체의 드라이버 링크는 [소개 장](!ko/Introduction)을 참조하세요.

이 디렉터리에는 개발에 유용한 다른 프로그램도 있습니다. `glslangValidator.exe`와 `glslc.exe` 프로그램은 사람이 읽을 수 있는 [GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language) 형식의 셰이더를 바이트코드로 컴파일하는 데 사용됩니다. 이 내용은 [셰이더 모듈](!ko/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules) 장에서 자세히 다룰 것입니다. `Bin` 디렉터리에는 Vulkan 로더와 유효성 검사 레이어의 바이너리가 포함되어 있으며, `Lib` 디렉터리에는 라이브러리가 포함되어 있습니다.

마지막으로, `Include` 디렉터리에는 Vulkan 헤더가 포함되어 있습니다. 다른 파일들도 자유롭게 둘러보셔도 되지만, 이 튜토리얼에서는 필요하지 않습니다.

### GLFW

앞서 언급했듯이, Vulkan 자체는 플랫폼에 구애받지 않는 API이며 렌더링 결과를 표시할 창을 만드는 도구를 포함하지 않습니다. Vulkan의 크로스 플랫폼 이점을 활용하고 Win32의 끔찍함을 피하기 위해, 우리는 [GLFW 라이브러리](http://www.glfw.org/)를 사용하여 창을 만들 것입니다. GLFW는 Windows, Linux, MacOS를 지원합니다. [SDL](https://www.libsdl.org/)과 같이 이 목적을 위한 다른 라이브러리도 있지만, GLFW의 장점은 단순히 창 생성 외에도 Vulkan의 다른 플랫폼별 사항들을 추상화해준다는 것입니다.

[공식 웹사이트](http://www.glfw.org/download.html)에서 최신 버전의 GLFW를 찾을 수 있습니다. 이 튜토리얼에서는 64비트 바이너리를 사용할 것이지만, 물론 32비트 모드로 빌드할 수도 있습니다. 그럴 경우 `Lib` 대신 `Lib32` 디렉터리에 있는 Vulkan SDK 바이너리와 링크해야 합니다. 다운로드한 후, 편리한 위치에 압축을 푸세요. 저는 문서 폴더 아래 Visual Studio 디렉터리 안에 `Libraries`라는 디렉터리를 만들었습니다.

![](/images/glfw_directory.png)

### GLM

DirectX 12와 달리, Vulkan은 선형 대수 연산을 위한 라이브러리를 포함하지 않으므로, 직접 다운로드해야 합니다. [GLM](http://glm.g-truc.net/)은 그래픽 API와 함께 사용하도록 설계된 멋진 라이브러리이며 OpenGL에서도 흔히 사용됩니다.

GLM은 헤더 전용 라이브러리이므로, [최신 버전](https://github.com/g-truc/glm/releases)을 다운로드하여 편리한 위치에 저장하기만 하면 됩니다. 이제 다음과 유사한 디렉터리 구조를 갖게 됩니다:

![](/images/library_directory.png)

### Visual Studio 설정하기

이제 모든 종속성을 설치했으니, Vulkan을 위한 기본 Visual Studio 프로젝트를 설정하고 모든 것이 작동하는지 확인하기 위해 약간의 코드를 작성해 봅시다.

Visual Studio를 시작하고 이름을 입력한 후 `확인`을 눌러 새 `Windows 데스크톱 마법사` 프로젝트를 만듭니다.

![](/images/vs_new_cpp_project.png)

디버그 메시지를 출력할 공간이 있도록 애플리케이션 종류로 `콘솔 애플리케이션(.exe)`을 선택하고, Visual Studio가 상용구 코드를 추가하지 않도록 `빈 프로젝트`를 체크합니다.

![](/images/vs_application_settings.png)

`확인`을 눌러 프로젝트를 만들고 C++ 소스 파일을 추가합니다. 이미 방법을 알고 계시겠지만, 완전성을 위해 단계를 포함했습니다.

![](/images/vs_new_item.png)

![](/images/vs_new_source_file.png)

이제 파일에 다음 코드를 추가하세요. 지금 당장 이 코드를 이해하려고 애쓰지 않아도 됩니다. 우리는 단지 Vulkan 애플리케이션을 컴파일하고 실행할 수 있는지 확인하는 것뿐입니다. 다음 장부터 처음부터 시작할 것입니다.

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

이제 오류를 없애기 위해 프로젝트를 구성해 봅시다. 프로젝트 속성 대화 상자를 열고 `모든 구성`이 선택되었는지 확인하세요. 대부분의 설정이 `Debug`와 `Release` 모드 모두에 적용되기 때문입니다.

![](/images/vs_open_project_properties.png)

![](/images/vs_all_configs.png)

`C/C++ -> 일반 -> 추가 포함 디렉터리`로 이동하여 드롭다운 상자에서 `<편집...>`을 누르세요.

![](/images/vs_cpp_general.png)

Vulkan, GLFW, GLM의 헤더 디렉터리를 추가하세요:

![](/images/vs_include_dirs.png)

다음으로, `링커 -> 일반` 아래에서 라이브러리 디렉터리 편집기를 엽니다:

![](/images/vs_link_settings.png)

그리고 Vulkan과 GLFW의 오브젝트 파일 위치를 추가하세요:

![](/images/vs_link_dirs.png)

`링커 -> 입력`으로 이동하여 `추가 종속성` 드롭다운 상자에서 `<편집...>`을 누르세요.

![](/images/vs_link_input.png)

Vulkan과 GLFW 오브젝트 파일의 이름을 입력하세요:

![](/images/vs_dependencies.png)

그리고 마지막으로 컴파일러가 C++17 기능을 지원하도록 변경하세요:

![](/images/vs_cpp17.png)

이제 프로젝트 속성 대화 상자를 닫아도 됩니다. 모든 것을 올바르게 했다면 코드에서 더 이상 오류가 강조 표시되지 않을 것입니다.

마지막으로, 실제로 64비트 모드로 컴파일하고 있는지 확인하세요:

![](/images/vs_build_mode.png)

`F5`를 눌러 프로젝트를 컴파일하고 실행하면 다음과 같이 명령 프롬프트와 창이 나타납니다:

![](/images/vs_test_window.png)

익스텐션의 개수는 0이 아니어야 합니다. 축하합니다, 이제 [Vulkan과 함께할](!ko/Drawing_a_triangle/Setup/Base_code) 준비가 모두 끝났습니다!

## Linux

이 설명서는 Ubuntu, Fedora, Arch Linux 사용자를 대상으로 하지만, 패키지 관리자별 명령을 자신의 배포판에 맞는 것으로 변경하여 따라 할 수 있습니다. C++17을 지원하는 컴파일러(GCC 7+ 또는 Clang 5+)가 필요하며, `make`도 필요합니다.

### Vulkan 패키지

Linux에서 Vulkan 애플리케이션을 개발하는 데 필요한 가장 중요한 구성 요소는 Vulkan 로더, 유효성 검사 레이어, 그리고 여러분의 컴퓨터가 Vulkan을 지원하는지 테스트하기 위한 몇 가지 명령줄 유틸리티입니다:

*   `sudo apt install vulkan-tools` 또는 `sudo dnf install vulkan-tools`: 명령줄 유틸리티로, 가장 중요한 `vulkaninfo`와 `vkcube`를 포함합니다. 이들을 실행하여 여러분의 컴퓨터가 Vulkan을 지원하는지 확인하세요.
*   `sudo apt install libvulkan-dev` 또는 `sudo dnf install vulkan-loader-devel` : Vulkan 로더를 설치합니다. 로더는 런타임에 드라이버에서 함수를 찾는데, OpenGL의 GLEW에 익숙하다면 이와 유사한 방식이라고 생각하면 됩니다.
*   `sudo apt install vulkan-validationlayers spirv-tools` 또는 `sudo dnf install mesa-vulkan-drivers vulkan-validation-layers-devel`: 표준 유효성 검사 레이어와 필요한 SPIR-V 도구를 설치합니다. 이들은 Vulkan 애플리케이션을 디버깅할 때 매우 중요하며, 다음 장에서 논의할 것입니다.

Arch Linux에서는 `sudo pacman -S vulkan-devel`을 실행하여 위의 모든 필수 도구를 설치할 수 있습니다.

설치가 성공적이었다면 Vulkan 관련 부분은 모두 준비된 것입니다. `vkcube`를 실행하여 다음과 같은 창이 나타나는지 확인하세요:

![](/images/cube_demo_nowindow.png)

오류 메시지가 나타나면 드라이버가 최신 버전인지, Vulkan 런타임을 포함하고 있는지, 그리고 그래픽 카드가 지원되는지 확인하세요. 주요 공급업체의 드라이버 링크는 [소개 장](!ko/Introduction)을 참조하세요.

### X 윈도우 시스템과 XFree86-VidModeExtension
이 라이브러리들이 시스템에 없을 수도 있습니다. 없다면 다음 명령을 사용하여 설치할 수 있습니다:
*   `sudo apt install libxxf86vm-dev` 또는 `dnf install libXxf86vm-devel`: XFree86-VidModeExtension에 대한 인터페이스를 제공합니다.
*   `sudo apt install libxi-dev` 또는 `dnf install libXi-devel`: XINPUT 확장에 대한 X 윈도우 시스템 클라이언트 인터페이스를 제공합니다.

### GLFW

앞서 언급했듯이, Vulkan 자체는 플랫폼에 구애받지 않는 API이며 렌더링 결과를 표시할 창을 만드는 도구를 포함하지 않습니다. Vulkan의 크로스 플랫폼 이점을 활용하고 X11의 끔찍함을 피하기 위해, 우리는 [GLFW 라이브러리](http://www.glfw.org/)를 사용하여 창을 만들 것입니다. GLFW는 Windows, Linux, MacOS를 지원합니다. [SDL](https://www.libsdl.org/)과 같이 이 목적을 위한 다른 라이브러리도 있지만, GLFW의 장점은 단순히 창 생성 외에도 Vulkan의 다른 플랫폼별 사항들을 추상화해준다는 것입니다.

다음 명령으로 GLFW를 설치할 것입니다:

```bash
sudo apt install libglfw3-dev
```
또는
```bash
sudo dnf install glfw-devel
```
또는
```bash
sudo pacman -S glfw
```

### GLM

DirectX 12와 달리, Vulkan은 선형 대수 연산을 위한 라이브러리를 포함하지 않으므로, 직접 다운로드해야 합니다. [GLM](http://glm.g-truc.net/)은 그래픽 API와 함께 사용하도록 설계된 멋진 라이브러리이며 OpenGL에서도 흔히 사용됩니다.

이것은 헤더 전용 라이브러리로, `libgl