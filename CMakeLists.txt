cmake_minimum_required(VERSION 2.4)
project(postsrsd C)

option(GENERATE_SRS_SECRET "Generate a random SRS secret if none exists during install" ON)
option(USE_APPARMOR "Enable AppArmor profile" OFF)

set(CHROOT_DIR "${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}" CACHE PATH "Chroot jail for daemon")

find_program(HELP2MAN help2man DOC "path to help2man executable")
find_program(DD dd DOC "path to dd executable")
find_program(BASE64 base64 DOC "path to base64 executable")
find_program(INSSERV insserv DOC "path to insserv executable")

if(NOT DEFINED INIT_FLAVOR)
    if(IS_DIRECTORY "/etc/init" AND EXISTS "/lib/init/upstart-job")
        message(STATUS "Detected init flavor: upstart")
        set(INIT_FLAVOR "upstart" CACHE STRING "Init daemon of this system")
    elseif(IS_DIRECTORY "/etc/init.d")
        message(STATUS "Detected init flavor: sysv")
        set(INIT_FLAVOR "sysv" CACHE STRING "Init daemon of this system")
    elseif(IS_DIRECTORY "/usr/local/etc/rc.d")
        message(STATUS "Detected init flavor: freebsd")
        set(INIT_FLAVOR "freebsd" CACHE STRING "Init daemon of this system")
    else()
        message(STATUS "Detected init flavor: none")
        message(STATUS "System startup files will not be installed")
        set(INIT_FLAVOR "" CACHE STRING "Init daemon of this system")
    endif()
endif()

add_executable(${PROJECT_NAME} postsrsd.c sha1.c srs2.c)

get_target_property(POSTSRSD ${PROJECT_NAME} LOCATION)
get_filename_component(POSTSRSD ${POSTSRSD} NAME_WE)
set(APPARMOR_PROFILE "${CMAKE_INSTALL_PREFIX}/sbin/${POSTSRSD}")
string(REGEX REPLACE "^/+" "" APPARMOR_PROFILE "${APPARMOR_PROFILE}")
string(REPLACE "/" "." APPARMOR_PROFILE "${APPARMOR_PROFILE}")

configure_file(${PROJECT_NAME}.init.in ${PROJECT_NAME}.init @ONLY)
configure_file(${PROJECT_NAME}.upstart.in ${PROJECT_NAME}.upstart @ONLY)
configure_file(${PROJECT_NAME}.freebsd.in ${PROJECT_NAME}.freebsd @ONLY)
configure_file(${PROJECT_NAME}.apparmor.in ${PROJECT_NAME}.apparmor @ONLY)
configure_file(postinstall.cmake.in postinstall.cmake @ONLY)

if(HELP2MAN)
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
	COMMAND ${HELP2MAN} ARGS -s8 -o${PROJECT_NAME}.8 -n "Postfix Sender Rewriting Scheme daemon" -N -h-h -v-v ${CMAKE_CURRENT_BINARY_DIR}/${POSTSRSD}
	VERBATIM
)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.8 DESTINATION "share/man/man8")
endif()

if(USE_APPARMOR)
	install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.apparmor DESTINATION "/etc/apparmor.d" RENAME "${APPARMOR_PROFILE}")
endif()

install(TARGETS ${PROJECT_NAME} DESTINATION "sbin")
install(FILES README.md main.cf.ex DESTINATION "share/doc/${PROJECT_NAME}")
install(SCRIPT "${CMAKE_CURRENT_BINARY_DIR}/postinstall.cmake")

