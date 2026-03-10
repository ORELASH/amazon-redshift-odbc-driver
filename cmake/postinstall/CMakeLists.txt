if (WIN32)
    # Set environment variables for package script
    set(ENV{DEPENDENCIES_INSTALL_DIR} "D:/deps")
    set(ENV{RS_BUILD_TYPE} "")
    install(CODE "execute_process(COMMAND cmd /c \"set DEPENDENCIES_INSTALL_DIR=D:\\\\deps && set RS_BUILD_TYPE= && ${CMAKE_SOURCE_DIR}/package64.bat\"  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})")
endif()
