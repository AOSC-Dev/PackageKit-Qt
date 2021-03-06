# CMakeLists for PackageKit-Qt library
# Copyright (C) 2014 Leslie Zhai <xiang.zhai@i-soft.com.cn>

if (USE_QT5)
  find_package(Qt5Core REQUIRED)
  set(Qt5_FOUND YES)
else ()
  find_package(Qt4 REQUIRED)
  set(Qt4_FOUND YES)
endif ()

# Set up Qt
if (Qt5_FOUND)
  find_package(Qt5DBus REQUIRED)
  find_package(Qt5Sql REQUIRED)
  include_directories(${Qt5Core_INCLUDE_DIRS}
                      ${Qt5DBus_INCLUDE_DIRS}
                      ${Qt5Sql_INCLUDE_DIRS}
  )
  add_definitions(${Qt5DBus_DEFINITIONS})
  add_definitions(${Qt5Sql_DEFINITIONS})
elseif (Qt4_FOUND)
  set(QT_USE_QTDBUS TRUE)
  set(QT_USE_QTSQL TRUE)
  set(QT_DONT_USE_QTGUI TRUE)
  include(${QT_USE_FILE})
endif ()

include_directories(${CMAKE_BINARY_DIR}
		    ${CMAKE_CURRENT_BINARY_DIR}
		    ${CMAKE_CURRENT_SOURCE_DIR}
)

set(packagekitqt_HEADERS
    Daemon
    Transaction
    common.h
    daemon.h
    transaction.h
    bitfield.h
)

set(packagekitqt_HEADERS_PRIVATE
    daemonprivate.h
    transactionprivate.h
)

set(packagekitqt_SRC
	bitfield.cpp
	daemon.cpp
    daemonprivate.cpp
	transaction.cpp
	transactionprivate.cpp
)

find_file(PK_INTERFACE_XML org.freedesktop.PackageKit.xml
	PATHS ${CMAKE_INSTALL_PREFIX}/share/dbus-1/interfaces/
)
find_file(PK_TRANSACTION_INTERFACE_XML org.freedesktop.PackageKit.Transaction.xml
	PATHS ${CMAKE_INSTALL_PREFIX}/share/dbus-1/interfaces/
)
if (PK_INTERFACE_XML STREQUAL "PK_INTERFACE_XML-NOTFOUND" OR PK_TRANSACTION_INTERFACE_XML STREQUAL "PK_TRANSACTION_INTERFACE_XML-NOTFOUND")
	message (FATAL_ERROR "Unable to find PackageKit DBus specifications! Please install PackageKit to continue!")
endif ()

set_source_files_properties(${PK_INTERFACE_XML} PROPERTIES CLASSNAME DaemonProxy)
set_source_files_properties(${PK_TRANSACTION_INTERFACE_XML} PROPERTIES CLASSNAME TransactionProxy)

if (Qt5_FOUND)
  qt5_add_dbus_interface(packagekitqt_SRC ${PK_INTERFACE_XML} daemonproxy)
  qt5_add_dbus_interface(packagekitqt_SRC ${PK_TRANSACTION_INTERFACE_XML} transactionproxy)
  qt5_generate_moc(daemon.h daemon.moc)
  qt5_generate_moc(transaction.h transaction.moc)
  add_custom_target(mocs DEPENDS daemon.moc transaction.moc)

  set(LIBNAME "packagekit-qt5")
elseif (Qt4_FOUND)
  qt4_add_dbus_interface(packagekitqt_SRC ${PK_INTERFACE_XML} daemonproxy)
  qt4_add_dbus_interface(packagekitqt_SRC ${PK_TRANSACTION_INTERFACE_XML} transactionproxy)
  qt4_automoc(${packagekitqt_SRC})
  qt4_wrap_cpp(packagekitqt_MOC_SRC ${packagekitqt_HEADERS} ${packagekitqt_HEADERS_PRIVATE})

  set(LIBNAME "packagekit-qt2")
endif ()

add_library(${LIBNAME} SHARED ${packagekitqt_SRC} ${packagekitqt_HEADERS} ${packagekitqt_HEADERS_PRIVATE})
set_target_properties(${LIBNAME} PROPERTIES VERSION ${QPACKAGEKIT_VERSION} SOVERSION ${QPACKAGEKIT_API_LEVEL})

if (Qt5_FOUND)
  add_dependencies(${LIBNAME} mocs)
  target_link_libraries(${LIBNAME}
		${Qt5DBus_LIBRARIES} ${Qt5Sql_LIBRARIES}
  )
elseif (Qt4_FOUND)
  target_link_libraries(${LIBNAME}
		${QT_LIBRARIES}
  )
endif ()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/${LIBNAME}.pc.in
  ${CMAKE_CURRENT_BINARY_DIR}/${LIBNAME}.pc
  @ONLY
)

install(TARGETS ${LIBNAME} DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${LIBNAME}.pc
	DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)
install(FILES ${packagekitqt_HEADERS}
        DESTINATION include/PackageKit/${LIBNAME}
)

add_subdirectory(modules)
