cmake_minimum_required(VERSION 3.5 FATAL_ERROR)

# 工具链设置
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR riscv64)

# 工具链路径配置 
set(CROSS_COMPILE riscv64-unknown-elf-)
set(CMAKE_C_COMPILER ${CROSS_COMPILE}gcc)
set(CMAKE_ASM_COMPILER ${CROSS_COMPILE}gcc)
set(CMAKE_OBJCOPY ${CROSS_COMPILE}objcopy)
set(CMAKE_OBJDUMP ${CROSS_COMPILE}objdump)
# 架构选项
set(RISCV_ARCH rv64g)
set(RISCV_ABI lp64)
list(APPEND ARCH_FLAGS "-march=${RISCV_ARCH}" "-mabi=${RISCV_ABI}")
# 编译选项
list(APPEND CFLAGS ${ARCH_FLAGS} "-nostdlib" "-fno-builtin" "-g" "-Wall")
list(APPEND ASMFLAGS ${ARCH_FLAGS} "-x" "assembler-with-cpp" "-nostdlib" "-fno-builtin" "-g" "-Wall")
# 禁用所有编译器检查
set(CMAKE_C_COMPILER_WORKS 1)
set(CMAKE_ASM_COMPILER_WORKS 1)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY) # 仅编译静态库，不执行链接
project(JaeOS LANGUAGES C ASM)


# QEMU配置
find_program(QEMU_EXECUTABLE
	NAMES
	qemu-system-riscv64
	HINTS
	"/usr/bin"
	DOC
	"Finding QEMU system emulator......"
	REQUIRED
)
execute_process(
	COMMAND
	${QEMU_EXECUTABLE} --version
	OUTPUT_VARIABLE QEMU_VERSION_OUTPUT
)
message(STATUS "Find QEMU: ${QEMU_EXECUTABLE} Version: ${QEMU_VERSION_OUTPUT}")
list(APPEND QFLAGS "-nographic" "-machine" "virt" "-smp" "4") # 参数配置

# GDB-multiarch配置
find_program(GDB_EXECUTABLE
	NAMES
	gdb-multiarch
	HINTS
	"/usr/bin"
	DOC
	"Finding GDB-Multiarch server......"
	REQUIRED
)
execute_process(
	COMMAND
	${GDB_EXECUTABLE} --version
	OUTPUT_VARIABLE GDB_VERSION_OUTPUT
)
message(STATUS "Find GDB-Multiarch: ${GDB_EXECUTABLE} Version: ${GDB_VERSION_OUTPUT}")

# 输出路径设置
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# 明确汇编文件类型
enable_language(ASM)

# 链接配置
option(USE_LINKER_SCRIPT "Use generated linker script" OFF)
message(STATUS "Use generated linker script?--> ${USE_LINKER_SCRIPT}")

# 汇编文件编译成库
add_library(start_up OBJECT "")
target_sources(start_up
	PRIVATE
	${CMAKE_SOURCE_DIR}/start.S)
target_compile_options(start_up
	PRIVATE
	${ASMFLAGS}
)
# 生成可执行文件
add_executable(kernel.elf "")
target_sources(kernel.elf 
	PRIVATE
	$<TARGET_OBJECTS:start_up>
	${CMAKE_SOURCE_DIR}/init.c
)
target_compile_options(kernel.elf
	PRIVATE
	${CFLAGS}
	"-O0"
)
target_include_directories(kernel.elf
	PRIVATE
	${CMAKE_SOURCE_DIR}
)
# 简单链接，不使用链接脚本
target_link_options(kernel.elf
	PRIVATE
	"-T"
	"${CMAKE_SOURCE_DIR}/linker/qemu_kernel.ld"
	"-nostdlib"
)
# 编译完成后从elf文件转换成bin文件，去掉调试信息
add_custom_command(
	TARGET
	kernel.elf
	POST_BUILD
	COMMAND
	${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:kernel.elf> ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/kernel.bin
	VERBATIM
	)

# 添加清理规则
add_custom_target(clean-all
	COMMAND
	${CMAKE_COMMAND} -E rm -rf *
	WORKING_DIRECTORY
	${CMAKE_BINARY_DIR}
	COMMENT
	"Clear All Builded Files......"
	VERBATIM
)
# 添加QEMU运行规则
add_custom_target(qemu
	COMMAND
	${QEMU_EXECUTABLE} ${QFLAGS} -kernel $<TARGET_FILE:kernel.elf>
	DEPENDS
	$<TARGET_FILE:kernel.elf>
	WORKING_DIRECTORY
	${CMAKE_BINARY_DIR}
	COMMENT
	"Running ${CMAKE_PROJECT_NAME} in QEMU......"
	VERBATIM
)
# 添加gdb调试规则
add_custom_target(debug
	COMMAND
	${QEMU_EXECUTABLE} ${QFLAGS} 
	-kernel $<TARGET_FILE:kernel.elf> 
	-S -gdb tcp::1234 
	-serial mon:stdio
	WORKING_DIRECTORY
	${CMAKE_BINARY_DIR}
	DEPENDS
	$<TARGET_FILE:kernel.elf>
	COMMENT
	"Start QEMU in Debug mode(use 'make gdb' to connect)"
	VERBATIM
)
# 先启用debug，在启用gdb
add_custom_target(gdb
	COMMAND
	${GDB_EXECUTABLE} 
	-ex "target remote :1234"
	-ex "break _start"
	$<TARGET_FILE:kernel.elf>
	COMMENT
	"Connecting to QEMU GDB Server......"
	VERBATIM
)
# 内核代码反汇编
add_custom_target(disassembly
	COMMAND
	${CMAKE_OBJDUMP} -d $<TARGET_FILE:kernel.elf> >> ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/diskernel.S
	WORKING_DIRECTORY
	${CMAKE_BINARY_DIR}
	DEPENDS
	$<TARGET_FILE:kernel.elf>
	COMMENT
	"Disassembly kernel.elf to diskernel.S......"
	VERBATIM
)
