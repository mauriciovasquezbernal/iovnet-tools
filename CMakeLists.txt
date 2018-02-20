cmake_minimum_required (VERSION 3.2)

project(iovnet-tools)

add_subdirectory(netdissect)

add_library(iovnet-tools STATIC $<TARGET_OBJECTS:netdissect>)
#set_target_properties(iovnet-tools PROPERTIES LINKER_LANGUAGE CXX)
#set_target_properties(iovnet-tools PROPERTIES LINK_FLAGS "-Wl,--whole-archive")

#set_property(TARGET iovnet-tools PROPERTY POSITION_INDEPENDENT_CODE ON)

#target_link_libraries(iovnet-tools netdissect)

set(CMAKE_INSTALL_LIBDIR /usr/lib)

install(
    TARGETS
    iovnet-tools
    DESTINATION
    "${CMAKE_INSTALL_LIBDIR}"
)

