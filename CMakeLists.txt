# Copyright (C) 2026- Stazed
#
# This file is part of New-Session-Manager.
# New-Session-Manager is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.

cmake_minimum_required(VERSION 3.20)

# Keep the version on its own line for automatic updates.
project(new-session-manager
    VERSION 1.6.1
    DESCRIPTION "New-Session-Manager"
    LANGUAGES C CXX
)

include(GNUInstallDirs)
include(CTest)
find_package(PkgConfig REQUIRED)
find_package(Threads REQUIRED)

option(JACKPATCH "Build jackpatch" ON)
option(NSM_PROXY "Build nsm-proxy and its FLTK GUI" ON)
option(NSM_LEGACY_GUI "Build the legacy FLTK GUI" ON)

# liblo is required by all project executables.
pkg_check_modules(LIBLO REQUIRED IMPORTED_TARGET liblo)

if(JACKPATCH)
    pkg_check_modules(JACK REQUIRED IMPORTED_TARGET jack)
endif()

# FLTK is only needed when building one of its GUI executables.
if(NSM_PROXY OR NSM_LEGACY_GUI)
    find_package(FLTK 1.4 CONFIG REQUIRED)
endif()

if(NSM_PROXY)
    find_program(FLUID_EXECUTABLE fluid REQUIRED)
    if(NOT TARGET fltk::images)
        message(FATAL_ERROR "FLTK 1.4 was found, but its fltk::images target is unavailable")
    endif()
endif()

add_executable(nsmd
    src/nsmd.cpp
    src/debug.cpp
    src/Endpoint.cpp
    src/file.cpp
    src/Thread.cpp
)
target_link_libraries(nsmd PRIVATE PkgConfig::LIBLO Threads::Threads)

install(TARGETS nsmd RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
install(FILES docs/src/nsmd.1 DESTINATION "${CMAKE_INSTALL_MANDIR}/man1")
install(FILES
    docs/index.html
    CHANGELOG
    README.md
    DESTINATION "${CMAKE_INSTALL_DATADIR}/doc/new-session-manager"
)
install(FILES docs/api/index.html
    DESTINATION "${CMAKE_INSTALL_DATADIR}/doc/new-session-manager/api"
)

if(JACKPATCH)
    add_executable(jackpatch src/jackpatch.c)
    target_link_libraries(jackpatch PRIVATE PkgConfig::LIBLO PkgConfig::JACK)

    install(TARGETS jackpatch RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
    install(FILES
        src/jackpatch.svg
        DESTINATION "${CMAKE_INSTALL_DATADIR}/icons/hicolor/scalable/apps"
    )
    install(FILES
        src/org.jackaudio.jackpatch.desktop
        DESTINATION "${CMAKE_INSTALL_DATADIR}/applications"
    )
    install(FILES docs/src/jackpatch.1
        DESTINATION "${CMAKE_INSTALL_MANDIR}/man1"
    )
endif()

if(NSM_PROXY)
    set(PROXY_UI_CPP "${CMAKE_CURRENT_BINARY_DIR}/NSM_Proxy_UI.C")
    set(PROXY_UI_H "${CMAKE_CURRENT_BINARY_DIR}/NSM_Proxy_UI.H")

    add_custom_command(
        OUTPUT "${PROXY_UI_CPP}" "${PROXY_UI_H}"
        COMMAND "${FLUID_EXECUTABLE}" -c -o "${PROXY_UI_CPP}"
                "${CMAKE_CURRENT_SOURCE_DIR}/src/NSM_Proxy_UI.fl"
        COMMAND "${FLUID_EXECUTABLE}" -c -h "${PROXY_UI_H}"
                "${CMAKE_CURRENT_SOURCE_DIR}/src/NSM_Proxy_UI.fl"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/src/NSM_Proxy_UI.fl"
        WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}"
        VERBATIM
    )


    add_executable(nsm-proxy
        src/nsm-proxy.cpp
        src/debug.cpp
    )
    target_link_libraries(nsm-proxy PRIVATE PkgConfig::LIBLO Threads::Threads)

    add_executable(nsm-proxy-gui
        src/nsm-proxy-gui.cpp
        "${PROXY_UI_CPP}"
        "${PROXY_UI_H}"
    )

    target_include_directories(nsm-proxy-gui PRIVATE
    "${CMAKE_CURRENT_BINARY_DIR}"
    )

    target_link_libraries(nsm-proxy-gui PRIVATE
        fltk::fltk
        PkgConfig::LIBLO
        Threads::Threads
    )

    install(TARGETS nsm-proxy nsm-proxy-gui
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    )
    install(FILES
        src/nsm-proxy.svg
        DESTINATION "${CMAKE_INSTALL_DATADIR}/icons/hicolor/scalable/apps"
    )
    install(FILES
        src/org.jackaudio.nsm-proxy.desktop
        DESTINATION "${CMAKE_INSTALL_DATADIR}/applications"
    )
    install(FILES
        docs/src/nsm-proxy.1
        docs/src/nsm-proxy-gui.1
        DESTINATION "${CMAKE_INSTALL_MANDIR}/man1"
    )
endif()

if(NSM_LEGACY_GUI)
    add_executable(nsm-legacy-gui
        src/nsm-legacy-gui.cpp
        src/debug.cpp
        src/Endpoint.cpp
        src/Thread.cpp
        src/FL/Fl_Scalepack.C
    )
    target_link_libraries(nsm-legacy-gui PRIVATE
        fltk::fltk
        fltk::images
        PkgConfig::LIBLO
        Threads::Threads
    )

    install(TARGETS nsm-legacy-gui
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    )
    install(FILES
        src/org.jackaudio.nsm-legacy-gui.desktop
        DESTINATION "${CMAKE_INSTALL_DATADIR}/applications"
    )
    install(FILES
        src/nsm-legacy-gui.svg
        DESTINATION "${CMAKE_INSTALL_DATADIR}/icons/hicolor/scalable/apps"
    )
    install(FILES
        docs/src/nsm-legacy-gui.1
        docs/src/non-session-manager.1
        DESTINATION "${CMAKE_INSTALL_MANDIR}/man1"
    )

    # Match Meson's one-way install-time symlink.
    install(CODE "
        execute_process(
            COMMAND \"${CMAKE_COMMAND}\" -E create_symlink
                nsm-legacy-gui
                \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_BINDIR}/non-session-manager\"
            COMMAND_ERROR_IS_FATAL ANY
        )
    ")
endif()
