
cmake_minimum_required(VERSION 3.0)
project(rosfmt)

find_package(catkin REQUIRED COMPONENTS
	roscpp
	rosconsole
)

# Get fmt from an official release
include(ExternalProject)
ExternalProject_Add(
	fmt
	URL https://github.com/fmtlib/fmt/releases/download/9.1.0/fmt-9.1.0.zip
	URL_HASH SHA256=cceb4cb9366e18a5742128cb3524ce5f50e88b476f1e54737a47ffdf4df4c996
	CONFIGURE_COMMAND ""

	# We touch all downloaded files, otherwise they get some timestamp in the
	# past and may confuse the dependency tracking below.
	BUILD_COMMAND find <SOURCE_DIR> -type f -exec touch {} +
	INSTALL_COMMAND ""
)

# Now extract the needed files. We need to copy headers to the devel space
# b/c of catkin policies (they count as "generated headers").
ExternalProject_Get_property(fmt SOURCE_DIR)
set(fmt_SOURCE_DIR "${SOURCE_DIR}")
set(fmt_HEADER_DEST "${CATKIN_DEVEL_PREFIX}/${CATKIN_GLOBAL_INCLUDE_DESTINATION}")

file(MAKE_DIRECTORY ${fmt_HEADER_DEST})
foreach(FILE args.h chrono.h color.h compile.h core.h format-inl.h format.h os.h ostream.h printf.h ranges.h std.h xchar.h)
	set(in "${fmt_SOURCE_DIR}/include/fmt/${FILE}")
	set(out "${fmt_HEADER_DEST}/fmt/${FILE}")
	list(APPEND fmt_HEADERS_OUT ${out})

	set_source_files_properties(${in} PROPERTIES GENERATED TRUE)
	set_source_files_properties(${out} PROPERTIES GENERATED TRUE)

	add_custom_command(
		OUTPUT ${out}
		DEPENDS ${in} fmt
		COMMAND cmake -E copy_if_different "${in}" "${out}"
	)
endforeach()

add_custom_target(fmt_headers
	DEPENDS ${fmt_HEADERS_OUT}
)

catkin_package(
	INCLUDE_DIRS include ${fmt_HEADER_DEST}
	CATKIN_DEPENDS roscpp rosconsole
	LIBRARIES rosfmt9
)

include_directories(
	${fmt_HEADER_DEST}
	include
	${catkin_INCLUDE_DIRS}
)

set_source_files_properties(
	${fmt_SOURCE_DIR}/src/format.cc
	${fmt_SOURCE_DIR}/src/os.cc
	PROPERTIES GENERATED TRUE
)

add_library(rosfmt9
	${fmt_HEADERS_OUT}
	${fmt_SOURCE_DIR}/src/format.cc
	${fmt_SOURCE_DIR}/src/os.cc
	src/rosfmt.cpp
)
target_link_libraries(rosfmt9
	${catkin_LIBRARIES}
)
add_dependencies(rosfmt9 fmt fmt_headers)

add_executable(simple_test
	test/test.cpp
)
target_link_libraries(simple_test rosfmt9)
add_test(
	NAME SimpleTest
	COMMAND simple_test
)
add_custom_target(run_simple_test
	COMMAND simple_test
)
add_dependencies(run_tests run_simple_test)


install(
	DIRECTORY ${fmt_HEADER_DEST}/fmt/
	DESTINATION ${CATKIN_GLOBAL_INCLUDE_DESTINATION}/fmt
)
install(
	DIRECTORY include/rosfmt/
	DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
)
install(
	TARGETS rosfmt9
	DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
)
