cmake_minimum_required(VERSION 3.1.0)

INCLUDE(FindPkgConfig)

include_directories(${PROJECT_BINARY_DIR}/frontend)
include_directories(${PROJECT_BINARY_DIR}/backend)
include_directories(${PROJECT_BINARY_DIR})

include_directories(${CMAKE_BINARY_DIR})

if(APPLE)
    file( WRITE ${PROJECT_BINARY_DIR}/osx_keychain.h "#define OSX_KEYCHAIN 1" )
else()
    file( WRITE ${PROJECT_BINARY_DIR}/osx_keychain.h "#define OSX_KEYCHAIN 0" )
endif()

message(STATUS "Checking for modules 'Qt5Widgets','Qt5Core' and 'Qt5LinguistTools'")

find_package(Qt5Widgets REQUIRED)
find_package(Qt5Core REQUIRED)
find_package(Qt5LinguistTools REQUIRED)

message(STATUS "  Found Qt5Widgets, version ${Qt5Widgets_VERSION}")
message(STATUS "  Found Qt5Core, version ${Qt5Core_VERSION}")
message(STATUS "  Found Qt5LinguistTools, version ${Qt5LinguistTools_VERSION}")

set(CMAKE_INCLUDE_CURRENT_DIR ON)

include_directories(${Qt5Widgets_INCLUDE_DIRS})
include_directories(${Qt5Core_INCLUDE_DIRS}})

add_definitions(${Qt5Widgets_DEFINITIONS})
add_definitions(${Qt5Core_DEFINITIONS})

if(NOSECRETSUPPORT)
    SET(SECRET_SUPPORT "false")
    file(WRITE ${PROJECT_BINARY_DIR}/storage_manager.h "\n#define HAS_SECRET_SUPPORT 0\n")
else()
    pkg_check_modules(LIBSECRET libsecret-1 QUIET)
    if(LIBSECRET_FOUND)
        add_subdirectory(libsecret)
        SET(SECRET_SUPPORT "true")
        file(WRITE ${PROJECT_BINARY_DIR}/storage_manager.h "\n#define HAS_SECRET_SUPPORT 1\n")
    else()
        SET(SECRET_SUPPORT "false")
        file(WRITE ${PROJECT_BINARY_DIR}/storage_manager.h "\n#define HAS_SECRET_SUPPORT 0\n")
    endif()
endif()

add_definitions(-std=c++14)

if(NOKDESUPPORT)
    SET(KWALLET_SUPPORT "false")
    file(APPEND  ${PROJECT_BINARY_DIR}/storage_manager.h "\n#define HAS_KWALLET_SUPPORT 0\n")
else()
    message(STATUS "Checking for modules 'KF5Wallet'")
    find_package(KF5Wallet QUIET)
    if(KF5Wallet_FOUND )
        message(STATUS "  Found KF5Wallet, version ${KF5Wallet_VERSION}")
        file(APPEND  ${PROJECT_BINARY_DIR}/storage_manager.h "\n#define HAS_KWALLET_SUPPORT 1\n")
        SET(KWALLET_SUPPORT "true")
    else()
        file(APPEND  ${PROJECT_BINARY_DIR}/storage_manager.h "\n#define HAS_KWALLET_SUPPORT 0\n")
        SET(KWALLET_SUPPORT "false")
    endif()
endif()

if(KWALLET_SUPPORT)
    message(STATUS "\n--------------------------------------------------------------------------")
    message(STATUS "kwallet support found,will build kwallet functionality")
    message(STATUS "-----------------------------------------------------------------------")
else()
    message(STATUS "\n--------------------------------------------------------------------------")
    message(STATUS "kwallet support NOT found,will not build kwallet functionality")
    message(STATUS "-----------------------------------------------------------------------")
endif()

if(SECRET_SUPPORT)
    message(STATUS "\n--------------------------------------------------------------------------")
    message(STATUS "libsecret support found,will build libsecret functionality")
    message(STATUS "-----------------------------------------------------------------------")
else()
    message(STATUS "\n--------------------------------------------------------------------------")
    message(STATUS "libsecret support NOT found,will not build libsecret functionality")
    message(STATUS "-----------------------------------------------------------------------")
endif()

set(COMMON_UI_FILES  changepassworddialog.ui password_dialog.ui)
set(COMMON_MOC_FILES task.h changepassworddialog.h password_dialog.h)
set(COMMON_SRC_FILES changepassworddialog.cpp password_dialog.cpp lxqt_windows_dpapi.cpp lxqt_internal_wallet.cpp lxqt_wallet.cpp lxqt_osx_keychain.cpp)

Qt5_WRAP_UI(UI ${COMMON_UI_FILES})
Qt5_WRAP_CPP(MOC ${COMMON_MOC_FILES})

if(KWALLET_SUPPORT)
    Qt5_WRAP_CPP(MOC_KWALLET lxqt_kwallet.h)
    set(KWALLET_COMMON lxqt_kwallet.cpp)
    if(SECRET_SUPPORT)
        add_library(lxqtwallet SHARED ${KWALLET_COMMON} lxqt_libsecret.cpp ${UI} ${COMMON_SRC_FILES} ${MOC} ${MOC_KWALLET})
        target_link_libraries(lxqtwallet secretlib)
    else()
        add_library(lxqtwallet SHARED ${KWALLET_COMMON} ${UI} ${COMMON_SRC_FILES} ${MOC} ${MOC_KWALLET})
    endif()
    target_link_libraries(lxqtwallet KF5::Wallet)
else()
    if(SECRET_SUPPORT)
        add_library(lxqtwallet SHARED lxqt_libsecret.cpp ${UI} ${COMMON_SRC_FILES} ${MOC})
        target_link_libraries(lxqtwallet secretlib)
    else()
        add_library(lxqtwallet SHARED ${UI} ${COMMON_SRC_FILES} ${MOC} ${MOC_KWALLET})
    endif()
endif()

target_link_libraries(lxqtwallet lxqtwallet-backend)

target_link_libraries(lxqtwallet ${Qt5Widgets_LIBRARIES})
target_link_libraries(lxqtwallet ${Qt5Core_LIBRARIES})

if(WIN32)
    target_link_libraries(lxqtwallet -lcrypt32 -lbcrypt)
endif()

if(APPLE)
    find_library( COREFOUNDATION_LIBRARY CoreFoundation REQUIRED )
    find_library( SECURITY_LIBRARY Security REQUIRED )
    target_link_libraries(lxqtwallet ${COREFOUNDATION_LIBRARY} ${SECURITY_LIBRARY})
endif()

set_target_properties(lxqtwallet PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64 -Wextra -Wall -s -fPIC -pedantic ")
set_target_properties(lxqtwallet PROPERTIES SOVERSION "${LIBRARY_VERSION}")

set_target_properties(lxqtwallet PROPERTIES OUTPUT_NAME lxqt-wallet)

file( WRITE ${PROJECT_BINARY_DIR}/translations_path.h
"
#define TRANSLATIONS_PATH \"${CMAKE_INSTALL_PREFIX}/share/lxqt/translations/lxqt-wallet/\"
")

SET(TRANSLATIONS
   ../translations/lxqt-wallet_nl_NL.ts
   ../translations/lxqt-wallet_fr_FR.ts
   ../translations/lxqt-wallet_de_DE.ts
   ../translations/lxqt-wallet_gr_GR.ts
   ../translations/lxqt-wallet_es_ES.ts
   ../translations/lxqt-wallet_pl_PL.ts
)

QT5_ADD_TRANSLATION(QM_FILES ${TRANSLATIONS})
SET(QM_FILES ${QM_FILES} PARENT_SCOPE)
ADD_CUSTOM_TARGET(translations ALL DEPENDS ${QM_FILES})

install(FILES ${CMAKE_BINARY_DIR}/frontend/lxqt-wallet_de_DE.qm DESTINATION share/lxqt/translations/lxqt-wallet/)
install(FILES ${CMAKE_BINARY_DIR}/frontend/lxqt-wallet_gr_GR.qm DESTINATION share/lxqt/translations/lxqt-wallet/)
install(FILES ${CMAKE_BINARY_DIR}/frontend/lxqt-wallet_nl_NL.qm DESTINATION share/lxqt/translations/lxqt-wallet/)
install(FILES ${CMAKE_BINARY_DIR}/frontend/lxqt-wallet_fr_FR.qm DESTINATION share/lxqt/translations/lxqt-wallet/)
install(FILES ${CMAKE_BINARY_DIR}/frontend/lxqt-wallet_es_ES.qm DESTINATION share/lxqt/translations/lxqt-wallet/)
install(FILES ${CMAKE_BINARY_DIR}/frontend/lxqt-wallet_pl_PL.qm DESTINATION share/lxqt/translations/lxqt-wallet/)

install(TARGETS lxqtwallet RUNTIME LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}")

install(FILES lxqt_wallet.h DESTINATION "${CMAKE_INSTALL_PREFIX}/include/lxqt")

file( WRITE ${PROJECT_BINARY_DIR}/lxqt-wallet.pc
"prefix=${CMAKE_INSTALL_PREFIX}
exec_prefix=${CMAKE_INSTALL_PREFIX}
libdir=${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}
includedir=${CMAKE_INSTALL_PREFIX}/include/lxqt

Name: lxqt_wallet
Description: lxqt secure storage system
Version: ${LIBRARY_VERSION}
Requires: Qt5Core Qt5Widgets
Libs: -L${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR} -llxqt-wallet
Cflags: -I${CMAKE_INSTALL_PREFIX}/include/lxqt
\n")

install(FILES ${PROJECT_BINARY_DIR}/lxqt-wallet.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig/ PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ)
