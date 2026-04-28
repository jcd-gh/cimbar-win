# libcimbar Windows MinGW Build Dependencies

This document lists the exact versions of all dependencies used to build libcimbar on Windows with MinGW-w64.

## Build Toolchain

| Dependency | Version | Path / Source | Notes |
|------------|---------|---------------|-------|
| MinGW-w64 GCC | 15.2.0 (x86_64-win32-seh-rev1) | C:\bin\mingw64 | C/C++ compiler, make, ar, ranlib |
| CMake | 4.3.2 | C:\Program Files\CMake | Build system generator |
| Python | 3.9.2rc1 | System bundled | Used for GLAD loader generation |
| Git | 2.46.0.windows.1 | C:\Program Files\Git | Version control |

## Third-Party Libraries

| Library | Version | Source | Build Type | Install Path |
|---------|---------|--------|------------|--------------|
| OpenCV | 4.12.0 | Built from source | Static (.a) | C:\bin\opencv\install-mingw |
| GLFW | 3.4 | Bundled source | Static (.a) | <project>\glfw-3.4\install-mingw |
| GLAD | 2.0.8 | Generated via pip | C loader | <project>\src\third_party_lib\glad |
| Khronos GLES Headers | Latest (Khronos Registry) | Downloaded | Headers only | <project>\third_party_gles_headers |

### OpenCV Build Configuration

- Generator: MinGW Makefiles
- BUILD_SHARED_LIBS=OFF (static)
- Modules enabled: calib3d, core, features2d, flann, imgcodecs, imgproc, photo
- Modules disabled: highgui, ml, objdetect, stitching, video, videoio, world, apps
- Dependencies excluded: Protobuf, WebP, OpenEXR, TIFF
- OpenCV_DIR for CMake: `C:/bin/opencv/install-mingw/x64/mingw/staticlib`

### GLFW Build Configuration

- Generator: MinGW Makefiles
- BUILD_SHARED_LIBS=OFF (static)
- Library name: libglfw3.a
- Install prefix: `<project>/glfw-3.4/install-mingw`

### GLAD Generation Command

```powershell
python -m pip install glad2==2.0.8
python -m glad --api gles2=3.0 --out-path src/third_party_lib/glad c
```

Generates:
- `include/glad/gles2.h`
- `src/gles2.c`

## Windows System Dependencies

The final executables only depend on standard Windows system DLLs:

- KERNEL32.dll
- msvcrt.dll
- GDI32.dll (GUI executables only)
- USER32.dll (GUI executables only)
- SHELL32.dll (GUI executables only)

No MinGW runtime DLLs (libgcc, libstdc++, libwinpthread) are required at runtime, as they are statically linked via `-static-libgcc -static-libstdc++ -static`.

## Bundled Third-Party Source (in libcimbar repo)

These libraries are included as source in the libcimbar repository and built as part of the project:

| Library | Location | License |
|---------|----------|---------|
| base91 | src/third_party_lib/base91 | — |
| cxxopts | src/third_party_lib/cxxopts | MIT |
| intx | src/third_party_lib/intx | Apache-2.0 |
| libcorrect | src/third_party_lib/libcorrect | — |
| libpopcnt | src/third_party_lib/libpopcnt | BSD-2 |
| wirehair | src/third_party_lib/wirehair | — |
| zstd | src/third_party_lib/zstd | BSD / GPLv2 |

## Verification

To verify dependency versions in your environment:

```powershell
# GCC
gcc --version

# CMake
cmake --version

# Python
python --version

# OpenCV (from installed headers)
Select-String -Path "C:\bin\opencv\install-mingw\include\opencv2\core\version.hpp" -Pattern "CV_VERSION_"

# GLFW (from bundled headers)
Select-String -Path "glfw-3.4\include\GLFW\glfw3.h" -Pattern "GLFW_VERSION_"

# GLAD
python -m pip show glad2
```
