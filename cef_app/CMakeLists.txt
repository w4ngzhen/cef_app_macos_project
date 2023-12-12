# ===== 主进程target配置 =====
# 主进程target名称
set(CEF_APP_TARGET cef_app)
# 最终 App Bundle生成的路径
set(CEF_APP_BUNDLE "${CMAKE_CURRENT_BINARY_DIR}/${CEF_APP_TARGET}.app")
# 添加项目所有的源文件：
add_executable(
    ${CEF_APP_TARGET}
    MACOSX_BUNDLE # macOS 使用 "MACOSX_BUNDLE" 标识，最后编译产物是一个mac下的App Bundle
    process_main.mm
)
# 使用CEF提供的预定义好的工具宏，该宏会帮助配置target一些编译上的配置
# 如果出现不符合预期的编译结果、运行错误，可以检查该宏的内部实现
SET_EXECUTABLE_TARGET_PROPERTIES(${CEF_APP_TARGET})
# 添加对 libcef_dll_wrapper 库的依赖
# 基于该配置，可以保证每次编译当前 cef_app target时候，确保 libcef_dll_wrapper 静态库编译完成
add_dependencies(${CEF_APP_TARGET} libcef_dll_wrapper)
# 链接库配置
target_link_libraries(
    ${CEF_APP_TARGET}
    PRIVATE
    # libcef_dll_wrapper库链接
    libcef_dll_wrapper
    # 该变量来自cef_variables.cmake中定义的配置
    # 主要是针对不同的平台，链接对应平台的一些标准库（Windows、Linux）或者framework（macOS）
    ${CEF_STANDARD_LIBS}
)
# 主进程编译后，会在输出目录下生成一个名为 cef_app.app 的macOS App Bundle。
# 该app内部 Contents/MacOS/cef_app 仅仅是包含了 add_executable 中的源码二进制，以及libcef_dll_wrapper静态库
# 在macOS下，我们还需要将"cef_binary_xxx/Debug或Release目录/Chromium Embedded Framework.framework"复制到
# cef_app.app/Contents/Frameworks目录下
# 为了避免手动复制的麻烦，我们使用如下的指令完成复制工作
add_custom_command(
    # 对 CEF_APP_TARGET 进行操作
    TARGET ${CEF_APP_TARGET}
    # 在构建完成后（POST_BUILD）
    POST_BUILD
    # COMMAND ${CMAKE_COMMAND}：就是命令行执行 "cmake"
    # -E：指可以执行一些cmake内置的工具命令
    # copy_directory：进行目录复制操作
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    # 复制源目录、文件，
    # CEF_BINARY_DIR变量来源于cef_variables.cmake
    # 等价于"cef_binary_xxx目录/Debug或Release目录/"
    "${CEF_BINARY_DIR}/Chromium Embedded Framework.framework"
    # 将上述 framework 复制到 当前生成的 cef_app.app/Contents/Frameworks/对应framework名称
    "${CEF_APP_BUNDLE}/Contents/Frameworks/Chromium Embedded Framework.framework"
    # 不进行文本的解析，使用源文字，考虑会有表达式情况
    VERBATIM
)
# 简单配置Info.plist的一些值
set_target_properties(
    ${CEF_APP_TARGET}
    PROPERTIES
    MACOSX_BUNDLE_BUNDLE_NAME ${CEF_APP_TARGET}
    MACOSX_BUNDLE_GUI_IDENTIFIER ${CEF_APP_TARGET}
)

# ===== 子进程 helper target配置 =====
# 定义helper子进程target名
set(CEF_APP_HELPER_TARGET "cef_app_helper")
# 定义helper子进程构建后的app的名称
set(CEF_APP_HELPER_OUTPUT_NAME "cef_app Helper")
# 注意，上述的名称都不是最终名称，它们更准确的意义是作为下面循环定义target的基础名称
# 后续循环的时候，会基于上述名称进行拼接

# 创建多个不同类型helper的target
# CEF_HELPER_APP_SUFFIXES来自cef_variables.cmake，是一个“字符串数组”，值有：
# "::"、" (Alerts):_alerts:.alerts"、" (GPU):_gpu:.gpu"、
# " (Plugin):_plugin:.plugin"、" (Renderer):_renderer:.renderer"
# 这里通过foreach，实现对字符串数组的遍历，每一次循环会得到一个字符串，存放在“_suffix_list”
foreach (_suffix_list ${CEF_HELPER_APP_SUFFIXES})
  # 将字符串转为";"分割，这样可以使用CMake支持的list(GET)指令来读取每一节字符串
  # 以 " (Renderer):_renderer:.renderer" 为例
  string(REPLACE ":" ";" _suffix_list ${_suffix_list}) # " (Renderer);_renderer;.renderer"
  list(GET _suffix_list 0 _name_suffix) # " (Renderer)"
  list(GET _suffix_list 1 _target_suffix) # "_renderer"
  list(GET _suffix_list 2 _plist_suffix) # ".renderer"
  # 当然，需要注意 CEF_HELPER_APP_SUFFIXES 中有一个"::"的字符串，
  # 会使得 _name_suffix = ""、_target_suffix = ""、_plist_suffix = ""

  # 定义一个Helper target以及BUNDLE名称
  # 以 " (Renderer):_renderer:.renderer" 为例
  # _helper_target = "cef_app_helper" + "_renderer" -> "cef_app_helper_renderer"
  # _helper_output_name = "cef_app Helper" + " (Renderer)" -> "cef_app Helper (Renderer)"
  set(_helper_target "${CEF_APP_HELPER_TARGET}${_target_suffix}")
  set(_helper_output_name "${CEF_APP_HELPER_OUTPUT_NAME}${_name_suffix}")

  # 读取templates/helper-Info.plist模板文件内容到_plist_contents
  # 然后使用上面得到的 _helper_output_name、_plist_suffix等变量进行文本内容的替换操作
  # 以便得到当前正在处理的helper对应的一份Info.plist
  file(READ "${CMAKE_CURRENT_SOURCE_DIR}/templates/helper-Info.plist" _plist_contents)
  string(REPLACE "\${HELPER_EXECUTABLE_NAME}" "${_helper_output_name}" _plist_contents ${_plist_contents})
  string(REPLACE "\${PRODUCT_NAME}" "${_helper_output_name}" _plist_contents ${_plist_contents})
  string(REPLACE "\${BUNDLE_ID_SUFFIX}" "${_plist_suffix}" _plist_contents ${_plist_contents})
  # helper的Info.plist文件路径，例如："${CMAKE_CURRENT_BINARY_DIR}/helper-Info[_renderer].plist"
  set(_helper_info_plist_file "${CMAKE_CURRENT_BINARY_DIR}/helper-Info${_target_suffix}.plist")
  # 通过CMake提供file(WRITE)命令，将前面定义的内容写入到对应.plist文件中
  file(WRITE ${_helper_info_plist_file} ${_plist_contents})

  # 创建当前helper的executable target，当然，也是一个App Bundle
  add_executable(${_helper_target}
      MACOSX_BUNDLE
      process_helper.mm
  )
  # 与主进程应用一样，
  # 通过cef提供的SET_EXECUTABLE_TARGET_PROPERTIES宏，来设置编译参数、头文件路径等
  SET_EXECUTABLE_TARGET_PROPERTIES(${_helper_target})
  # 编译当前Helper target前，先编译 libcef_dll_wrapper target
  add_dependencies(${_helper_target} libcef_dll_wrapper)
  # 当前Helper target的库链接
  target_link_libraries(${_helper_target} libcef_dll_wrapper ${CEF_STANDARD_LIBS})
  # 定义当前Helper target的一些属性
  set_target_properties(${_helper_target} PROPERTIES
      # 这里使用“MACOSX_BUNDLE_INFO_PLIST”，
      # 来定义构建过程Bundle使用的Info.plist来源于前面我们通过模板文件生成的.plist
      MACOSX_BUNDLE_INFO_PLIST ${_helper_info_plist_file}
      # 定义最终生成的App Bundle的名称
      OUTPUT_NAME ${_helper_output_name}
  )

  # 构建主进程应用前，会先构建当前Helper target
  add_dependencies(${CEF_APP_TARGET} "${_helper_target}")

  # 将构建的Helper App Bundle拷贝到主进程cef_app的Bundle中
  add_custom_command(
      TARGET ${CEF_APP_TARGET}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy_directory
      "${CMAKE_CURRENT_BINARY_DIR}/${_helper_output_name}.app"
      "${CEF_APP_BUNDLE}/Contents/Frameworks/${_helper_output_name}.app"
      VERBATIM
  )
endforeach ()
