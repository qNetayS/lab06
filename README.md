# Лабораторная работа 06

Цель работы: создание покетов для изменнени(должны сочетаться с тегами)
Для этого нужно добавить ветлвение в конфигурационные файлы для СL.

## 1. Настройка переменных
```
export GITHUB_USERNAME=qNetayS
export GITHUB_EMAIL=qNetayS@github.com
alias edit=nano
alias gsed=sed
```

## 2. Добавляем версионирование в CMakeLists.txt
```
gsed -i '/project(lab06)/a\ set(PRINT_VERSION_MAJOR 0)' CMakeLists.txt
gsed -i '/project(lab06)/a\ set(PRINT_VERSION_MINOR 1)' CMakeLists.txt
gsed -i '/project(lab06)/a\ set(PRINT_VERSION_PATCH 0)' CMakeLists.txt
gsed -i '/project(lab06)/a\ set(PRINT_VERSION_TWEAK 0)' CMakeLists.txt
gsed -i '/project(lab06)/a\ set(PRINT_VERSION "${PRINT_VERSION_MAJOR}.${PRINT_VERSION_MINOR}.${PRINT_VERSION_PATCH}.${PRINT_VERSION_TWEAK}")' CMakeLists.txt
gsed -i '/project(lab06)/a\ set(PRINT_VERSION_STRING "v\\${PRINT_VERSION}")' CMakeLists.txt
```
## 3. Создаем DESCRIPTION и ChangeLog.md
```
touch DESCRIPTION && edit DESCRIPTION
touch ChangeLog.md
export DATE="LANG=en_US date +'%a %b %d %Y'"
cat > ChangeLog.md <<EOF
* ${DATE} ${GITHUB_USERNAME} <${GITHUB_EMAIL}> 0.1.0.0
- Initial RPM release
EOF

```
## 4. Создаем CPackConfig.cmake
```
cat > CPackConfig.cmake <<EOF
include(InstallRequiredSystemLibraries)
EOF

cat >> CPackConfig.cmake <<EOF
set(CPACK_PACKAGE_CONTACT ${GITHUB_EMAIL})
set(CPACK_PACKAGE_VERSION_MAJOR \${PRINT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR \${PRINT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH \${PRINT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_TWEAK \${PRINT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION \${PRINT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_FILE \${CMAKE_CURRENT_SOURCE_DIR}/DESCRIPTION)
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Static C++ library for printing")
EOF

cat >> CPackConfig.cmake <<EOF
set(CPACK_RESOURCE_FILE_LICENSE \${CMAKE_CURRENT_SOURCE_DIR}/LICENSE)
set(CPACK_RESOURCE_FILE_README \${CMAKE_CURRENT_SOURCE_DIR}/README.md)
EOF

cat >> CPackConfig.cmake <<EOF
set(CPACK_RPM_PACKAGE_NAME "print-devel")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_PACKAGE_GROUP "print")
set(CPACK_RPM_CHANGELOG_FILE \${CMAKE_CURRENT_SOURCE_DIR}/ChangeLog.md)
set(CPACK_RPM_PACKAGE_RELEASE 1)
EOF

cat >> CPackConfig.cmake <<EOF
set(CPACK_DEBIAN_PACKAGE_NAME "libprint-dev")
set(CPACK_DEBIAN_PACKAGE_PREDEPENDS "cmake >= 3.0")
set(CPACK_DEBIAN_PACKAGE_RELEASE 1)
EOF

cat >> CPackConfig.cmake <<EOF
include(CPack)
EOF
```
## 4. Подключаем CPack в CMakeLists.txt
```
cat >> CMakeLists.txt <<EOF

include(CPackConfig.cmake)
EOF
```

## 5. Коммитим и пушим
```
git add .
git commit -m "added cpack config"
git tag v0.1.0.0
git push origin master --tags
```
## 6. Настраиваем GitHub Actions (вместо Travis CI)
```
mkdir -p .github/workflows
cat > .github/workflows/linux.yml << 'EOF'
name: Linux CI (gcc & clang)

on:
  push:
    branches: [ main, master ]
    tags:
      - 'v*'
  pull_request:
    branches: [ main, master ]

jobs:
  build:
    runs-on: ubuntu-22.04
    strategy:
      matrix:
        compiler: [gcc, clang]
    env:
      CC: ${{ matrix.compiler }}
      CXX: ${{ matrix.compiler == 'gcc' && 'g++' || 'clang++' }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Install dependencies
        run: sudo apt-get update && sudo apt-get install -y cmake build-essential rpm
      
      - name: Configure
        run: cmake -H. -B_build -DBUILD_TESTS=ON
      
      - name: Build
        run: cmake --build _build
      
      - name: Run tests
        run: ctest --test-dir _build --output-on-failure
      
      - name: Create packages (if tag)
        if: startsWith(github.ref, 'refs/tags/')
        run: |
          cd _build
          cpack -G DEB
          cpack -G RPM
          cpack -G TGZ
          cpack -G ZIP
      
      - name: Upload packages to Release
        if: startsWith(github.ref, 'refs/tags/')
        uses: softprops/action-gh-release@v1
        with:
          files: |
            _build/*.deb
            _build/*.rpm
            _build/*.tar.gz
            _build/*.zip
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
EOF
```
При каждом push запускаются тесты
При создании тэга v* CPack создаёт DEB, RPM, TGZ, ZIP пакеты
Пакеты автоматически загружаются в Release на GitHub
## 7. Коммитим GitHub Actions
```
git add .github/workflows/linux.yml
git commit -m "Add GitHub Actions with CPack packaging on tags"
git push origin master
```
## 8. Создаем пакеты локально для проверки
```
cmake -H. -B_build
cmake --build _build
cd _build
cpack -G "TGZ"
cpack -G "DEB"
cpack -G "RPM"
cd ..
```
# HOMEWORK
## 1. Добавление solver в CPack конфигурацию
```
cat >> CPackConfig.cmake <<'EOF'
set(CPACK_PACKAGE_NAME "solver")
set(CPACK_RPM_PACKAGE_NAME "solver")
set(CPACK_DEBIAN_PACKAGE_NAME "solver")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Solver application with formatter")
set(CPACK_SOURCE_GENERATOR "TGZ;ZIP")
set(CPACK_SOURCE_PACKAGE_FILE_NAME "solver-\${CPACK_PACKAGE_VERSION}-Source")
EOF
```
Обновление CPackConfig.cmake для создания пакетов solver
## 2.Cоздание библиотек formatter и formatter_ex
```
mkdir -p formatter_lib
cat > formatter_lib/formatter.cpp <<'EOF'
#include "formatter.h"
std::string formatter(const std::string& text) { return text; }
EOF

cat > formatter_lib/formatter.h <<'EOF'
#pragma once
#include <string>
std::string formatter(const std::string& text);
EOF

cat > formatter_lib/CMakeLists.txt <<'EOF'
cmake_minimum_required(VERSION 3.5)
project(formatter)
set(CMAKE_CXX_STANDARD 11)
add_library(formatter STATIC formatter.cpp)
EOF
```
Результат: Создание библиотек formatter и formatter_ex
## 3.Создание библиотеки solver_lib и приложения solver
```
mkdir -p solver_lib
cat > solver_lib/solver.cpp <<'EOF'
#include "solver.h"
double solver(double a, double b) { return a + b; }
EOF

cat > solver_lib/solver.h <<'EOF'
#pragma once
double solver(double a, double b);
EOF

cat > solver_lib/CMakeLists.txt <<'EOF'
cmake_minimum_required(VERSION 3.5)
project(solver_lib)
set(CMAKE_CXX_STANDARD 11)
add_library(solver_lib STATIC solver.cpp)
EOF

mkdir -p solver_application
cat > solver_application/equation.cpp <<'EOF'
#include <iostream>
#include "formatter_ex.h"
#include "solver.h"

int main() {
    std::cout << formatter_ex("Solving equation: x + 5 = 10") << std::endl;
    double result = solver(5, 5);
    std::cout << "Result: x = " << result << std::endl;
    return 0;
}
EOF

cat > solver_application/CMakeLists.txt <<'EOF'
cmake_minimum_required(VERSION 3.5)
project(solver)
set(CMAKE_CXX_STANDARD 11)
add_executable(solver equation.cpp)
include_directories(${CMAKE_SOURCE_DIR}/formatter_lib)
include_directories(${CMAKE_SOURCE_DIR}/formatter_ex_lib)
include_directories(${CMAKE_SOURCE_DIR}/solver_lib)
target_link_libraries(solver formatter_ex solver_lib formatter)
EOF
```
## 4.Обновление корневого CMakeLists.txt
```
cat >> CMakeLists.txt <<'EOF'
add_subdirectory(formatter_lib)
add_subdirectory(formatter_ex_lib)
add_subdirectory(solver_lib)
add_subdirectory(solver_application)
EOF
```
Результат: Подключение всех поддиректорий к сборке


## 5.Локальная сборка всех пакетов
```
rm -rf _build
cmake -H. -B_build -DCPACK_GENERATOR="TGZ;DEB;RPM"
cmake --build _build
cmake --build _build --target package

ls -la _build/*.deb _build/*.rpm _build/*.tar.gz 2>/dev/null
```
Результат: Созданы пакеты DEB, RPM, TGZ для solver
## 6.Итоговая структура
```
lab06/
├── artifacts/
│   └── banking-0.1.0.0-Linux.tar.gz
├── .github/workflows/
│   └── linux.yml
├── formatter_lib/
├── formatter_ex_lib/
├── solver_lib/
├── solver_application/
├── banking/
├── tests/
├── CMakeLists.txt
├── CPackConfig.cmake
├── DESCRIPTION
├── ChangeLog.md
├── LICENSE
└── README.md
```

## 7. Создаем artifacts (скриншот)
```
mkdir artifacts
sleep 20s && gnome-screenshot --file artifacts/screenshot.png
```
