cmake_minimum_required(VERSION 2.8 FATAL_ERROR)


project(ardupilot_sitl_gazebo)

#######################
## Find Dependencies ##
#######################
set(Boost_INCLUDE_DIRS /usr/include/boost)
set(Boost_LIBRARY_DIRS /usr/include/lib)
include_directories(${Boost_INCLUDE_DIRS})
include_directories(${Boost_LIBRARY_DIRS})

find_package(Boost 1.58 REQUIRED COMPONENTS system thread filesystem)
find_package(gazebo REQUIRED)
find_package(PkgConfig REQUIRED)
find_program(px4 REQUIRED)
find_package(PythonInterp REQUIRED)
find_package(OpenCV REQUIRED)

#fix this: find_package(TinyXML REQUIRED)


pkg_check_modules(OGRE OGRE)

include_directories(SYSTEM ${GAZEBO_INCLUDE_DIRS})
link_directories(${GAZEBO_LIBRARY_DIRS})

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${GAZEBO_CXX_FLAGS}")

if (BUILD_ROS_INTERFACE)
  find_package(geometry_msgs REQUIRED)
  find_package(roscpp REQUIRED)
  find_package(sensor_msgs REQUIRED)
endif()

# find MAVLink
find_package(MAVLink)

set(Boost_DEBUG 1)
if("${GAZEBO_VERSION}" VERSION_LESS "7.0")
    message(FATAL_ERROR "You need at least Gazebo 7.0. Your version: ${GAZEBO_VERSION}")
else()
    message("Gazebo version: ${GAZEBO_VERSION}")
endif()

# XXX this approach is extremely error prone
# it would be preferable to either depend on the
# compiled headers from Gazebo directly
# or to have something entirely independent.
#
set(PROTOBUF_IMPORT_DIRS "")
foreach(ITR ${GAZEBO_INCLUDE_DIRS})
  if(ITR MATCHES ".*gazebo-[0-9.]+$")
    set(PROTOBUF_IMPORT_DIRS "${ITR}/gazebo/msgs/proto")
  endif()
endforeach()

# PROTOBUF_IMPORT_DIRS has to be set before
# find_package is called
find_package(Protobuf REQUIRED)
pkg_check_modules(PROTOBUF protobuf)

if ("${PROTOBUF_VERSION}" VERSION_LESS "2.5.0")
  message(FATAL_ERROR "protobuf version: ${PROTOBUF_VERSION} not compatible, must be >= 2.5.0")
endif()

find_package(Eigen3 QUIET)
if(NOT EIGEN3_FOUND)
  # Fallback to cmake_modules
  find_package(Eigen QUIET)
  if(NOT EIGEN_FOUND)
    pkg_check_modules(EIGEN3 REQUIRED eigen3)
  else()
    set(EIGEN3_INCLUDE_DIRS ${EIGEN_INCLUDE_DIRS})
    set(EIGEN3_LIBRARIES ${EIGEN_LIBRARIES})
  endif()
else()
  set(EIGEN3_INCLUDE_DIRS ${EIGEN3_INCLUDE_DIR})
endif()

###########
## Build ##
###########

set(GAZEBO_MSG_INCLUDE_DIRS)
foreach(ITR ${GAZEBO_INCLUDE_DIRS})
  if(ITR MATCHES ".*gazebo-[0-9.]+$")
    set(GAZEBO_MSG_INCLUDE_DIRS "${ITR}/gazebo/msgs")
  endif()
endforeach()

include_directories(
        ${PROJECT_SOURCE_DIR}
        include  
	${Boost_INCLUDE_DIR}
  	${CMAKE_CURRENT_BINARY_DIR}
  	${EIGEN3_INCLUDE_DIRS}
  	${EIGEN3_INCLUDE_DIRS}/eigen3	# Workaround for Eigen3
  	${GAZEBO_INCLUDE_DIRS}
  	${GAZEBO_MSG_INCLUDE_DIRS}
  	${MAVLINK_INCLUDE_DIRS}
  	${OGRE_INCLUDE_DIRS}
  	${OGRE_INCLUDE_DIRS}/Paging		# Workaround for "fatal error: OgrePagedWorldSection.h: No such file or directory"
  	${OpenCV_INCLUDE_DIRS}
  	${OpticalFlow_INCLUDE_DIRS}
  	#${TinyXML_INCLUDE_DIRS}
        )

link_libraries(
        ${CMAKE_CURRENT_BINARY_DIR}
	${Boost_SYSTEM_LIBRARY_RELEASE}
	${Boost_THREAD_LIBRARY_RELEASE}
	${Boost_TIMER_LIBRARY_RELEASE}
        ${GAZEBO_LIBRARIES}
	${OpenCV_LIBRARIES}
	${PROTOBUF_LIBRARY}
        )

link_directories(
  ${GAZEBO_LIBRARY_DIRS}
  ${CMAKE_CURRENT_BINARY_DIR}
  ${OGRE_LIBRARY_DIRS}
  )
  
#--------------------------#
# Generation of SDF models #
#--------------------------#

set(enable_mavlink_interface "true")
set(enable_ground_truth "false")
set(enable_logging "false")
set(enable_camera "false")
set(enable_wind "false")
set(rotors_description_dir "${CMAKE_CURRENT_SOURCE_DIR}/models/rotors_description")
set(scripts_dir "${CMAKE_CURRENT_SOURCE_DIR}/scripts")
# set the vision estimation to be sent if set by the CMake option SEND_VISION_ESTIMATION_DATA
set(send_vision_estimation "false")
if (SEND_VISION_ESTIMATION_DATA)
  set(send_vision_estimation "true")
endif()

# if SEND_ODOMETRY_DATA option is set, then full odometry data is sent instead of
# only the visual pose estimate
set(send_odometry "false")
if (SEND_ODOMETRY_DATA)
  set(send_odometry "true")
  set(send_vision_estimation "false")
endif()

#-----------#
# Functions #
#-----------#

function(glob_generate target file_glob)
  file(GLOB_RECURSE glob_files RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} ${file_glob})
	set(gen_files)
	foreach(glob_file ${glob_files})
		string(REGEX REPLACE "\\.[^.]*$" "" file_name ${glob_file})
		string(REGEX MATCH "[^.]*$" file_ext ${glob_file})
		get_filename_component(file_dir ${glob_file} DIRECTORY)
		set(in_file ${CMAKE_CURRENT_SOURCE_DIR}/${glob_file})
		file(MAKE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/${file_dir})
		set(out_file ${CMAKE_CURRENT_SOURCE_DIR}/${file_name})
		string(REGEX REPLACE ".sdf" "-gen.sdf" out_file ${out_file})
		if (${file_ext} STREQUAL "jinja")
			add_custom_command(OUTPUT ${out_file}
        COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/scripts/jinja_gen.py
        ${in_file} ${CMAKE_CURRENT_SOURCE_DIR}
				DEPENDS ${in_file}
				VERBATIM
				)
			list(APPEND gen_files_${target} ${out_file})
		endif()
	endforeach()
	add_custom_target(${target} ALL DEPENDS ${gen_files_${target}})
endfunction()

glob_generate(models_gen ${CMAKE_CURRENT_SOURCE_DIR}/models/*.jinja)

#--------------------#
# Message Generation #
#--------------------#

set(mav_msgs
  msgs/CommandMotorSpeed.proto
  msgs/MotorSpeed.proto
  )
set(nav_msgs msgs/Odometry.proto)
set(physics_msgs msgs/Wind.proto)
set(std_msgs msgs/Int32.proto)
set(sensor_msgs
  msgs/Imu.proto
  msgs/IRLock.proto
  msgs/Float.proto
  msgs/Groundtruth.proto
  msgs/Range.proto
  msgs/SITLGps.proto
  msgs/OpticalFlow.proto
  msgs/MagneticField.proto
  msgs/Pressure.proto
  )

PROTOBUF_GENERATE_CPP(MAV_PROTO_SRCS MAV_PROTO_HDRS ${mav_msgs})
PROTOBUF_GENERATE_CPP(NAV_PROTO_SRCS NAV_PROTO_HDRS ${nav_msgs})
PROTOBUF_GENERATE_CPP(PHY_PROTO_SRCS PHY_PROTO_HDRS ${physics_msgs})
PROTOBUF_GENERATE_CPP(STD_PROTO_SRCS STD_PROTO_HDRS ${std_msgs})
PROTOBUF_GENERATE_CPP(SEN_PROTO_SRCS SEN_PROTO_HDRS ${sensor_msgs})

add_library(mav_msgs SHARED ${MAV_PROTO_SRCS})
add_library(nav_msgs SHARED ${NAV_PROTO_SRCS})
add_library(physics_msgs SHARED ${PHY_PROTO_SRCS})
add_library(std_msgs SHARED ${STD_PROTO_SRCS})
add_library(sensor_msgs SHARED ${SEN_PROTO_SRCS})
	
#---------#
# Plugins #
#---------#
link_libraries(mav_msgs nav_msgs std_msgs sensor_msgs)
link_libraries(physics_msgs)

set (plugins
        ArduPilotPlugin
        ArduCopterIRLockPlugin
	gazebo_geotagged_images_plugin
	gazebo_gps_plugin
	gazebo_lidar_plugin
	gazebo_gimbal_controller_plugin
	gazebo_imu_plugin
	gazebo_mavlink_interface
	gazebo_multirotor_base_plugin
	gazebo_magnetometer_plugin
	gazebo_barometer_plugin
	LiftDragPlugin
				# ArduCopterPlugin
        # GimbalSmall2dPlugin
	# WindPlugin
        )

add_library(ArduCopterIRLockPlugin SHARED src/ArduCopterIRLockPlugin.cc)
target_link_libraries(ArduCopterIRLockPlugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(ArduPilotPlugin SHARED src/ArduPilotPlugin.cc)
target_link_libraries(ArduPilotPlugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(gazebo_geotagged_images_plugin SHARED src/gazebo_geotagged_images_plugin.cpp)
target_link_libraries(gazebo_geotagged_images_plugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(gazebo_gps_plugin SHARED src/gazebo_gps_plugin.cpp)
target_link_libraries(gazebo_gps_plugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(gazebo_lidar_plugin SHARED src/gazebo_lidar_plugin.cpp)
target_link_libraries(gazebo_lidar_plugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(gazebo_gimbal_controller_plugin SHARED src/gazebo_gimbal_controller_plugin.cpp)
target_link_libraries(gazebo_gimbal_controller_plugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(gazebo_imu_plugin SHARED src/gazebo_imu_plugin.cpp)
target_link_libraries(gazebo_imu_plugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(gazebo_mavlink_interface SHARED src/gazebo_mavlink_interface.cpp)
target_link_libraries(gazebo_mavlink_interface ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(gazebo_multirotor_base_plugin SHARED src/gazebo_multirotor_base_plugin.cpp)
target_link_libraries(gazebo_multirotor_base_plugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(gazebo_magnetometer_plugin SHARED src/gazebo_magnetometer_plugin.cpp src/geo_mag_declination.cpp)
target_link_libraries(gazebo_magnetometer_plugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

add_library(gazebo_barometer_plugin SHARED src/gazebo_barometer_plugin.cpp)
target_link_libraries(gazebo_barometer_plugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})


add_library(LiftDragPlugin SHARED src/LiftDragPlugin.cc)
target_link_libraries(LiftDragPlugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

# add_library(ArduCopterPlugin SHARED src/ArduCopterPlugin.cc)
# target_link_libraries(ArduCopterPlugin ${Boost_LIBRARIES} ${GAZEBO_LIBRARIES} ${TinyXML_LIBRARIES})

if("${GAZEBO_VERSION}" VERSION_LESS "8.0")
    add_library(GimbalSmall2dPlugin SHARED src/GimbalSmall2dPlugin.cc)
    target_link_libraries(GimbalSmall2dPlugin ${GAZEBO_LIBRARIES})
    install(TARGETS GimbalSmall2dPlugin DESTINATION ${GAZEBO_PLUGIN_PATH})
endif()

foreach(plugin ${plugins})
  add_dependencies(${plugin} mav_msgs nav_msgs std_msgs sensor_msgs)
  add_dependencies(${plugin} physics_msgs)
endforeach()

# Configure the setup script
if (catkin_FOUND)
  catkin_add_env_hooks(50_sitl_gazebo_setup
    DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/cmake
    SHELLS sh)
endif()

#############
## Install ##
#############

set(PLUGIN_PATH ${CMAKE_INSTALL_LIBDIR}/${PROJECT_NAME}/plugins)
set(MODEL_PATH ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/models)
set(RESOURCE_PATH ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME})

file(REMOVE_RECURSE ${PROJECT_SOURCE_DIR}/models/.DS_Store)
file(GLOB models_list LIST_DIRECTORIES true ${PROJECT_SOURCE_DIR}/models/*)

file(REMOVE_RECURSE ${PROJECT_SOURCE_DIR}/worlds/.DS_Store)
file(GLOB worlds_list LIST_DIRECTORIES true ${PROJECT_SOURCE_DIR}/worlds/*)

install(TARGETS ${plugins} mav_msgs nav_msgs std_msgs sensor_msgs DESTINATION ${PLUGIN_PATH})
install(DIRECTORY ${models_list} DESTINATION ${MODEL_PATH})
install(FILES ${worlds_list} DESTINATION ${RESOURCE_PATH}/worlds)

configure_file(src/setup.sh.in "${CMAKE_CURRENT_BINARY_DIR}/setup.sh" @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/setup.sh DESTINATION ${RESOURCE_PATH})

install(FILES ${PROJECT_SOURCE_DIR}/package.xml DESTINATION ${RESOURCE_PATH})

#############
## Testing ##
#############

# TODO

###############
## Packaging ##
###############

set(CPACK_PACKAGE_NAME ${PROJECT_NAME}-${GAZEBO_MAJOR_VERSION})
set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
set(CPACK_PACKAGE_CONTACT pxusers@googlegroups.com)
set(DEBIAN_PACKAGE_DEPENDS "")
set(RPM_PACKAGE_DEPENDS "")

set(CPACK_DEBIAN_PACKAGE_DEPENDS ${DEBIAN_PACKAGE_DEPENDS})
set(CPACK_DEBIAN_PACKAGE_SECTION "devel")
set(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
set(CPACK_DEBIAN_PACKAGE_DESCRIPTION "gazebo plugins for ardupilot sitl.")

set(CPACK_RPM_PACKAGE_REQUIRES "${DEBIAN_PACKAGE_DEPENDS}")
set(CPACK_RPM_PACKAGE_DESCRIPTION "Gazebo plugins for ardupilot sitl.")
set(CPACK_PACKAGE_FILE_NAME "${PROJECT_NAME}-${GAZEBO_MAJOR_VERSION}-${PROJECT_VERSION}")
set(CPACK_SOURCE_PACKAGE_FILE_NAME "${PROJECT_NAME}-${GAZEBO_MAJOR_VERSION}-${PROJECT_VERSION}")

include(CPack)
