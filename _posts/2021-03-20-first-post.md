---
title: "Getting started OpenGL ES with EGL"
date: 2021-03-20 18:38:28 -0400
categories: jekyll update
---
# Getting started EGL

## What is EGL ?

`EGL`이란 `native window system`과 `grapic API`를 연결해주는 그래픽 인터페이스를 의미

**ref : https://docs.imgtec.com/OpenGLES_HelloAPI_Guide/OpenGLES_Introduction/topics/createEGLDisplay.html **

`EGL`은 `display` 개념을 사용한다. 이것은 렌더링 된 그래픽을 출력하여 보여줄 추상 객체를 의미한다. 주어진 윈도우 시스템에 대한 기본 디스플레이를 작성한 후 `EGL`은 이 핸들을 사용하여 렌더링에 사용할 `EGLDisplay` 핸들을 가져올 수 있다.

`EGL Configuration`은 어플리케이션에 필요한 기능과 그리기에 사용할 수 있는 표면 유형을 의미한다.

`Configuration`의 `attribute`에 대한 내용은 https://www.khronos.org/registry/EGL/sdk/docs/man/html/eglChooseConfig.xhtml 참고

`EGL Surface`는 `OpenGL ES`가 렌더링 할 수 있는 오브젝트를 의미한다.

`EGL`에는 세 가지 주요 `surface` 유형이 있으며, 이는 모두 동일한 방식으로 사용할 수 있지만 약간 다르게 동작한다.

- `Window Surfaces `:` native window`에 생성 및 그려진다
- `Pixmap Surfaces `:` native window`에서 생성되지만 화면에 표시되지 않는다
- `PBuffer Surfcaes` : `EGL` 내에서 직접 생성되며 화면에 표시되지 않는다



## OpenGL ES 동작 단계

`OpenGL ES` 동작 단계를 설명하기 앞서 이 과정에 사용될 예제는 `GLES`의 `hello world`라고 볼 수 있는 삼각형 렌더링을 위한 예제임을 참고하길 바란다.

`OpenGL ES` 동작 단계는 크게 초기화, 렌더링, 릴리즈로 구성되어 있다. `OpenGL ES` 초기화 과정은 아래와 같다.

  - `API` 초기화, `display` 및 `surface` 와 같은 화면 렌더링에 필요한 기본 요소, 정점 버퍼 및 셰이더와 같은 객체 별 렌더링 구조 초기화
  - 초기화 과정은 `EGL API `초기화와 `OpenGL ES` 초기화 과정으로 구성된다

**EGL 초기화 과정**

  - `display` 생성. 대부분 실제 화면에 해당.
  - `EGL Configuration `선택. 작성할 어플리케이션에 맞는 `EGL Configuration`을 선택하기 위한 `attribute를` 생성하여 설정해준다.
  - `Surface` 생성. 화면에 랜더링 될 객체에 해당되는 Surface를 생성
  - `EGL Context `작성. GL 프로그램의 전역 상태를 유지하는데 사용

**OpenGL ES 초기화 과정**

1. 객체의 `vertex` 데이터를 저장할 `vertex `버퍼를 초기화. 해당 텍스처 좌표와 함께 초기 `vertex` 좌표는 모두 버퍼가 생성될 때 버퍼에 삽입된다
   - `Vertex` - `OpenGL ES`가 렌더링 객체를 그릴 위치와 더 근본적인 그 모양에 대한 데이터를 `vertex`라 의미한다. 3D 공간상 객체가 그려질 좌표라고 표현할 수 있다
   - `Vertex Buffer` - `OpenGL`이 `GPU`를 통해 렌더링하기 위해 `GPU`에 사용할 메모리에 연결된 버퍼를 의미
   - `Texture` ? `OpenGL ES`가 화면에 객체를 그릴 때, 객체를 구성하는 폴리곤에 대응되는 각 픽셀에 대한 색상 값을 지정해야한다.(When OpenGL ES draws a polygon to the screen, it must be given colour values for each pixel the polygon covers.) 이러한 작업을 텍스처링이라고 하며, 다각형에 색상을 지정하는데 사용되는 데이터를 텍스처(Texture)라고 말한다.

2. vertex 와 fragment 셰이더 객체 생성. 생성된 객체는 셰이더 프로그램과 함께 바인딩 됨
   - `Shader` - `OpenGL ES 2.0` 부터 도입되었으며, 화면에서 정점이 변환되는 방식과 사용되는 데이터, 화면의 각 픽셀에 색상이 지정되는 방식을 프로그래밍 방식으로 정의하는 프로그래밍 기법이다.
   - `Vertex Shader` - 3D 공간에서 방향을 맞추기 위해 정점 집합을 변환 할 수있는 셰이더이며, 데이터를 보간하여 `Fragment Shader` 로 전달한다. 이 가이드에서 소개한 간단한 예제는 객체의 정점을 객체 공간에서 화면 공간으로 변환하고 텍스처 좌표를 따라 전달하여 색상을 결정합니다.
   - `Fragment  Shader` - 프레임 버퍼에 그릴 때 최종 픽셀의 색상을 결정하는 파이프라인의 일부를 말한다.

3. 셰이더 객체에 대한 매개 변수를 설정. 설정된 매개 변수를 통해 객체를 조정한다.



### 1. Initialize the EGL & OpenGL ES

#### 1-1 EGL initialize

1. Creating an EGL Display

`EGL Display`는 `eglGetDisplay`를 호출하고 기본 디스플레이를 전달받아 간단하게 만들 수 있다. 여기에 되시된 기본 디스플레이는 이미 플랫폼 별 래퍼 코드로 생성되어 있다.

```cpp
_eglDisplay = eglGetDisplay((EGLNativeDisplayType)_surfaceData.deviceContext);
```

만약 위 코드에서 디스플레이 객체를 받아오지 못하면 ( `EGL_NO_DISPLAY` ) 아래 코드와 같이 기본 디스플레이에 대한 액세스를 제공 받을 수 있다. `eglGetDisplay`로 생성된 객체는 이후 `_eglDisplay` 핸들로 관리하게 된다.

```cpp
 if (_eglDisplay == EGL_NO_DISPLAY)
 {
 	_eglDisplay = eglGetDisplay((EGLNativeDisplayType)EGL_DEFAULT_DISPLAY);
 } 
```

위 코드로도 디스플레이를 받을 수 없다면 아래와 같이 오류를 반환해야 한다.

```cpp
 if (_eglDisplay == EGL_NO_DISPLAY)
 {
 	Log(true, "Failed to get an EGLDisplay");
 	return false;
 } 	
```

`EGL API`는 위 과정을 통해 얻은 디스플레이로 초기화해야한다. 

`eglGetDisplay` 및 `eglGetError`를 제외한 모든 `EGL` 함수에는 초기화 된 `EGLDisplay`가 필요하다. 

작성 중인 어플리케이션이 EGL 버전과 상관없는 경우 `elgInitialize` 함수의 두 번째 및 세 번째 매개 변수에 대해 `NULL`을 전달할 수 있다.

```cpp
 EGLint eglMajorVersion = 0;
 EGLint eglMinorVersion = 0;
 if (!eglInitialize(_eglDisplay, &eglMajorVersion, &eglMinorVersion))
 {
 	Log(true, "Failed to initialize the EGLDisplay");
 	return false;
 } 
```

`EGL`은 `OpenGL ES 외`에 다른 `API( ex : OpenVG, OpenCL)`과 상호 작용할 수 있다. `eglBindAPI`는 나중에 사용할 `API`를 정의하는 것을 의미한다. ( 생략 가능 )

```cpp
 int result = EGL_FALSE;
 
 result = eglBindAPI(EGL_OPENGL_ES_API);
 
 if (result != EGL_TRUE)
 {
 	return false;
 }
 
 return true;
```



2. Creating an EGL Configuration

가장 먼저 `EGL`에 사용될 속성 배열을 만들어 필요한 환경 구성을 지정한다. 이는 요청할 특정 기능을 설명하는 키/값 쌍의 배열을 의미한다. 예제에서는 특별한 것이 필요하지 않으므로 최소한의 속성 배열을 구성하여 쿼리한다. 

속성 배열의 마지막은 `EGL_NONE` 값으로 표시한다.

```cpp
 const EGLint configurationAttributes[] = { EGL_SURFACE_TYPE, EGL_WINDOW_BIT, EGL_RENDERABLE_TYPE, EGL_OPENGL_ES2_BIT, EGL_NONE };
 
```

속성 구성을 완료하였다면 `eglChooseConfig`를 통해 지정한 속성과 일치하는 `EGL` 프레임 버퍼 구성 목록을 리턴받는다. 

`EGL Configuration list`에 대한 자세한 정보가 필요하다면 `khronos` 홈페이지를 참고하도록 한다. ( ex : https://www.khronos.org/registry/EGL/specs/eglspec.1.5.pdf, https://www.khronos.org/registry/EGL/sdk/docs/man/html/eglChooseConfig.xhtml )

```cpp
 EGLint configsReturned;
 if (!eglChooseConfig(_eglDisplay, configurationAttributes, &config, 1, &configsReturned) || (configsReturned != 1))
 {
 	Log(true, "eglChooseConfig() failed.");
 	return false;
 }
 return true;
```



3. Creating an EGL Surface

`eglCreateWindowSurfcae`를 통해 간단히 `surface` 객체를 생성할 수 있다. 위 과정에서 작성된 `EGLDisplay`와 `EGLConfig` 및 `native window`를 전달하여 주면 된다. 마지막 인자인 `NULL`은 `surfcae`에 대한 속성 목록을 의미하며, 필요에 따라 해당 속성 값을 부여할 수 있다. 생성된 `surface` 객체는 `_eglSurface` 핸들을 통해 관리하게 된다.

```cpp
 _eglSurface = eglCreateWindowSurface(_eglDisplay, config, _surfaceData.window, NULL);
 	
```

만약 위 코드가 실패할 경우 `native window`를 제외시킨 뒤 시도할 수 있다.

```cpp
 if (_eglSurface == EGL_NO_SURFACE)
 {
 	eglGetError(); // Clear error
 	_eglSurface = eglCreateWindowSurface(_eglDisplay, config, NULL, NULL);
 }
```



4. Creating an EGL Context

`eglBindAPI`를 통해 `EGL API`에 사용할 그래픽 `API`에 대해 알려준다. ( 생략 가능 )

```cpp
eglBindAPI(EGL_OPENGL_ES_API);
if (!testEGLError(_surfaceData, "eglBindAPI"))
{
 	return false;
}
```

그 다음 앞서 `display`와 `surface` 생성 때와 마찬가지로 `EGL Context`에 사용할 그래픽 `API`에 대한 속성 배열을 작성한다.

```cpp
EGLint contextAttributes[] = { EGL_CONTEXT_CLIENT_VERSION, 2, EGL_NONE };
```

`context` 속성 배열을 작성한 뒤 `eglCreateContext`를 통해 `eglContext`를 생성하고 오류 검사를 하도록 한다. `testEGLError`는 이미지네이션에서 제공하는 예제의 에러 검사 함수이다.

```cpp
context = eglCreateContext(_eglDisplay, config, NULL, contextAttributes);
if (!testEGLError(_surfaceData, "eglCreateContext"))
{
 	return false;
}
```

`OpenGL ES`가 전역 함수를 사용하는 방식으로 인해 모든 함수 호출이 올바른 컨텍스트에서 작동할 수 있도록 컨텍스트를 항상 최신 상태로 만들어야 한다.

`eglMakeCurrent`는 컨텍스트를 호출 된 현재 렌더링 스레드에 바인딩하는 함수이다.

호출 스레드에 이미 현재 렌더링 컨텍스트가 있는 경우 해당 컨텍스트가 끝나고(`flush`)  `eglMakeCurrent`에 의해 새로운 컨텍스트를 가리키게 한다.

동시에 여러 컨텍스트를 사용하려면 여러 스레드를 사용하고 스레드간에 동기화를 하도록 한다. 이 함수는 컨텍스트를 `draw surface`와 `read surface`에 바인딩한다. 이렇게하면 올바른 표면이 렌더링 된다.  아래 예는 `draw surface`와 `read surface`가 동일하기에 두 `surface` 모두 `_eglSurface`로 넘겨주고 있다.

```cpp
eglMakeCurrent(_eglDisplay, _eglSurface, _eglSurface, context);
```



#### 1-2 OpenGL ES initialize

`OpenGL ES`에서 사용되는 `vertex` 데이터와 `vertex, fragment` 셰이더, 텍스처에 대한 초기화를 진행한다

1. Creating a Vertex data
   `vertex data`는 `vertex buffer`에 로드 될 값으로, 위치 좌표를 위한 3개의 float( x, y, z )와 텍스처 좌표 2개 ( u, v )로 구성된다. 텍스처 좌표는 각 정점에서 픽셀이 어떤 색상이 될지 결정하기 위해 샘플링되는 텍스처 이미지의 좌표를 의미한다. 중요한 것은 정점 사이를 보간하고 삼각형 내부의 모든 픽셀의 텍스처 좌표를 찾는 앵커(`anchor`) 포인트로 사용된다는 것이다. 이러한 방식으로 텍스처의 이미지가 삼각형에 매핑되는 방식을 결정하게 된다.

   아래는  `vertex data`를 직접 선언 및 초기화 한 예이다.

`GLfloat` 타입으로 배열을 선언, 각 정점을 구성할 `vertex`와 `texture`의 좌표들로 초기화한다.

```cpp
   GLfloat vertexData [] = { -0.4f , -0.4f , 0.0f ,  // first vertex point
   						0.0f , 0.0f , 		   // first texture point
   						0.4f , -0.4f , 0.0f ,  // second vertex point
   						1.0f , 0.0f , 		  // second texture point
   						0.0f , 0.4f , 0.0f ,  // third vertex point
   						0.5f , 1.0f };		 // third texture point
```

   `vertex data`가 준비되었다면 GL에 바인딩해줄 버퍼 객체를 생성해준다.생성된 버퍼 객체는 `glGenBuffers`로 `GL`에 버퍼 객체임을 알려주고 `glBindBuffer`를 통해 `GL`에 바인딩해줄 수 있다.

   ```cpp
   GLuint * _vertexBuffer;
   
   glGenBuffers ( 1 , & _vertexBuffer);
   glBindBuffer (GL_ARRAY_BUFFER, _vertexBuffer);
   ```

   glbindBuffer의 첫번째 변수는 바인딩 될 객체에 대한 타입을 지정하는 것으로  `GL_ARRAY_BUFFER, GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, GL_ELEMENT_ARRAY_BUFFER, GL_PIXEL_PACK_BUFFER, GL_PIXEL_UNPACK_BUFFER, GL_UNIFORM_BUFFER` 또는 `GL_UNIFORM_BUFFER, GL_COPY_READ_BUFFER` 와 같이 다양한 타입이 존재한다.

   각 타입에 대한 자세한 내용은 https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glBindBuffer.xhtml 을 참고하면 된다.


   GL에 버퍼 객체를 바인딩 했다면 버퍼의 크기와 사용량을 설정하고 버퍼에 데이터를 로드해주도록 한다.

    	glBufferData (GL_ARRAY_BUFFER, sizeof (vertexData), vertexData, GL_STATIC_DRAW);

  


2. Initialized Shader

   `OpenGL`에서는 화면에 결과물을 출력하는 과정에서 `vertex shader`와 `fragment shader`를 사용한다..

   

   `Vertex Shader`는 `vertex` 정보를 디스플레이하기 위한 과정으로 `GL` 그래픽 파이프라인의 `Vertex Specification` 단계에서 작성한 `vetex` 집합들을 2D 화면에 맞춰 배치해주는 과정을 의미한다. 이 과정에서 `3D World Coordinates` 를 `2D display coordinates`로 변경해준다. ( 즉, 3D 객체를 2D 화면으로 보일 수 있도록 변경 )


   Fragment Shader는 래스터 변환 단계를 거친 객체에 대해 색 정보 혹은 질감을 적용해주는 단계이다.

   

   **Initialized Fragment Shader**

   아래는 GLSL ( `GL Shading Laguage ES` ) 을 이용한 `fragment shader` 작성을 하는 간단한 예이다. 2D 텍스처와 텍스처 좌표 쌍 ( `textCoords` )의 입력만을 가지고 있는 것을 볼 수 있다. 텍스처 좌표는 이전에 생성 된 `vetex buffer`에 의해 제공된다. 셰이더는 제공된 텍스처 좌표에서 텍스처를 샘플링하여 현재 `fragmen`  (`gl_fragColor`)의 색상을 결정한다.
   이 셰이더는 `GPU`의 병렬화된 특성을 사용하여 많은 작업을 동시에 수행함으로써 `fragment`의 색상을 매우 빠르게 계산한다. 

   ```cpp
   const  char * const fragmentShaderSource = "\
   										 uniform sampler2D texture; \ 
   										 \ 
   										 varying mediump vec2 texCoords; \ 
   										 void main ( void )\ 
   										 {\ 
   											gl_FragColor = texture2D (texture, texCoords); \ 
   										 } ";
   ```

   아래 `glCreateShader`를 통해 `fragment shader` 핸들러를 생성한다.

    	_fragmentShader = glCreateShader (GL_FRAGMENT_SHADER);

   그런 다음 미리 작성된 `fragment shader` 코드를 셰이더에 로드해준다.

   ```cpp
   glShaderSource (_fragmentShader, 1 , ( const  char **) & fragmentShaderSource, NULL);
   ```

   `glCompileShader` 함수를 통해 `fragment shader`를 컴파일하고, 

    	glCompileShader (_fragmentShader);

   `glGetShaderiv` 함수를 통해 셰이더 유형 또는 소스코드 길이 등의 정보를 반환받아 제대로 컴파일 되었는지 확인할 수 있다.

    	GLint isShaderCompiled;
    	glGetShaderiv (_fragmentShader, GL_COMPILE_STATUS, & isShaderCompiled);

   


   **Initialized Vertex Shader**

   `Vertex Shader`또한 `fragment shader`와 동일한 방법으로 초기화를 진행하면 된다.

   아래는 `vertex shader`에 대한 예제이며 입력은 vertex 좌표와 ( `myVertex` ) , 텍스처 좌표 ( `myTexCoords` ) 및 변환 행렬 ( `transformationMatrix` ) 로 구성되어있다.

   변환 행렬과 vertex 좌표를 곱하여 정점 위치를 화면 공간으로 변환해준다. 이는 셰이더에 의해 출력된 정점 좌표가 화면의 크기와 관련하여 정의된다는 것을 의미한다. 이를 수행하는데 사용되는 변환 행렬은 매 프레임을 그려줄 draw function (사용자 정의)  에서 계산된다.

   텍스처 좌표는 가변 변수 (`texCoords`)를 사용하여 수정되지 않은 상태에서 전달된다. 다양한 변수는 렌더링되는 전체 프리미티브(`primitive`)에 걸쳐 보간된다. 

   ```cpp
   const  char * const vertexShaderSource = "\
   									   attribute highp vec4 myVertex; \ 
   									   attribute mediump vec2 myTexCoords; \ 
   									   \ 
   									   uniform highp mat4 transformationMatrix; \ 
   									   \ 
   									   varying mediump vec2 texCoords; \ void main ( void ) \ 
   									   {\ 
   									   texCoords = myTexCoords; \ 
   									   gl_Position = transformationMatrix * myVertex; \ 
   									   } ";
   ```

   셰이더 코드가 작성되었다면 이전 `fragment shader`와 같이 `shader`를 생성하여 셰이더에 대한 핸들러를 얻고 셰이더에 코드를 로드, 컴파일하는 과정을 거치면 된다.


   ```cpp
   _vertexShader = glCreateShader (GL_VERTEX_SHADER);
   
   glShaderSource (_vertexShader, 1 , ( const  char **) & vertexShaderSource, NULL);
   glCompileShader (_vertexShader);
   
   glGetShaderiv (_vertexShader, GL_COMPILE_STATUS, & isShaderCompiled);
   ```

 

  **Load Shader Program**
   컴파일 된 셰이더는 렌더링에 사용되기 전 셰이더 프로그램에 연결되어야한다.


   아래 코드와 같이 `glCreateProgram` 함수를 통해 셰이더 프로그램을 생성해주고, 셰이더 코드들을 연결해주어 GL에서 사용될 셰이더 프로그램에 대해 정의해주게 된다.

   ```cpp
   _shaderProgram = glCreateProgram (); 
   glAttachShader (_shaderProgram, _fragmentShader);
   glAttachShader (_shaderProgram, _vertexShader);
   ```

   그 다음 `glBindAttribLocation` 함수를 통해 셰이더 코드에서 선언하였던 매개 변수들을 `GL application`에서 사용할 수 있도록 연결시켜준다. 

    	glBindAttribLocation (_shaderProgram, _vertexArray, "myVertex" );
    	glBindAttribLocation (_shaderProgram, _textureArray, "myTextureCoords" );


   위 코드를 통해 셰이더 프로그램의 변수와 연결된 `_vertexArray`와 `_textureArray`는 `GL application` 코드에서 셰이더 프로그램의 변수 값을 임의로 변경할 수 있는 역할을 해준다.

   셰이더 프로그램에 대한 정의 및 변수 바인딩이 완료되었다면 `glLinkProgram`을 통해 `GL application`과 셰이더 프로그램을 연결해주도록 한다.

    	glLinkProgram (_shaderProgram);

   앞서 셰이더 코드들이 제대로 연결되었는지 확인했던 것과 같이 셰이더 프로그램 또한 `glGetProgramiv` 함수를 통해 연결 여부를 확인할 수 있다.

    	GLint isLinked;
    	glGetProgramiv (_shaderProgram, GL_LINK_STATUS, & isLinked);

   셰이더에 대한 모든 초기화 과정이 끝났으므로 `GL application`에서 셰이더 프로그램을 사용할 것임을 선언해주도록 한다. `glUseProgram`을 호출하면 `GL application`에서 렌더링 시 해당 셰이더 프로그램을 사용할 것임을 GL ES에 알려준다. 

   여기서 중요한 점은 GL ES에서는 한번에 단 하나의 프로그램만 활성화 할 수 있다는 점이다. 만약 다양한 장면을 연출하기 위해 셰이더 프로그램을 여러개 사용할 경우 각 순간에 대한 연산 시 적절히 셰이더 프로그램을 활성/비활성 해 주어야 한다는 점이다.

    	glUseProgram (_shaderProgram);

3. Creating a Texture

   텍스처(`Texture`)는 OpenGL ES가 화면에 객체를 그릴 때, 객체를 구성하는 폴리곤에 대응되는 각 픽셀에 대한 색상 값을 지정해야한다.(When OpenGL ES draws a polygon to the screen, it must be given colour values for each pixel the polygon covers.) 이러한 작업을 텍스처링이라고 하며, 다각형에 색상을 지정하는데 사용되는 데이터를 텍스처라고 말한다.

   

   우선 텍스처를 `GL`에 렌더링하기 위해서 `GL application`이 참조할 텍스처 핸들을 생성한다.

    	glGenTextures ( 1 , & _texture);

   

   생성된 핸들러를 통해 텍스처 대상을 수정할 수 있도록 `glBindTexture` 함수를 통해 어떤 형식의 텍스처를 GL에 연결하여 관리할지를 선언해준다.

   

   그 다음 텍스처에 사용될 데이터를 생성해주도록 한다. 아래는 임의의 `RGBA` 값을 생성해주는 코드이다.

   ```cpp
   for (int x = 0; x < imageWidth; x++)
   {
   	for (int y = 0; y < imageHeight; y++)
   	{
   		int c = 0;
    
   		if (x % 128 < 64 && y % 128 < 64)
   		{
   			c = 255;
   		}
   		if (x % 128 >= 64 && y % 128 >= 64)
   		{
   			c = 255;
   		}
    
   		_image[x][y][0] = c;
   		_image[x][y][1] = c;
   		_image[x][y][2] = c;
   		_image[x][y][3] = 255;
   	}
   }
   ```

   `glTexParameteri`함수는 차원 기하에 필요한 화소를 2차원 이미지에서 샘플링하는 방법을 설정하는 함수이다. ( ref :  http://docs.gl/es2/glTexParameter )

   `glTexParameteri` 에서 지정할 수 있는 샘플링 기법은 `GL_TEXTURE_MIN_FILTER`, `GL_TEXTURE_MAG_FILTER`, `GL_TEXTURE_WRAP_S`, 또는 `GL_TEXTURE_WRAP_T` 로 구성된다.

   

    `GL_TEXTURE_WRAP_S`과 `GL_TEXTURE_WRAP_T`는 2D 이미지가 3D 이미지보다 작을때, 3D 이미지공간의 남은 빈 영역을 어떻게 채울 것인가에 대해 이미지 자체를 반복(`GL_REPEAT`) 또는 이미지 테두리 값 반복(`GL_CLAMP_TO_EDGE`) , 텍스처를 반복하며 좌표의 정수 부분이 홀수 일 때 미러링( `GL_MIRRORED_REPEAT`)하거나 범위를 벗어난 좌표에는 지정된 테두리 색상을 지정하는(`GL_CLAMP_TO_BORDER`) 중 한 가지로 설정할 수 있다.




    	glTexParameteri (GL_TEXTURE_ 2 D, GL_TEXTURE_WRAP_S, GL_REPEAT); 
    	glTexParameteri (GL_TEXTURE_ 2 D, GL_TEXTURE_WRAP_T, GL_REPEAT);

   아래 사진은 Wapping 타입 별 출력 예시이다.

   ![](.\Fig\GL_Warpping.PNG)


   `GL_TEXTURE_MIN_FILTER`과 `GL_TEXTURE_MAG_FILTER`는 텍스처와 해상도가 정확히 일치하지 않을 때 이미지를 어떻게 샘플링할 것인가에 대해 설정해준다. 좌표에서 가장 가까운 픽셀을 반환(`GL_LEAREST`)하여 샘플링하거나 주어진 좌표를 둘러싸고 있는 4픽셀의 평균 가중치를 반환(`GL_LINEAR`)할 수도 있으며, MIPMAP 샘플을 이용할 수도 있다(`GL_NEAREST_MIPMAP_NEAREST`, `GL_LINEAR_MIPMAP_NEAREST`, `GL_NEAREST_MIPMAP_LINEAR`, `GL_LINEAR_MIPMAP_LINEAR`).

    	glTexParameteri (GL_TEXTURE_ 2 D, GL_TEXTURE_MIN_FILTER, GL_LINEAR); 
    	glTexParameteri (GL_TEXTURE_ 2 D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

   <img src=".\Fig\GL_Filtering.PNG" style="zoom:50%;" />

   위 코드를 통해 텍스처 이미지에 대한 샘플링 방법에 대해 지정했다면 텍스처에 매핑될 픽셀 데이터를 로드할 차례이다.

    	glTexImage2D (GL_TEXTURE_ 2D, 0 , GL_RGBA, imageWidth, imageHeight, 0 , GL_RGBA, GL_UNSIGNED_BYTE, _image);

   `glTexImage2D` 함수는 텍스처에 2D 이미지를 지정하는 함수로 원하는 세부 사항에 따라 밉맵 레벨 ( 2번째 매개변수 : 0 ) 이나 이미지 구성 요소의 수 ( width, height ), 색상 ( `GL_RGBA` : RGBA ) 등과 이미지 데이터 ( `_image` ) 를 지정하여 사용할 수 있다.

### 2. Rendering 

길고 긴 초기화 과정을 끝내고 이미지를 출력할 차례가 왔다. 앞서 사용한 예제들은 `GL`의 `hello world`라고 볼 수 있는 삼각형 렌더링을 위한 초기화 과정이었다.

그 과정에서 `vertex data`를 생성하여 삼각형 객체의 좌표 정보를 설정하였고, 셰이더 프로그램을 초기화 및 연결해주어 `vertex data`가 어떻게 그려질지에 대해 프로그래밍 하였으며, 텍스처(`texture`) 데이터를 생성 및 로드하여 `vertex data`에 입혀질 텍스처를 설정해주었다.

다음으로 설명할 과정에서는 삼각형을 그리기 위한 방법에 대해 배워볼 차례이다.

1. 삼각형의 회전 각도를 업데이트 한다.
2. 색상 버퍼의 모든 픽셀을 미리 정의 된 색상으로 지운다.
3. 투영 행렬을 만들어 삼각형의 정점 ( `vertex data` )를 객체 공간에서 화면 공간으로 투영시킨다.
4. 정점 데이터에서 각 정점의 속성을 설정한다. 예를 들어 각 정점의 차원 수 ( x, y, z)와 버퍼에 있는 각 정점의 데이터 크기 등이 있다.
5. 각 정점에서 텍스처 좌표의 속성을 설정한다. 텍스처는 2D 이므로 각 정점 (u, v)에 두 개의 좌표로 구성된다.
6. **삼각형을 그린다**
7. `GLES`에 다음 프레임을 렌더링 할 때 데이터가 필요하지 않으므로 이 프레임 버퍼를 버릴 수 있음을 알려준다. 프레임 시작 시 프레임 버퍼를 지우는 것과 유사하게 프레임 버퍼 데이터를 시스템 메모리로 전송하는데 대역폭이 낭비되지 않도록 하는 작업이다.
8. 렌더링 된 프레임 버퍼를 디스플레이에 표시하기 위해 디스플레이 버퍼 교환을 해준다.
9. 다음 프레임을 시작한다. ( 1단계 부터 같은 과정을 반복하게 된다 )

#### 2-1 Rendering the Screan

앞서 소개한 렌더링 과정 중 1 ~ 7 과정에 해당하는 부분으로 버퍼를 비우고, 정점에 대한 변환 행렬을 계산, 삼각형을 그리는 과정이다.


삼각형을 회전 시키기 위한 변수 `_angle`을 선언하고 매 프레임마다 각도 값이 변화하도록 설정해준다.

```cpp
_angle += 0.01f;
```


그 다음 프레임이 시작될 때 `GLES`에 이전에 그린 모든 작업이 완료되었으며 새 프레임을 그려야함을 알리기 위해 프레임 이미지를 지워주도록 한다. `glClearColor`는 (r, g, b, a) 채널에 대해 0.0 ~ 1.0 사이의 부동소수점 값으로 설정한다. 각 값은 색상 채널의 강도를 의미하며 0.0은 투명한 검정, 1.0은 불투명한 흰색을 의미한다.

 	glClearColor ( 0.00f , 0.70f , 0.67f , 1.0f );

`glClearDepth` 및 `glClearStencil` 함수를 사용하면 응용 프로그램에서 각각 깊이 및 스텐실 값으로 동일한 작업을 수행 할 수 있다.


`glClear`는 색상 버퍼와 함께 색상을 지우는데 사용된다. `Gl_DEPTH_BUFFER_BIT` 또는 `GL_STENCIL_BUFFER_BIT`를 사용하여 깊이 및 스텐실 버퍼를 지우는데 사용할 수도 있다.

 	glClear (GL_COLOR_BUFFER_BIT);


다음으로 `glGetUniformLocation` 함수를 이용해 초기화 과정에서 작성했던 셰이더 프로그램으로부터 셰이더 변수를 가져온다. 변환 매트릭스는 `vertex shader`에 의해 정점의 방향이 어떻게 변경할 것인지를 결정한다.

```cpp
int matrixLocation = glGetUniformLocation (_shaderProgram, "transformationMatrix" );
```

`aspect` 변수는 화면의 종횡비를 나타내기 위한 변수이다. 투영 행렬을 계산하는데 사용할 것이다.

```cpp
float aspect = (_surfaceData.width / _surfaceData .height);
```

다음으로 직교 투영 행렬을 계산한다. 직교 투영이란 원근이 사용되지 않는 단순한 유형의 투여이며, 투영 행렬은 물체의 3D좌표를 화면의 2D 좌표로 변환하여 3D 객체를 화면에 투영할 때 사용된다. 

아래 코드는 직교 투영 계싼을 단순화하기 위해 glm을 사용하였다.

 	glm::mat4 projMatrix = glm::ortho(-aspect, aspect, -1.0f, 1.0f);

변환 행렬은 화면에 투영 될 삼각형의 방향을 지정하는데 사용되는 행렬이다. 회전 행렬과 투영 행렬을 결합한다. 삼각형의 현재 각도 변수는 각 프레임의 시작 부분에 업데이트 되므로 각 프레임마다 회전 행렬을 새로 계산해주어야한다.

일반적으로 변환 행렬은 project * view * translation * rotation * scale 로 구해진다.
하지만 아래 코드에서는 glm을 이용해 단순화하여 표현했다.

 	glm :: mat4 transformationMatrix = projMatrix * glm :: rotate (_angle, glm :: vec3 ( 0.0f , 0.0f , 1.0f ));


계산된 변환 행렬을 셰이더에 반영하기 위해 `glUniformMatrix4fv` 함수를 통해 셰이더에 값을 전달하도록 한다.

 	glUniformMatrix4fv (matrixLocation, 1 , GL_FALSE, glm :: value_ptr (transformationMatrix));

그 다음 셰이더 프로그램으로부터 `glEnableVertexAttribArray` 함수를 이용하여 렌더링에 사용할 정점 배열을 활성화시킨다.  이어서 `glVertexAttribPointer` 함수를 호출해주는데, 이 함수는 `GL_ARRAY_BUFFER` 로 `GL`버퍼에 바인딩 된 데이터 저장소로부터 `vertex`의 크기( 1,2,3,4로 지정할 수 있다 )와 `vertex`의 타입, 그리고 가져올 데이터의 사이즈 등을 지정하여 버퍼 데이터를 `vertex shader`에 넘겨주는 역할을 한다.

앞서 우리가 `GL` 버퍼에 로드한 데이터는 5개의 데이터를 가진 ( vertex 3 point , texture 2 point ) 3개의 정점으로 구성되었으며 `vertex`의 크기는 3, 각 정점의 오프셋은 5 * sizeof (GLfloat), 인덱싱 시작 위치는 0 으로 지정해주었다.

보다 자세한 내용은 https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glVertexAttribPointer.xhtml 를 참고하길 바란다.

 	glEnableVertexAttribArray (_vertexArray);
 	glVertexAttribPointer (_vertexArray, 3 , GL_FLOAT, GL_FALSE, 5 * sizeof (GLfloat), 0 );

똑같은 방식으로 `glEnableVertexAttribArray` 함수를 이용하여 `texture buffer`를 `vertex shader`에 넘겨준다.
이번에는 `GL` 버퍼로부터 `texture data`를 가져와야 하므로 `texture`의 크기는 2, 각 정점의 오프셋은 5 * sizeof (GLfloat)으로 동일하게 적용, 인덱싱 시작 위치는 3 * sizeof (GLfloat)으로 지정해주었다.

 	glEnableVertexAttribArray (_textureArray);
 	glVertexAttribPointer (_textureArray, 2 , GL_FLOAT, GL_FALSE, 5 * sizeof (GLfloat), ( void *) ( 3 * sizeof (GLfloat)));

마지막으로 `GLES`에 렌더링을 하기 위해 `glDrawArrays` 함수를 호출해준다.

`glDrawArrays` 함수를 호출하면 활성화 된 배열로부터 셰이더 프로그래밍으로 지정된 그림을 그리게 된다.

```cpp
glDrawArrays (GL_TRIANGLES, 0 , 3 );
```



#### 2-2 Swapping the Buffers

렌더링의 마지막으로 버퍼 교체 단계가 남았다. 이 단계는 디스플레이 될 데이터를 화면에 표시하는 단계이다.
`OpenGL ES`로 화면에 렌더링할때 일반적으로 이중 또는 삼중 버퍼링을 사용한다. 즉, `OpenGL ES`는 `back buffer`라고 하는 하나의 프레임 버퍼로 직접 렌더링하고, 디스플레이는 다른 프레임 버퍼인 `front buffer`에서 데이터를 읽는다.

`eglSwapBuffers`는 `OpenGL ES`가 `back buffer`로의 렌더링을 완료했음을 `window system`에 알려준다. 따라서 `back buffer`가 `front buffer`로 디스플레이에 표시될 수 있음을 알려주는 것이다. 이와 동시에 `front buffer`는 `back buffer`가 되어 다음 렌더링에 사용하도록 `Swap` 해주는 역할을 한다.

 	eglSwapBuffers (_eglDisplay, _eglSurface)) ;


### 3. Release the EGL & OpenGL ES

이제 모든 과정을 끝내고 할당 된 리소스를 해제할 차례이다. 사용된 `EGL` 객체와 `GLES` 객체를 메모리에서 해제하여 리소스 누수가 발생하지 않게 해주어야한다.

#### 3-1 Releasing EGL Resource

앞서 `GLES`로 렌더링한 객체를 화면에 출력하기 위해 사용된 `EGL Resource`는 `display`와 `context`이므로 두 객체만을 메모리에서 해제해주면 된다. `eglMakeCurrent` 함수와 `eglTerminate` 함수를 통해 더이상 `resource`를 사용하지 않음을 명시하고 종료해주면 된다.

 	eglMakeCurrent (_eglDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
 	eglTerminate (_eglDisplay);



#### 3-2 Releasing GLES Resource

`GLES Resource` 는 셰이더 프로그램에 사용된 `resource`와 렌더링 시 버퍼 핸들러로 사용한 `resource`를 해제해주면 된다.

 	glDeleteShader (_fragmentShader);
 	glDeleteShader (_vertexShader);
 	glDeleteProgram (_shaderProgram);
 	glDeleteBuffers ( 1 , & _vertexBuffer);



## 4. 결론

이 가이드는 OpenGL ES 애플리케이션이 화면에 프리미티브를 렌더링하는 방법, 즉 3D 객체를 화면에 출력하는 방법과 그래픽 프로그래밍 및 OpenGL ES의 기본 개념이 실제로 어떻게 작동하는지에 대해 안내되었다.

아래는 가이드에서 다룬 개념에 대해 간략화한 내용이다.


### 앞서 다룬 OpenGL ES / EGL 개념

- `display` - 렌더링 된 그래픽을 출력하여 보여줄 추상 객체를 의미한다. 주어진 윈도우 시스템에 대한 기본 디스플레이를 작성한 후 `EGL`은 이 핸들을 사용하여 렌더링에 사용할 `EGLDisplay` 핸들을 가져올 수 있다.
- `Surface`는 `OpenGL ES`가 렌더링 할 수 있는 오브젝트를 의미한다.
  `EGL`에는 세 가지 주요 `surface` 유형이 있으며, 이는 모두 동일한 방식으로 사용할 수 있지만 약간 다르게 동작한다.
  - `Window Surfaces `:` native window`에 생성 및 그려진다
  - `Pixmap Surfaces `:` native window`에서 생성되지만 화면에 표시되지 않는다
  - `PBuffer Surfcaes` : `EGL` 내에서 직접 생성되며 화면에 표시되지 않는다
- `Configuration` - 어플리케이션에 필요한 기능과 그리기에 사용할 수 있는 표면 유형을 의미한다.
- `Vertex` - `OpenGL ES`가 렌더링 객체를 그릴 위치와 더 근본적인 그 모양에 대한 데이터를 `vertex`라 의미한다. 3D 공간상 객체가 그려질 좌표라고 표현할 수 있다
- `Vertex Buffer` - `OpenGL`이 `GPU`를 통해 렌더링하기 위해 `GPU`에 사용할 메모리에 연결된 버퍼를 의미
- `Shader` - `OpenGL ES 2.0` 부터 도입되었으며, 화면에서 정점이 변환되는 방식과 사용되는 데이터, 화면의 각 픽셀에 색상이 지정되는 방식을 프로그래밍 방식으로 정의하는 프로그래밍 기법이다.
  - `Vertex Shader` - 3D 공간에서 방향을 맞추기 위해 정점 집합을 변환 할 수있는 셰이더이며, 데이터를 보간하여 `Fragment Shader` 로 전달한다. 이 가이드에서 소개한 간단한 예제는 객체의 정점을 객체 공간에서 화면 공간으로 변환하고 텍스처 좌표를 따라 전달하여 색상을 결정합니다.
  - `Fragment  Shader` - 프레임 버퍼에 그릴 때 최종 픽셀의 색상을 결정하는 파이프라인의 일부를 말한다.
- `Texture` - `OpenGL ES`가 화면에 객체를 그릴 때, 객체를 구성하는 폴리곤에 대응되는 각 픽셀에 대한 색상 값을 지정해야한다.(When OpenGL ES draws a polygon to the screen, it must be given colour values for each pixel the polygon covers.) 이러한 작업을 텍스처링이라고 하며, 다각형에 색상을 지정하는데 사용되는 데이터를 텍스처(Texture)라고 말한다.
