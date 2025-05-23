add_compile_definitions(
        DATADIR_PATH="${PROJECTM_DATADIR_PATH}"
        $<IF:$<PLATFORM_ID:Darwin>,GL_SILENCE_DEPRECATION,>
        $<IF:$<PLATFORM_ID:Windows>,NOMINMAX,>
        $<IF:$<PLATFORM_ID:Windows>,WIN32_LEAN_AND_MEAN,>
        $<IF:$<PLATFORM_ID:Windows>,STBI_NO_DDS,>
        )

add_subdirectory(Audio)
add_subdirectory(MilkdropPreset)
add_subdirectory(Renderer)
add_subdirectory(UserSprites)

add_library(projectM_main OBJECT
        "${PROJECTM_EXPORT_HEADER}"
        Preset.hpp
        PresetFactory.cpp
        PresetFactory.hpp
        PresetFactoryManager.cpp
        PresetFactoryManager.hpp
        PresetFileParser.cpp
        PresetFileParser.hpp
        ProjectM.cpp
        ProjectM.hpp
        ProjectMCWrapper.cpp
        ProjectMCWrapper.hpp
        TimeKeeper.cpp
        TimeKeeper.hpp
        Utils.cpp
        Utils.hpp
        projectM-opengl.h
        )

target_link_libraries(projectM_main
        PUBLIC
        Audio
        MilkdropPreset
        Renderer
        hlslparser
        SOIL2
        libprojectM::API
        ${PROJECTM_FILESYSTEM_LIBRARY}
        )

if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    target_link_libraries(projectM_main
            PUBLIC
            "-framework CoreFoundation"
            )
endif()

target_include_directories(projectM_main
        PRIVATE
        "${PROJECTM_SOURCE_DIR}/src"
        "${CMAKE_CURRENT_SOURCE_DIR}"
        "${CMAKE_CURRENT_SOURCE_DIR}/Renderer"
        "${PROJECTM_SOURCE_DIR}/vendor/hlslparser/src"
        "${CMAKE_CURRENT_SOURCE_DIR}/MilkdropPreset"
        "${MSVC_EXTRA_INCLUDE_DIR}"
        )

# CMake cannot combine multiple static libraries using target_link_libraries.
# This syntax will pull in the compiled object files into the final library.
add_library(projectM
        ${PROJECTM_DUMMY_SOURCE_FILE} # CMake needs at least one "real" source file.
        $<TARGET_OBJECTS:Audio>
        $<TARGET_OBJECTS:MilkdropPreset>
        $<TARGET_OBJECTS:Renderer>
        $<TARGET_OBJECTS:UserSprites>
        $<TARGET_OBJECTS:hlslparser>
        $<TARGET_OBJECTS:SOIL2>
        $<TARGET_OBJECTS:projectM_main>
        $<TARGET_OBJECTS:projectM::Eval>
        )

target_include_directories(projectM
        PUBLIC
        "$<INSTALL_INTERFACE:${PROJECTM_INCLUDE_DIR}>"
        )

target_link_libraries(projectM
        PUBLIC
        ${PROJECTM_OPENGL_LIBRARIES}
        libprojectM::API
        ${PROJECTM_FILESYSTEM_LIBRARY}
        )

if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    target_link_libraries(projectM
            PUBLIC
            "-framework CoreFoundation"
            )
endif()

set_target_properties(projectM PROPERTIES
        VERSION "${PROJECTM_LIB_VERSION}"
        SOVERSION "${PROJECTM_SO_VERSION}"
        FOLDER libprojectM
        OUTPUT_NAME ${PROJECTM_LIBRARY_BASE_OUTPUT_NAME}
        )

if(BUILD_SHARED_LIBS)
    target_compile_definitions(projectM_main
            PRIVATE
            projectM_api_EXPORTS
            )

    target_link_libraries(projectM
            PUBLIC
            ${CMAKE_DL_LIBS}
            )
else()
    target_compile_definitions(projectM_main
            PUBLIC
            PROJECTM_STATIC_DEFINE
            )

    set_target_properties(projectM PROPERTIES
            OUTPUT_NAME ${PROJECTM_LIBRARY_BASE_OUTPUT_NAME}
            FOLDER libprojectM
            )
endif()

add_library(libprojectM::projectM ALIAS projectM)

if(ENABLE_INSTALL)

    install(TARGETS projectM
            EXPORT libprojectMTargets
            LIBRARY DESTINATION "${PROJECTM_LIB_DIR}" COMPONENT Runtime
            RUNTIME DESTINATION "${PROJECTM_RUNTIME_DIR}" COMPONENT Runtime
            ARCHIVE DESTINATION "${PROJECTM_LIB_DIR}" COMPONENT Devel
            PUBLIC_HEADER DESTINATION "${PROJECTM_INCLUDE_DIR}/libprojectM" COMPONENT Devel
            )

    if(ENABLE_CXX_INTERFACE)
        install(FILES
                Audio/PCM.hpp
                ProjectM.hpp
                DESTINATION "${PROJECTM_INCLUDE_DIR}/projectM-4"
                COMPONENT Devel
                )
    else()
        # Set PROJECTM_STATIC_EXPORT for C++ implementations to use project default visibility
        # and no dllimport/dllexport.
        set_source_files_properties(ProjectM.cpp Audio/PCM.cpp PROPERTIES
                COMPILE_DEFINITIONS PROJECTM_STATIC_DEFINE
                )
        target_compile_definitions(projectM
                INTERFACE
                PROJECTM_STATIC_DEFINE
                )
    endif()


    # CMake target exports

    # For use from a local projectM build tree (without installing)
    export(TARGETS
            projectM_api
            projectM
            NAMESPACE libprojectM::
            FILE projectM-exports.cmake
            )


    # For use from an installed package (system install, vcpkg, homebrew etc.)
    include(CMakePackageConfigHelpers)

    write_basic_package_version_file(
            "${CMAKE_CURRENT_BINARY_DIR}/libprojectM/projectM4ConfigVersion.cmake"
            VERSION ${PROJECT_VERSION}
            COMPATIBILITY AnyNewerVersion
            )

    configure_package_config_file(projectM4Config.cmake.in
            "${CMAKE_CURRENT_BINARY_DIR}/libprojectM/projectM4Config.cmake"
            INSTALL_DESTINATION "${PROJECTM_LIB_DIR}/cmake/projectM4"
            PATH_VARS PROJECTM_BIN_DIR PROJECTM_INCLUDE_DIR
            )

    install(FILES
            "${CMAKE_CURRENT_BINARY_DIR}/libprojectM/projectM4ConfigVersion.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/libprojectM/projectM4Config.cmake"
            DESTINATION "${PROJECTM_LIB_DIR}/cmake/projectM4"
            COMPONENT Devel
            )

    if(NOT ENABLE_EMSCRIPTEN AND ENABLE_GLES)
        install(FILES
                "${PROJECTM_SOURCE_DIR}/cmake/gles/FindOpenGL.cmake"
                DESTINATION "${PROJECTM_LIB_DIR}/cmake/projectM4"
                COMPONENT Devel
                )
    endif()

    install(EXPORT libprojectMTargets
            FILE projectM4Targets.cmake
            NAMESPACE libprojectM::
            DESTINATION "${PROJECTM_LIB_DIR}/cmake/projectM4"
            COMPONENT Devel
            )

    # pkg-config export, only supported on UNIX systems.
    if(UNIX)
        include(GeneratePkgConfigFiles)

        if(ENABLE_BOOST_FILESYSTEM)
            # Since there are many different CMake scripts out there to find Boost, e.g. Gentoo has its own Find module,
            # it's very hard to extract the proper library names and paths from the Boost package targets and convert them
            # into pkgconfig's expected format.
            # We will just assume the build uses Boost from the default location (e.g. not a custom CMAKE_PREFIX_PATH) and
            # the library name is correct.
            set(PKGCONFIG_LIBS "${PKGCONFIG_LIBS} -l:boost_filesystem")
        endif()

        set(PKGCONFIG_PACKAGE_NAME "${PROJECTM_LIBRARY_BASE_OUTPUT_NAME}")
        set(PKGCONFIG_PACKAGE_DESCRIPTION "projectM Music Visualizer")
        set(PKGCONFIG_PACKAGE_REQUIREMENTS_ALL "opengl")

        generate_pkg_config_files(projectM ${PROJECTM_LIBRARY_BASE_OUTPUT_NAME})

    endif()

endif()
