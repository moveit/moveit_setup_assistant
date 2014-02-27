cmake_minimum_required(VERSION 2.8.3)
project(moveit_setup_assistant)

if(NOT CMAKE_CONFIGURATION_TYPES AND NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

include_directories(include)

find_package(Boost REQUIRED COMPONENTS thread filesystem system program_options)
include_directories(SYSTEM ${Boost_INCLUDE_DIR})
link_directories(${Boost_LIBRARY_DIRS})

find_package(catkin REQUIRED COMPONENTS
  moveit_ros_planning
  moveit_ros_visualization
  rviz
  urdf
  rosconsole
  roscpp
)
include_directories(${catkin_INCLUDE_DIRS})
link_directories(${catkin_LIBRARY_DIRS})

## Find Ogre
find_package(PkgConfig REQUIRED)
pkg_check_modules(OGRE_OV OGRE OGRE-Overlay)

# Old versions of OGRE (pre 1.9) included OGRE-Overlay in the main package
# (i.e. there was no OGRE-Overlay component).  So if the above
# pkg_check_modules() failed, try looking for just OGRE.
if(NOT OGRE_OV_FOUND)
  pkg_check_modules(OGRE_OV REQUIRED OGRE)
endif(NOT OGRE_OV_FOUND)

## Find OGRE Plugin path (not necessarily platform-independent, I guess)
execute_process(COMMAND
  pkg-config --variable=plugindir OGRE
  OUTPUT_VARIABLE OGRE_PLUGIN_PATH
  OUTPUT_STRIP_TRAILING_WHITESPACE
  )
message(STATUS OGRE_PLUGIN_PATH=${OGRE_PLUGIN_PATH})

# find absolute path of ogre libraries.
# This is stored in the cache to save time on cmake re-run.
# Using absolute paths is necessary if pkg-config finds Ogre in a different
# location than the default.  This can happen when Ogre is built from source,
# or when 2 versions of Ogre are installed.  Using absolute paths ensures that
# components that link against Ogre (rviz and any packages depending on rviz)
# all find the same Ogre shared library.
if(NOT DEFINED OGRE_OV_LIBRARIES_ABS)
  unset(OGRE_OV_LIBRARIES_ABS_TMP)
  foreach(_lib ${OGRE_OV_LIBRARIES})
    set(OGRE_OV_LIB_TAG "OGRE_OV_RVIZ_LIB_${_lib}")

    find_library(${OGRE_OV_LIB_TAG}
            NAMES ${_lib}
            HINTS ${OGRE_OV_LIBRARY_DIRS}
            PATHS ${OGRE_OV_LIBRARY_DIRS}
    )

    set(OGRE_OV_LIBRARIES_ABS_TMP ${OGRE_OV_LIBRARIES_ABS_TMP} ${${OGRE_OV_LIB_TAG}})
  endforeach(_lib)

  set(OGRE_OV_LIBRARIES_ABS ${OGRE_OV_LIBRARIES_ABS_TMP}
        CACHE FILEPATH "Pathname of library ${_lib}")
endif(NOT DEFINED OGRE_OV_LIBRARIES_ABS)

include_directories(${OGRE_OV_INCLUDE_DIRS})
link_directories(${OGRE_OV_LIBRARY_DIRS})

# Qt Stuff
find_package(Qt4 REQUIRED)
include(${QT_USE_FILE})
add_definitions(-DQT_NO_KEYWORDS)
include_directories(${QT_INCLUDE_DIR})
include_directories(${CMAKE_BINARY_DIR})

find_library(YAML yaml-cpp REQUIRED)

# Header files that need Qt Moc pre-processing for use with Qt signals, etc:
set( headers
  src/widgets/navigation_widget.h
  src/widgets/header_widget.h
  src/widgets/setup_assistant_widget.h
  src/widgets/start_screen_widget.h
  src/widgets/planning_groups_widget.h
  src/widgets/double_list_widget.h
  src/widgets/kinematic_chain_widget.h
  src/widgets/group_edit_widget.h
  src/widgets/default_collisions_widget.h
  src/widgets/robot_poses_widget.h
  src/widgets/end_effectors_widget.h
  src/widgets/virtual_joints_widget.h
  src/widgets/passive_joints_widget.h
  src/widgets/configuration_files_widget.h
  src/widgets/setup_screen_widget.h
)

# Convert the Qt Signals and Slots for QWidget events
qt4_wrap_cpp(moc_sources ${headers})

# Tools Library
add_library(${PROJECT_NAME}_tools
  src/tools/compute_default_collisions.cpp
  src/tools/moveit_config_data.cpp
  src/tools/srdf_writer.cpp
)
target_link_libraries(${PROJECT_NAME}_tools ${YAML} ${catkin_LIBRARIES} ${Boost_LIBRARIES})

# Main Widgets Library - all screens (navigation options)
add_library(${PROJECT_NAME}_widgets
  src/widgets/start_screen_widget.cpp
  src/widgets/planning_groups_widget.cpp
  src/widgets/double_list_widget.cpp
  src/widgets/kinematic_chain_widget.cpp
  src/widgets/group_edit_widget.cpp
  src/widgets/default_collisions_widget.cpp
  src/widgets/robot_poses_widget.cpp
  src/widgets/end_effectors_widget.cpp
  src/widgets/virtual_joints_widget.cpp
  src/widgets/passive_joints_widget.cpp
  src/widgets/configuration_files_widget.cpp
  src/widgets/navigation_widget.cpp
  src/widgets/header_widget.cpp
  src/widgets/setup_assistant_widget.cpp
  src/widgets/setup_screen_widget.cpp
  ${moc_sources}
)
target_link_libraries(${PROJECT_NAME}_widgets ${PROJECT_NAME}_tools ${QT_LIBRARIES} ${catkin_LIBRARIES} ${Boost_LIBRARIES})

# Source files for the exectuable
set( sources
  src/setup_assistant_main.cpp
)

catkin_package(
  INCLUDE_DIRS
    include
  LIBRARIES
    ${PROJECT_NAME}_tools
  CATKIN_DEPENDS
    moveit_ros_planning
    moveit_ros_visualization
    )

# Link libraries
add_executable(${PROJECT_NAME} ${sources} ${headers})
target_link_libraries(${PROJECT_NAME}
  ${PROJECT_NAME}_widgets ${PROJECT_NAME}_tools
  ${QT_LIBRARIES} ${catkin_LIBRARIES} ${Boost_LIBRARIES} log4cxx)

install(TARGETS ${PROJECT_NAME} ${PROJECT_NAME}_widgets ${PROJECT_NAME}_tools
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION})
install(DIRECTORY include/ DESTINATION include)

install(DIRECTORY launch DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION})
install(DIRECTORY resources DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION})
install(DIRECTORY templates DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION})
