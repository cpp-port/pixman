cmake_minimum_required(VERSION 3.16)
project(pixman C)

if(__REACTOS__)
    set(PIXMAN_VERSION_MAJOR 0)
    set(PIXMAN_VERSION_MINOR 40)
    set(PIXMAN_VERSION_MICRO 0)
    file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/generated/pixman")
    configure_file(pixman/pixman-version.h.in "${CMAKE_CURRENT_BINARY_DIR}/generated/pixman/pixman-version.h" @ONLY)
    file(READ "${WORKSPACE_FOLDER}/operating_system/operating_system-windows/configuration/pixman/config.h" pixman_config)
    string(REGEX REPLACE "#define USE_[A-Z0-9_]+ 1" "" pixman_config "${pixman_config}")
    file(GENERATE OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/config.h" CONTENT "${pixman_config}")

endif()


message("CMake <<== Trash : ${PROJECT_NAME}")


set(PIXMAN_TARGET_ARMV7 FALSE)

if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm|ARM|armv7|ARMV7)" OR ANDROID_ABI STREQUAL "armeabi-v7a")
  set(PIXMAN_TARGET_ARMV7 TRUE)
endif()


list(APPEND library_source
       pixman/pixman.c
       pixman/pixman-access.c
       pixman/pixman-access-accessors.c
       pixman/pixman-arm.c
       pixman/pixman-bits-image.c
       pixman/pixman-combine32.c
       pixman/pixman-combine-float.c
       pixman/pixman-conical-gradient.c
       pixman/pixman-edge.c
       pixman/pixman-edge-accessors.c
       pixman/pixman-fast-path.c
       pixman/pixman-filter.c
       pixman/pixman-general.c
       pixman/pixman-glyph.c
       pixman/pixman-gradient-walker.c
       pixman/pixman-image.c
       pixman/pixman-implementation.c
       pixman/pixman-linear-gradient.c
       pixman/pixman-matrix.c
       pixman/pixman-mips.c
       pixman/pixman-mmx.c
       pixman/pixman-noop.c
       pixman/pixman-ppc.c
       pixman/pixman-radial-gradient.c
       #pixman/pixman-region.c
       pixman/pixman-region16.c
       pixman/pixman-region32.c
       pixman/pixman-solid-fill.c
       pixman/pixman-timer.c
       pixman/pixman-trap.c
       pixman/pixman-utils.c
       pixman/pixman-version.h.in
)


if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64")
  # Code specific to 64-bit Intel
  list(APPEND library_source
     pixman/pixman-sse2.c
     pixman/pixman-ssse3.c
  )
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "i.86")
  # Code specific to 32-bit Intel
  list(APPEND library_source
     pixman/pixman-sse2.c
     pixman/pixman-ssse3.c
  )
endif()


if(${ALTIVEC})

  list(APPEND library_source
          pixman/pixman-vmx.c
  )

endif()
if(IS_ARM_NEON)

  list(APPEND library_source
          pixman/pixman-arm-neon.c
          pixman/pixman-arm-neon-asm.S
          pixman/pixman-arm-neon-asm-bilinear.S
  )

endif()
if(IS_ARM_SIMD)

  list(APPEND library_source
          pixman/pixman-arm-simd.c
          pixman/pixman-arm-simd-asm.S
          pixman/pixman-arm-simd-asm-scaled.S

  )

endif()

#if(${IS_X86_ARCH})

  list(APPEND library_source
          pixman/pixman-x86.c

  )

#endif()

if(${IS_MIPS})
  list(APPEND library_source

pixman/pixman-mips-dspr2.c
pixman/pixman-mips-dspr2-asm.S
pixman/pixman-mips-memcpy-asm.S
  )

endif()

add_library(${PROJECT_NAME}
  SHARED ${library_source}
)
add_library(static_${PROJECT_NAME}
        STATIC ${library_source}
)

target_compile_definitions(${PROJECT_NAME} PRIVATE HAVE_CONFIG_H)
target_compile_definitions(static_${PROJECT_NAME} PRIVATE HAVE_CONFIG_H)

if(${this_system_libraries})
  target_link_libraries(${PROJECT_NAME} PRIVATE ${this_system_libraries})
endif()


if(${WIN32})

  set_target_properties(${PROJECT_NAME} PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON )


   if(MSVC)


      target_compile_options(${PROJECT_NAME} PRIVATE    "/MD$<$<CONFIG:Debug>:d>")
      target_link_options(${PROJECT_NAME} PRIVATE    "/DEFAULTLIB:msvcrt$<$<CONFIG:Debug>:d>")
      target_link_options(${PROJECT_NAME} PRIVATE    "/SUBSYSTEM:WINDOWS")
      target_link_options(${PROJECT_NAME} PRIVATE    "/bigobj")
      target_link_options(${PROJECT_NAME} PRIVATE    "/IMPLIB:${LIBRARY_OUTPUT_PATH}/${PROJECT_NAME}.lib")
      target_link_options(${PROJECT_NAME} PRIVATE    "/DEF:${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}.def")


   else()


      target_link_options(${PROJECT_NAME} PRIVATE "-mwindows")
      target_sources(${PROJECT_NAME} PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}.def")


   endif()



   ##target_link_options(${PROJECT_NAME}   PRIVATE "/IMPLIB:${LIBRARY_OUTPUT_PATH}/${PROJECT_NAME}.lib")
   ##target_link_options(${PROJECT_NAME}   PRIVATE "/DEF:${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}.def")




endif()


list(APPEND library_include_directories ${CMAKE_CURRENT_SOURCE_DIR})
list(APPEND library_include_directories ${CMAKE_CURRENT_SOURCE_DIR}/..)
list(APPEND library_include_directories ${CMAKE_CURRENT_SOURCE_DIR}/../..)
list(APPEND library_include_directories ${CMAKE_CURRENT_SOURCE_DIR}/include/pixman)
list(APPEND library_include_directories ${CMAKE_CURRENT_SOURCE_DIR}/../../../operating_system/operating_system-${PLATFORM_NAME}/configuration/${PROJECT_NAME})

if(__ANDROID__)
  set(ANDROID_CPU_FEATURES_INCLUDE_DIR "${ANDROID_NDK}/sources/android/cpufeatures")
  if(EXISTS "${ANDROID_CPU_FEATURES_INCLUDE_DIR}/cpu-features.h")
    list(APPEND library_include_directories ${ANDROID_CPU_FEATURES_INCLUDE_DIR})
  endif()
  if((IS_ARM_NEON OR IS_ARM_SIMD) AND EXISTS "${ANDROID_CPU_FEATURES_INCLUDE_DIR}/cpu-features.c")
    target_sources(${PROJECT_NAME} PRIVATE "${ANDROID_CPU_FEATURES_INCLUDE_DIR}/cpu-features.c")
    target_sources(static_${PROJECT_NAME} PRIVATE "${ANDROID_CPU_FEATURES_INCLUDE_DIR}/cpu-features.c")
  endif()
endif()


target_include_directories(${PROJECT_NAME} PRIVATE ${library_include_directories})
target_include_directories(static_${PROJECT_NAME} PRIVATE ${library_include_directories})



if(__REACTOS__)
    target_include_directories(pixman BEFORE PRIVATE "${CMAKE_CURRENT_BINARY_DIR}" "${CMAKE_CURRENT_BINARY_DIR}/generated" "${CMAKE_CURRENT_SOURCE_DIR}/include")
    target_include_directories(pixman INTERFACE "${CMAKE_CURRENT_BINARY_DIR}/generated" "${CMAKE_CURRENT_SOURCE_DIR}/include" "${CMAKE_CURRENT_SOURCE_DIR}/include/pixman")
    set_target_properties(static_pixman PROPERTIES EXCLUDE_FROM_ALL TRUE)
endif()
