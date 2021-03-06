set(CMAKE_AUTOMOC ON)
set(CMD_NAME deepin-movie)

set(PROJECT_INCLUDE ${PROJECT_SOURCE_DIR}/src/widgets
    ${PROJECT_SOURCE_DIR}/src/common
    ${PROJECT_SOURCE_DIR}/src/libdmr)

include_directories(${CMAKE_INCLUDE_CURRENT_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR})

find_package(Qt5Widgets)
find_package(Qt5DBus)
find_package(Qt5X11Extras)
find_package(Qt5LinguistTools)
find_package(Qt5Network)
find_package(Qt5Concurrent)
find_package(Qt5Sql)

pkg_check_modules(Dtk REQUIRED IMPORTED_TARGET dtkwidget)
pkg_check_modules(Mpv REQUIRED IMPORTED_TARGET mpv)
pkg_check_modules(Xcb REQUIRED IMPORTED_TARGET xcb xcb-aux
    xcb-proto xcb-ewmh xcb-shape)
pkg_check_modules(AV REQUIRED IMPORTED_TARGET libavformat
    libavutil libavcodec libavresample)
# IMPORTED_TARGET failed to work for some of libs under flatpak env
pkg_check_modules(Other REQUIRED libffmpegthumbnailer
    libpulse libpulse-simple openssl dvdnav)

qt5_add_resources(RCS resources.qrc)
if (${Dtk_VERSION} LESS 2.0.6.1)
    qt5_add_resources(RCS theme.qrc)
endif()


file(GLOB_RECURSE SRCS LIST_DIRECTORIES false common/*.cpp widgets/*.cpp libdmr/*.cpp)
list(APPEND SRCS main.cpp)

# mpv backend
file(GLOB_RECURSE MPV_SRCS LIST_DIRECTORIES false backends/mpv/*.cpp)
list(APPEND SRCS ${MPV_SRCS})
list(APPEND PROJECT_INCLUDE ${PROJECT_SOURCE_DIR}/src/backends/mpv)
#~

add_custom_target(json_i18n ALL
    /usr/lib/dtk2/dtk-settings resources/data/settings.json -o common/settings_translation.cpp
    COMMAND sed -i "s,#include .*,#include <DSettings>," common/settings_translation.cpp
    VERBATIM
    SOURCES resources/data/settings.json
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/src/)

## translations
file(GLOB TS LIST_DIRECTORIES false translations/${CMD_NAME}*.ts)

set_source_files_properties(${TS}
    PROPERTIES OUTPUT_LOCATION ${PROJECT_SOURCE_DIR}/src/translations)

qt5_create_translation(QM ${SRCS} ${TS})
# ~

add_executable(${CMD_NAME} ${SRCS} ${RCS} ${QM})
add_dependencies(${CMD_NAME} json_i18n)
target_include_directories(${CMD_NAME} PUBLIC ${PROJECT_INCLUDE})

set(TARGET_LIBS X11 Xext Xtst PkgConfig::Xcb Qt5::Widgets Qt5::X11Extras Qt5::Network
    Qt5::Concurrent Qt5::DBus Qt5::Sql PkgConfig::Dtk PkgConfig::Mpv
    PkgConfig::AV pthread GL)
target_link_libraries(${CMD_NAME} ${TARGET_LIBS} ${Other_LIBRARIES})


install(TARGETS ${CMD_NAME} DESTINATION bin)
install(DIRECTORY ${PROJECT_SOURCE_DIR}/src/translations
    DESTINATION share/${CMD_NAME}
    FILES_MATCHING PATTERN "*.qm")
install(FILES ${PROJECT_SOURCE_DIR}/src/deepin-movie.desktop
    DESTINATION share/applications)
install(FILES ${PROJECT_SOURCE_DIR}/src/resources/icons/logo-big.svg
    DESTINATION share/icons/hicolor/scalable/apps
    RENAME deepin-movie.svg)

install(DIRECTORY ${PROJECT_SOURCE_DIR}/src/dman/deepin-movie DESTINATION share/dman)

add_subdirectory(libdmr)
add_subdirectory(test)
