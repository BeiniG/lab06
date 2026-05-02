# Лабораторная работа №6

Данная лабораторная работа посвещена изучению средств пакетирования на примере **CPack**

## Homework
В главный `CMakeLists.txt` добавлена команда `install`, копирующая `solver` в папку `bin`.
```cmake
cmake_minimum_required(VERSION 3.10)
project(examples)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(PRINT_VERSION_MAJOR 1)
set(PRINT_VERSION_MINOR 0)
set(PRINT_VERSION_PATCH 0)
set(PRINT_VERSION_TWEAK 0)
set(PRINT_VERSION ${PRINT_VERSION_MAJOR}.${PRINT_VERSION_MINOR}.${PRINT_VERSION_PATCH}.${PRINT_VERSION_TWEAK})
set(PRINT_VERSION_STRING "v${PRINT_VERSION}")

add_subdirectory(formatter_lib)
add_subdirectory(formatter_ex_lib)
add_subdirectory(hello_world)
add_subdirectory(solver)

install(TARGETS solver
        DESTINATION bin
        COMPONENT applications)

include(CPackConfig.cmake)
```
Файл CPackConfig.cmake:
```cpack
include(InstallRequiredSystemLibraries)

set(CPACK_PACKAGE_CONTACT ivanvas1247@gmail.com)
set(CPACK_PACKAGE_VERSION_MINOR ${PRINT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_TWEAK ${PRINT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION_PATCH ${PRINT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_MAJOR ${PRINT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION ${PRINT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "solver lib")

set(CPACK_RESOURCE_FILE_README ${CMAKE_CURRENT_SOURCE_DIR}/README.md)
set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE)

set(CPACK_RPM_PACKAGE_NAME "solver")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_PACKAGE_GROUP "solver")
set(CPACK_RPM_PACKAGE_RELEASE 1)
set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")

set(CPACK_DEBIAN_PACKAGE_NAME "libsolver-dev")
set(CPACK_DEBIAN_PACKAGE_PREDEPENDS "cmake >= 3.0")
set(CPACK_DEBIAN_PACKAGE_RELEASE 1)

if(WIN32)
    set(CPACK_GENERATOR "WIX")
    set(CPACK_WIX_ARCHITECTURE "x64")
    set(CPACK_WIX_PRODUCT_GUID "05b23be7-c85f-40fb-942c-7a678101c21a")
    set(CPACK_WIX_LICENSE_RTF "${CMAKE_SOURCE_DIR}/LICENSE")
    set(CPACK_WIX_PROGRAM_MENU_FOLDER "out")
    set(CPACK_WIX_VERSION "4")
    set(CPACK_COMPONENTS_ALL applications)
elseif(APPLE)
    set(CPACK_GENERATOR "DragNDrop")
    set(CPACK_DMG_VOLUME_NAME "solver")
    set(CPACK_DMG_FORMAT "UDBZ")
else()
    set(CPACK_GENERATOR "DEB;RPM;TGZ;ZIP")
endif()

include(CPack)

```

Создаем `CI.yml`. Файл собирает пакеты с расширениями `.deb`, `.rpm`, `.msi`, `.dmg`, чтобы проверить работу программ на разных операционных системах. При первом запуске сборки пакетов не осуществляются, для этого нужно добавить тэги.

```yml
name: lab03 library build

on:
  push:
    tags: [ "v*" ]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Get repo files
        uses: actions/checkout@v6
      - name: Build dir configuratiob
        run: cmake -H. -B build
      - name: Build
        run: cmake --build build
      - name: Upload artifacts
        uses: actions/upload-artifact@v6
        with:
          name: Applications
          path: |
            ./build/hello_world/hello_world
            ./build/solver/solver
          retention-days: 1

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v6
        with:
          name: Applications
          path: ./apps
      - name: Fix permissions
        run: |
          chmod +x ./apps/hello_world/hello_world
          chmod +x ./apps/solver/solver
      - name: Hello_world test
        run: ./apps/hello_world/hello_world
      - name: Solver test
        run: echo "1 5 -6" | ./apps/solver/solver

  create-msi:
    needs: test
    runs-on: windows-latest
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - name: Get repo files
        uses: actions/checkout@v6
      - name: .NET setup
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.x'
      - name: WiX Toolset installation
        run: |
          dotnet tool install --global wix --version 4.0.0
          wix extension add --global WixToolset.UI.wixext/4.0.4
      - name: Build
        run: |
          mkdir build
          cmake -B build
          cmake --build build --config Release --verbose
        shell: pwsh
      - name: Create MSI package
        run: |
          cd build
          cpack -G WIX -C Release --verbose
      - name: Upload .msi
        uses: softprops/action-gh-release@v2
        with:
          files: build/*.msi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  create-dmg:
    needs: test
    runs-on: macos-latest
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - name: Get repo files
        uses: actions/checkout@v6
      - name: Build
        run: |
          cmake -H. -B build -DCMAKE_BUILD_TYPE=Release
          cmake --build build
      - name: Create .dmg
        run: |
          cd build
          cpack -G DragNDrop -C Release
      - name: Upload .dmg
        uses: softprops/action-gh-release@v2
        with:
          files: build/*.dmg
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  create-rpm-deb:
    needs: test
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')
    steps:
      - name: Get repo files
        uses: actions/checkout@v6
      - name: Rpm install
        run: |
          sudo apt-get update
          sudo apt-get install -y rpm
      - name: Create packages
        run: |
          cmake -H. -B build -DCMAKE_BUILD_TYPE=Release
          cmake --build build
          cd build
          cpack -G DEB -C Release
          cpack -G RPM -C Release
      - name: Upload binary packages
        uses: softprops/action-gh-release@v2
        with:
          files: |
            build/*.deb
            build/*.rpm
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
 Сборка и тестирование прошли без ошибок.
