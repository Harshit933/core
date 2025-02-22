# Check if this loader is enabled
if(NOT OPTION_BUILD_LOADERS OR NOT OPTION_BUILD_LOADERS_RS)
	return()
endif()

#
# Plugin name and options
#

file(READ ${CMAKE_CURRENT_SOURCE_DIR}/rust-toolchain TARGET_TOOLCHAIN)
string(STRIP "${TARGET_TOOLCHAIN}" TARGET_TOOLCHAIN)

# For checking availability of external tools like rls or miri (in the future), use: https://rust-lang-nursery.github.io/rust-toolstate/
set(Rust_TOOLCHAIN_COMPONENT_LIST
	cargo
	clippy
	llvm-tools-preview
	rls
	rust-analysis
	rust-analyzer-preview
	rust-std
	rustc
	rustc-dev
	rustfmt
	rust-src
)

find_package(Rust COMPONENTS ${TARGET_TOOLCHAIN})

if(NOT Rust_FOUND)
	message(SEND_ERROR "Rust not found")
	return()
endif()

# Target name
set(target rs_loader_impl)

# Exit here if required dependencies are not met
message(STATUS "Plugin ${target}")

set(TARGET_BUILD_NAME "${CMAKE_SHARED_LIBRARY_PREFIX}${target}${CMAKE_SHARED_LIBRARY_SUFFIX}")

if(CMAKE_BUILD_TYPE STREQUAL "Release")
	set(TARGET_BUILD_TYPE "--release")
	set(TARGET_BUILD_PATH "${CMAKE_CURRENT_SOURCE_DIR}/target/release/${TARGET_BUILD_NAME}")
	set(TARGET_OUTPUT_NAME "${CMAKE_SHARED_LIBRARY_PREFIX}${target}${CMAKE_SHARED_LIBRARY_SUFFIX}")
else()
	set(TARGET_BUILD_TYPE)
	set(TARGET_BUILD_PATH "${CMAKE_CURRENT_SOURCE_DIR}/target/debug/${TARGET_BUILD_NAME}")
	set(TARGET_OUTPUT_NAME "${CMAKE_SHARED_LIBRARY_PREFIX}${target}d${CMAKE_SHARED_LIBRARY_SUFFIX}")
endif()

set(TARGET_OUTPUT ${PROJECT_OUTPUT_DIR}/${TARGET_OUTPUT_NAME})

add_custom_target(${target}_runtime
	COMMAND ${CMAKE_COMMAND} -E copy_if_different ${Rust_RUSTC_LIBRARIES} ${PROJECT_OUTPUT_DIR}
)

# On Linux, execute patchelf in order to patch rpath in the dependencies
if(PROJECT_OS_FAMILY MATCHES "unix" OR PROJECT_OS_FAMILY MATCHES "macos")
	# Find Patchelf
	find_package(Patchelf)

	if(NOT Patchelf_FOUND)
		include(InstallPatchelf)

		if(NOT Patchelf_FOUND)
			message(SEND_ERROR "Patchelf not found")
			return()
		endif()

		set(TARGET_INSTALL_DEPENDENCY Patchelf)
	endif()

	set(RUSTC_LIBS)
	add_custom_target(${target}_patchelf DEPENDS ${target}_runtime ${TARGET_INSTALL_DEPENDENCY})
	foreach(rustc_lib ${Rust_RUSTC_LIBRARIES})
		get_filename_component(rustc_lib_name ${rustc_lib} NAME)
		list(APPEND RUSTC_LIBS ${PROJECT_OUTPUT_DIR}/${rustc_lib_name})
		add_custom_command(TARGET ${target}_patchelf POST_BUILD
			COMMAND ${Patchelf_EXECUTABLE} --set-rpath [=["\$$ORIGIN"]=] ${PROJECT_OUTPUT_DIR}/${rustc_lib_name}
		)
	endforeach()

	set(TARGET_DEPENDENCY ${target}_patchelf)
else()
	# TODO: Implement patchelf equivalent (if needed) for other platforms
	set(RUSTC_LIBS ${Rust_RUSTC_LIBRARIES})
endif()

if(OPTION_BUILD_MUSL)
	set(RUST_CTR_STATIC "-Ctarget-feature=-crt-static -Clink-self-contained=off -L/usr/lib/x86_64-linux-musl") #-Clink-args=--dynamic-linker /lib/ld-musl-x86_64.so.1
	set(RUSTFLAGS ${CMAKE_COMMAND} -E env RUSTFLAGS=${RUST_CTR_STATIC})
else()
	set(RUSTFLAGS)
endif()

add_custom_target(${target} ALL
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	COMMAND ${RUSTFLAGS} ${Rust_CARGO_EXECUTABLE} build ${TARGET_BUILD_TYPE}
	COMMAND ${CMAKE_COMMAND} -E copy_if_different ${TARGET_BUILD_PATH} ${TARGET_OUTPUT}
	DEPENDS ${target}_runtime ${TARGET_DEPENDENCY}
)

set_property(TARGET ${target}
	PROPERTY OUTPUT_NAME ${TARGET_OUTPUT}
)

# Prepare the environment variable file for VSCode tests
include(Portability)

project_library_path(TEST_LIB_PATH
	${PROJECT_OUTPUT_DIR}
)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/.vscode/launch.json.in ${CMAKE_CURRENT_SOURCE_DIR}/.vscode/launch.json @ONLY)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/.vscode/settings.json.in ${CMAKE_CURRENT_SOURCE_DIR}/.vscode/settings.json @ONLY)

#
# Deployment
#

# Library
install(FILES
	${TARGET_OUTPUT}
	DESTINATION ${INSTALL_LIB}
	COMPONENT runtime
)

# Rust Runtime (pack the runtime meanwhile Rust runtime is not distributed as a library)
install(FILES
	${RUSTC_LIBS}
	DESTINATION ${INSTALL_LIB}
	COMPONENT runtime
)
