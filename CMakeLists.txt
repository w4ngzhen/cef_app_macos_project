CMAKE_MINIMUM_REQUIRED(VERSION 3.21)

PROJECT(cef_app_macos_project LANGUAGES CXX)

# 基础配置
SET(CMAKE_BUILD_TYPE DEBUG)
SET(CMAKE_CXX_STANDARD 17)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_INCLUDE_CURRENT_DIR ON)

# ===== CEF =====
if (NOT DEFINED ENV{CEF_ROOT})
    message(FATAL_ERROR "环境变量CEF_ROOT未定义！")
endif ()
# 执行下面之前，请确保环境变量CEF_ROOT已经配置为了对应cef_binary_xxx目录
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
find_package(CEF REQUIRED)

# ===== 子模块引入 =====
# 1. CEF前置准备完成后，此处便可以使用变量 CEF_LIBCEF_DLL_WRAPPER_PATH ，该值会返回libcef_dll_wrapper的目录地址
add_subdirectory(${CEF_LIBCEF_DLL_WRAPPER_PATH} libcef_dll_wrapper)
# 2. 将cef_app作为子模块引入
add_subdirectory(./cef_app)
