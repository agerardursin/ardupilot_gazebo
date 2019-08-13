cmake_minimum_required(VERSION 2.8 FATAL_ERROR)


project(ardupilot_sitl_gazebo)

#######################
## Find Dependencies ##
#######################
set(Boost_INCLUDE_DIRS /usr/include/boost)
set(Boost_LIBRARY_DIRS /usr/include/lib)
include_directories(${Boost_INCLUDE_DIRS})
include_directories(${Boost_LIBRARY_DIRS})

find_package(gazebo REQUIRED)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${GAZEBO_CXX_FLAGS}")



set(Boost_DEBUG 1)
if("${GAZEBO_VERSION}" VERSION_LESS "7.0")
    message(FATAL_ERROR "You need at least Gazebo 7.0. Your version: ${GAZEBO_VERSION}")
else()
    message("Gazebo version: ${GAZEBO_VERSION}")
endif()

###########
## Build ##
###########

include_directories(
        ${PROJECT_SOURCE_DIR}
        include
        ${GAZEBO_INCLUDE_DIRS}
        )

link_libraries(
        ${CMAKE_CURRENT_BINARY_DIR}
        ${GAZEBO_LIBRARIES}
        )

link_directories(
        ${GAZEBO_LIBRARY_DIRS}
        )

set (plugins_single_header
        ArduPilotPlugin
        ArduCopterIRLockPlugin
	#gazebo_geotagged_images_plugin
	#gazebo_gps_plugin
	#gazebo_lidar_plugin
	#gazebo_gimbal_controller_plugin
	#gazebo_imu_plugin
	#gazebo_mavlink_interface
	#gazebo_multirotor_base_plugin
	#gazebo_magnetometer_plugin
	#gazebo_barometer_plugin
				# ArduCopterPlugin
        # GimbalSmall2dPlugin
	# WindPlugin
        )

add_library(ArduCopterIRLockPlugin SHARED src/ArduCopterIRLockPlugin.cc)
target_link_libraries(ArduCopterIRLockPlugin ${GAZEBO_LIBRARIES})

add_library(ArduPilotPlugin SHARED src/ArduPilotPlugin.cc)
target_link_libraries(ArduPilotPlugin ${GAZEBO_LIBRARIES})
#[[
#add_library(gazebo_geotagged_images_plugin SHARED src/gazebo_geotagged_images_plugin.cpp)
#target_link_libraries(gazebo_geotagged_images_plugin ${GAZEBO_LIBRARIES})

#add_library(gazebo_gps_plugin SHARED src/gazebo_gps_plugin.cpp)
#target_link_libraries(gazebo_gps_plugin ${GAZEBO_LIBRARIES})

#add_library(gazebo_lidar_plugin SHARED src/gazebo_lidar_plugin.cpp)
#target_link_libraries(gazebo_lidar_plugin ${GAZEBO_LIBRARIES})

add_library(gazebo_gimbal_controller_plugin SHARED src/gazebo_gimbal_controller_plugin.cpp)
target_link_libraries(gazebo_gimbal_controller_plugin ${GAZEBO_LIBRARIES})

add_library(gazebo_imu_plugin SHARED src/gazebo_imu_plugin.cpp)
target_link_libraries(gazebo_imu_plugin ${GAZEBO_LIBRARIES})

add_library(gazebo_mavlink_interface SHARED src/gazebo_mavlink_interface.cpp)
target_link_libraries(gazebo_mavlink_interface ${GAZEBO_LIBRARIES})

add_library(gazebo_multirotor_base_plugin SHARED src/gazebo_multirotor_base_plugin.cpp)
target_link_libraries(gazebo_multirotor_base_plugin ${GAZEBO_LIBRARIES})

add_library(gazebo_magnetometer_plugin SHARED src/gazebo_magnetometer_plugin.cpp src/geo_mag_declination.cpp)
target_link_libraries(gazebo_magnetometer_plugin ${GAZEBO_LIBRARIES})

add_library(gazebo_barometer_plugin SHARED src/gazebo_barometer_plugin.cpp)
target_link_libraries(gazebo_barometer_plugin ${GAZEBO_LIBRARIES})

# add_library(ArduCopterPlugin SHARED src/ArduCopterPlugin.cc)
# target_link_libraries(ArduCopterPlugin ${GAZEBO_LIBRARIES})
]]
if("${GAZEBO_VERSION}" VERSION_LESS "8.0")
    add_library(GimbalSmall2dPlugin SHARED src/GimbalSmall2dPlugin.cc)
    target_link_libraries(GimbalSmall2dPlugin ${GAZEBO_LIBRARIES})
    install(TARGETS GimbalSmall2dPlugin DESTINATION ${GAZEBO_PLUGIN_PATH})
endif()

install(TARGETS ArduCopterIRLockPlugin DESTINATION ${GAZEBO_PLUGIN_PATH})
install(TARGETS ArduPilotPlugin DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS gazebo_geotagged_images_plugin DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS gazebo_gps_plugin DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS gazebo_lidar_plugin DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS gazebo_gimbal_controller_plugin DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS gazebo_imu_plugin DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS gazebo_mavlink_interface DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS gazebo_multirotor_base_plugin DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS gazebo_magnetometer_plugin DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS gazebo_barometer_plugin DESTINATION ${GAZEBO_PLUGIN_PATH})
#install(TARGETS ArduCopterPlugin DESTINATION ${GAZEBO_PLUGIN_PATH})
