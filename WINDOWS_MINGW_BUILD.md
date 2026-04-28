# libcimbar Windows MinGW 构建指南

本文档记录如何在 Windows 上使用 MinGW-w64 工具链从零构建 libcimbar（包括 OpenCV、GLFW 的源码编译）。

---

## 1. 软件环境

| 软件 | 路径/来源 | 版本 | 说明 |
|------|-----------|------|------|
| CMake | C:\Program Files\CMake\bin\cmake.exe | 4.3.2 | 构建系统 |
| MinGW-w64 | C:\bin\mingw64 | GCC 15.2.0 (x86_64-win32-seh-rev1) | 编译器、make、ar、ranlib |
| Python | 系统自带 | 3.9+ | 用于生成 GLAD 加载器 |

确保 C:\bin\mingw64\bin 在 PATH 中：

```powershell
$env:PATH = "C:\bin\mingw64\bin;" + $env:PATH
```

---

## 2. 依赖项准备

libcimbar 需要 OpenCV、GLFW3、GLES3 头文件。Windows 默认没有这些，需要自行准备。

### 2.1 OpenCV（MinGW 静态编译）

不建议使用 MSVC 预编译包（vc16 等），它与 MinGW ABI 不兼容。必须从源码用 MinGW 编译。

```powershell
cd C:\bin\opencv\sources          # 解压的 OpenCV 源码目录
mkdir build-mingw && cd build-mingw

& "C:\Program Files\CMake\bin\cmake.exe" `
  -G "MinGW Makefiles" `
  -DCMAKE_BUILD_TYPE=Release `
  -DBUILD_SHARED_LIBS=OFF `
  -DBUILD_opencv_apps=OFF `
  -DBUILD_opencv_calib3d=ON `
  -DBUILD_opencv_core=ON `
  -DBUILD_opencv_features2d=ON `
  -DBUILD_opencv_flann=ON `
  -DBUILD_opencv_highgui=OFF `
  -DBUILD_opencv_imgcodecs=ON `
  -DBUILD_opencv_imgproc=ON `
  -DBUILD_opencv_ml=OFF `
  -DBUILD_opencv_objdetect=OFF `
  -DBUILD_opencv_photo=ON `
  -DBUILD_opencv_stitching=OFF `
  -DBUILD_opencv_video=OFF `
  -DBUILD_opencv_videoio=OFF `
  -DBUILD_opencv_world=OFF `
  -DWITH_PROTOBUF=OFF `
  -DWITH_WEBP=OFF `
  -DWITH_OPENEXR=OFF `
  -DWITH_TIFF=OFF `
  -DCMAKE_INSTALL_PREFIX="C:/bin/opencv/install-mingw" `
  ..

mingw32-make -j7
mingw32-make install
```

安装后会得到：
- 头文件：C:\bin\opencv\install-mingw\include
- 静态库：C:\bin\opencv\install-mingw\x64\mingw\staticlib\libopencv_*.a
- CMake 配置：C:\bin\opencv\install-mingw\x64\mingw\staticlib\OpenCVConfig.cmake

注意：如果导出目标中引用了 libade.a 但实际未安装，需要在编译时确保 ade 模块被正确构建，或从 OpenCVModules.cmake 中手动移除 libade.a 引用。

### 2.2 GLFW3（MinGW 静态编译）

```powershell
cd <glfw-source-dir>               # 例如 glfw-3.4/
mkdir build-mingw && cd build-mingw

& "C:\Program Files\CMake\bin\cmake.exe" `
  -G "MinGW Makefiles" `
  -DCMAKE_BUILD_TYPE=Release `
  -DBUILD_SHARED_LIBS=OFF `
  -DCMAKE_INSTALL_PREFIX="$PWD/../install-mingw" `
  ..

mingw32-make -j7
mingw32-make install
```

将安装目录复制/移动到项目目录下，方便相对路径引用：

```
<libcimbar-project>/glfw-3.4/install-mingw/
  include/GLFW/glfw3.h
  lib/libglfw3.a
```

### 2.3 GLES3 头文件

libcimbar 使用 <GLES3/gl3.h>，Windows/MinGW 默认没有。需要下载 Khronos 官方头文件：

```powershell
cd <libcimbar-project>
mkdir third_party_gles_headers
```

下载并解压 Khronos OpenGL-Registry 的 GLES 头文件到 third_party_gles_headers/，最终应包含：
- third_party_gles_headers/GLES3/gl3.h
- third_party_gles_headers/GLES2/gl2ext.h
- third_party_gles_headers/KHR/khrplatform.h

### 2.4 GLAD 加载器（Windows 必需）

Windows 的 opengl32.dll 仅导出 OpenGL 1.1 函数，现代 OpenGL/GLES 函数需要通过加载器获取。使用 GLAD2 生成一个最小 GLES3 加载器：

```powershell
python -m pip install glad2
python -m glad --api gles2=3.0 --out-path src/third_party_lib/glad c
```

这会生成：

```
src/third_party_lib/glad/
  include/glad/gles2.h
  src/gles2.c
```

---

## 3. 源码修改（平台适配）

### 3.1 顶层 CMakeLists.txt

添加 MinGW 专属配置（静态运行时、LTO 工具链、GLES/GLFW 路径）：

```cmake
# MinGW 静态运行时 + LTO 支持
if(MINGW)
    set(CMAKE_AR "gcc-ar" CACHE FILEPATH "Archiver" FORCE)
    set(CMAKE_RANLIB "gcc-ranlib" CACHE FILEPATH "Ranlib" FORCE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libgcc -static-libstdc++")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++ -static")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -static-libgcc -static-libstdc++ -static")
    set(CPPFILESYSTEM "")
endif()

# OpenCV（MinGW 静态 build）
set(OpenCV_DIR "C:/bin/opencv/install-mingw/x64/mingw/staticlib")

# 平台相关的库名
if(WIN32)
    set(OPENGL_LIB "opengl32")
    set(GLFW_LIB "glfw3")
else()
    set(OPENGL_LIB "GL")
    set(GLFW_LIB "glfw")
endif()

# 本地依赖路径
if(MINGW AND EXISTS "${libcimbar_SOURCE_DIR}/third_party_gles_headers")
    include_directories("${libcimbar_SOURCE_DIR}/third_party_gles_headers")
endif()

include_directories(
    ${libcimbar_SOURCE_DIR}/src/lib
    ${libcimbar_SOURCE_DIR}/src/third_party_lib
    "${libcimbar_SOURCE_DIR}/glfw-3.4/install-mingw/include"
)
link_directories(
    "${libcimbar_SOURCE_DIR}/glfw-3.4/install-mingw/lib"
)
```

### 3.2 GUI 子目录（glad 集成）

src/lib/gui/CMakeLists.txt：在 Windows 下将 gui 从 INTERFACE 改为 STATIC，编译 glad 的 gles2.c。

```cmake
if(WIN32)
    add_library(gui STATIC)
    target_sources(gui PRIVATE
        ${libcimbar_SOURCE_DIR}/src/third_party_lib/glad/src/gles2.c
    )
    target_include_directories(gui PUBLIC
        ${libcimbar_SOURCE_DIR}/src/third_party_lib/glad/include
    )
else()
    add_library(gui INTERFACE)
endif()
```

### 3.3 GUI 头文件（条件包含 glad）

所有包含 <GLES3/gl3.h> 的头文件需要改为在 Windows 上使用 glad：

```cpp
#ifdef _WIN32
#include <glad/gles2.h>
#else
#include <GLES3/gl3.h>
#include <GLES2/gl2ext.h>
#endif
```

涉及文件：
- src/lib/gui/gl_shader.h
- src/lib/gui/gl_program.h
- src/lib/gui/gl_2d_display.h
- src/lib/gui/mat_to_gl.h

### 3.4 window_glfw.h（初始化 glad）

在 glfwMakeContextCurrent(_w) 之后添加：

```cpp
#ifdef _WIN32
    if (!gladLoadGLES2((GLADloadfunc)glfwGetProcAddress))
    {
        _good = false;
        return;
    }
#endif
```

### 3.5 OpenGL / GLFW 链接目标

修改以下 CMakeLists.txt，将 GL 替换为 ${OPENGL_LIB}，glfw 替换为 ${GLFW_LIB}：
- src/lib/cimbar_js/CMakeLists.txt
- src/exe/cimbar_recv/CMakeLists.txt
- src/exe/cimbar_recv2/CMakeLists.txt

### 3.6 链接 gui 目标

cimbar_js、cimbar_recv、cimbar_recv2 需要链接 gui 目标（以继承 glad 的 include 路径和对象文件）：

```cmake
target_link_libraries(cimbar_js ... gui ...)
target_link_libraries(cimbar_recv ... gui ...)
target_link_libraries(cimbar_recv2 ... gui ...)
```

### 3.7 std::filesystem 路径转换

MinGW 的 libstdc++ 不支持 std::filesystem::path 隐式转 std::string。所有 tempdir.path() / "..." 赋值给 std::string 的地方需要显式 .string()：

涉及文件：
- src/lib/extractor/test/ExtractorTest.cpp
- src/lib/compression/test/zstd_decompressorTest.cpp
- src/lib/fountain/test/fountain_sinkTest.cpp
- src/lib/fountain/test/fountain_sinkSpecialTest.cpp
- src/lib/encoder/test/DecoderTest.cpp
- src/lib/encoder/test/EncoderTest.cpp
- src/lib/encoder/test/EncoderRoundTripTest.cpp

---

## 4. 构建步骤

### 4.1 配置

```powershell
cd <libcimbar-project>
mkdir build-mingw
cd build-mingw

& "C:\Program Files\CMake\bin\cmake.exe" `
  -G "MinGW Makefiles" `
  -DCMAKE_BUILD_TYPE=RelWithDebInfo `
  -DOpenCV_DIR="C:/bin/opencv/install-mingw/x64/mingw/staticlib" `
  ..
```

### 4.2 编译

```powershell
mingw32-make -j7
```

### 4.3 收集可执行文件

所有输出位于 build-mingw/build/src/ 下：
- 主程序：build-mingw/build/src/exe/*/*.exe
- 测试：build-mingw/build/src/lib/*/test/*.exe

---

## 5. 验证静态链接

使用 objdump 检查 MinGW 运行时 DLL 依赖：

```powershell
$env:PATH = "C:\bin\mingw64\bin;" + $env:PATH
objdump -p cimbar.exe | Select-String "DLL Name"
```

正确输出应仅包含 Windows 系统 DLL，例如：

```
DLL Name: KERNEL32.dll
DLL Name: msvcrt.dll
```

不应出现：
- libgcc_s_seh-1.dll
- libstdc++-6.dll
- libwinpthread-1.dll

如果仍有上述 DLL，检查 CMakeLists.txt 中的 -static 链接器标志是否正确设置。

---

## 6. 已知警告（非致命）

| 警告来源 | 内容 | 影响 |
|----------|------|------|
| intx.hpp | C++23 deprecated literal operator (operator"" _u256) | 可忽略 |
| base91/base.hpp | unused function | 可忽略 |
| wirehair/gf256.cpp | implicit-fallthrough | 可忽略 |
| libcorrect/polynomial.c | stringop-overflow | 可忽略 |

---

## 7. 测试

运行不依赖样本图片的测试：

```powershell
.\build\src\lib\compression\test\compression_test.exe
.\build\src\lib\fountain\test\fountain_test.exe
.\build\src\lib\bit_file\test\bit_file_test.exe
.\build\src\lib\image_hash\test\image_hash_test.exe
```

注意：encoder_test、extractor_test 等需要 samples/ 目录下的测试图片（Git LFS），如果 samples/ 为空，这些测试会因 imread 失败而报错。这不代表构建失败。
