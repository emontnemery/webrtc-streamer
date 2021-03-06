cmake_minimum_required (VERSION 3.0.0)
project (webrtc-streamer)

set (WEBRTCROOT "../webrtc/src" CACHE STRING "WEBRTC root directory")
set (WEBRTCBUILD "Release" CACHE STRING "WEBRTC build type")
set (WEBRTCOZONE "No" CACHE STRING "WEBRTC use ozone")

MESSAGE("WEBRTCBUILD = ${WEBRTCBUILD}\nWEBRTCDESKTOP = ${WEBRTCDESKTOP}")

set (CMAKE_BUILD_TYPE ${WEBRTCBUILD}) 
set (CMAKE_POSITION_INDEPENDENT_CODE ON)
set (CMAKE_CXX_STANDARD 11)

# build version identifier
find_package(Git)
if(GIT_FOUND)
    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} describe --tags --always --dirty OUTPUT_VARIABLE PROJECTVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
	set (VERSION "${PROJECTVERSION}")
	
    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} -C civetweb describe --tags --always --dirty OUTPUT_VARIABLE CIVETVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
	set (VERSION "${VERSION} civetweb@${CIVETVERSION}")

    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} -C ${WEBRTCROOT} describe --tags --always --dirty OUTPUT_VARIABLE WEBRTCVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
	set (VERSION "${VERSION} webrtc@${WEBRTCVERSION}")

    EXECUTE_PROCESS(COMMAND ${GIT_EXECUTABLE} -C live555helper describe --tags --always --dirty OUTPUT_VARIABLE LIVEVERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
	set (VERSION "${VERSION} live555helper@${LIVEVERSION}")
endif()
add_definitions(-DVERSION=\"${VERSION}\")

# target
FILE(GLOB SOURCE src/*.cpp)
add_executable (${PROJECT_NAME} ${SOURCE})
target_include_directories(${PROJECT_NAME} PRIVATE inc) 

# thread
find_package (Threads)
target_link_libraries (${PROJECT_NAME} ${CMAKE_THREAD_LIBS_INIT}) 

# civetweb
add_library (civetweb STATIC civetweb/src/civetweb.c civetweb/src/CivetServer.cpp)	
target_link_libraries (${PROJECT_NAME} civetweb) 
target_include_directories(civetweb PUBLIC civetweb/include) 

# webrtc
if (NOT EXISTS ${WEBRTCROOT}/out/${WEBRTCBUILD}/args.gn)
  set (WEBRTCARGS rtc_use_h264=true\nffmpeg_branding="Chrome"\nrtc_include_tests=false\nrtc_enable_protobuf=false\nuse_custom_libcxx=false\nrtc_include_pulse_audio=false\nrtc_build_examples=false\nuse_sysroot=false\nis_clang=false\ntreat_warnings_as_errors=false\n)
  if(WEBRTCBUILD STREQUAL "Release")
  	set (WEBRTCARGS is_debug=false\n${WEBRTCARGS})
  endif()
  if(WEBRTCOZONE STREQUAL "Yes")
  	set (WEBRTCARGS use_ozone=true\n${WEBRTCARGS})
  endif()
  FILE(WRITE ${WEBRTCROOT}/out/${WEBRTCBUILD}/args.gn ${WEBRTCARGS})

  if (WIN32)
  	SET (SHELLCOMMAND cmd /c )
  endif(WIN32)

  EXECUTE_PROCESS(WORKING_DIRECTORY ${WEBRTCROOT}/out/${WEBRTCBUILD} COMMAND ${SHELLCOMMAND} gn gen .)
  EXECUTE_PROCESS(WORKING_DIRECTORY ${WEBRTCROOT}/out/${WEBRTCBUILD} COMMAND ninja webrtc rtc_json jsoncpp builtin_video_decoder_factory builtin_video_encoder_factory create_pc_factory peerconnection)
endif() 

set (WEBRTCINCLUDE ${WEBRTCROOT} ${WEBRTCROOT}/third_party/abseil-cpp ${WEBRTCROOT}/third_party/jsoncpp/source/include ${WEBRTCROOT}/third_party/libyuv/include)
target_include_directories(${PROJECT_NAME} PRIVATE ${WEBRTCINCLUDE})

set (WEBRTCOBJS ${WEBRTCROOT}/out/${WEBRTCBUILD}/obj)

# TODO: find a way to do the same for windows & linux
if (MSVC)
	FILE(GLOB_RECURSE WEBRTBBRCOBJS ${WEBRTCOBJS}/modules/congestion_controller/bbr/*${CMAKE_C_OUTPUT_EXTENSION})
	FILE(GLOB_RECURSE WEBRTJSONCPPCOBJS ${WEBRTCOBJS}/third_party/jsoncpp/jsoncpp/*${CMAKE_C_OUTPUT_EXTENSION})
	SET (WEBRTCEXTRAOBJS ${WEBRTJSONCPPCOBJS} ${WEBRTBBRCOBJS} ${WEBRTCOBJS}/rtc_base/rtc_json/json${CMAKE_C_OUTPUT_EXTENSION})	

	find_library(WEBRTC_LIBRARY NAMES webrtc PATHS ${WEBRTCOBJS})
	target_link_libraries (${PROJECT_NAME} ${WEBRTC_LIBRARY}) 
	find_library(WEBRTC_VIDEO_DECODER_LIBRARY NAMES builtin_video_decoder_factory PATHS ${WEBRTCOBJS}/api/video_codecs/)
	find_library(WEBRTC_VIDEO_ENCODER_LIBRARY NAMES builtin_video_encoder_factory PATHS ${WEBRTCOBJS}/api/video_codecs/)
	find_library(WEBRTC_PC_LIBRARY NAMES create_pc_factory PATHS ${WEBRTCOBJS}/pc)
	find_library(WEBRTC_PEERCONNECTION_LIBRARY NAMES peerconnection PATHS ${WEBRTCOBJS}/pc)
	target_link_libraries (${PROJECT_NAME} ${WEBRTC_VIDEO_DECODER_LIBRARY} ${WEBRTC_VIDEO_ENCODER_LIBRARY} ${WEBRTC_PC_LIBRARY} ${WEBRTC_PEERCONNECTION_LIBRARY}) 
else()
	FILE(GLOB_RECURSE WEBRTCEXTRAOBJS ${WEBRTCOBJS}/*${CMAKE_C_OUTPUT_EXTENSION})
endif()
add_library (webrtcextra STATIC ${WEBRTCEXTRAOBJS})
set_target_properties(webrtcextra PROPERTIES LINKER_LANGUAGE CXX)		
target_link_libraries (${PROJECT_NAME} webrtcextra) 


# live555helper & live555
if (NOT EXISTS live)
	file (DOWNLOAD http://www.live555.com/liveMedia/public/live555-latest.tar.gz ${CMAKE_SOURCE_DIR}/live555-latest.tar.gz )
    EXECUTE_PROCESS(COMMAND 7z x live555-latest.tar.gz -so COMMAND 7z x -aoa -ttar -si)
endif(NOT EXISTS live) 

FILE(GLOB LIVEHELPERSOURCE live/groupsock/*.c* live/liveMedia/*.c* live/UsageEnvironment/*.c* live/BasicUsageEnvironment/*.c* live555helper/src/*.cpp)
add_library (live555helper STATIC ${LIVEHELPERSOURCE})	
target_include_directories(live555helper PUBLIC live/groupsock/include live/liveMedia/include live/UsageEnvironment/include live/BasicUsageEnvironment/include live555helper/inc)
target_link_libraries (${PROJECT_NAME} live555helper) 
add_definitions(-DHAVE_LIVE555 -DHAVE_JPEG)

# compiler specific
if (MSVC)
	# live555
	add_definitions(-D_CRT_SECURE_NO_WARNINGS)
	target_link_libraries (${PROJECT_NAME} ws2_32)
	# getopt
    add_library (getopt getopt/getopt.c)
	target_include_directories(getopt PUBLIC getopt) 
	target_link_libraries (${PROJECT_NAME} getopt)
	# webrtc
	add_definitions(-DWEBRTC_WIN -EHsc -DNOMINMAX -D__PRETTY_FUNCTION__=__FUNCTION__)
	set(CompilerFlags CMAKE_CXX_FLAGS_DEBUG CMAKE_C_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE CMAKE_C_FLAGS_RELEASE)
	foreach(CompilerFlag ${CompilerFlags})
	  string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
	endforeach()
	add_definitions(-DUSE_X11)
	target_link_libraries (${PROJECT_NAME} secur32 dmoguids wmcodecdspuuid strmiids msdmo winmm dxgi d3d11)
else(MSVC)
	# live555
	add_definitions(-DBSD=1 -DSOCKLEN_T=socklen_t -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE=1)
	# webrtc
	add_definitions(-DWEBRTC_POSIX -fno-rtti)
	if(WEBRTCBUILD STREQUAL "Debug")
		add_definitions(-D_GLIBCXX_DEBUG=1)
	endif()
	target_link_libraries (${PROJECT_NAME} dl)
	if (EXISTS ${WEBRTCROOT}/out/${WEBRTCBUILD}/obj/modules/desktop_capture/desktop_capture.ninja)
		add_definitions(-DUSE_X11)
		target_link_libraries (${PROJECT_NAME} X11 Xext Xdamage Xfixes Xcomposite)
	endif()
endif(MSVC)

#cpack
install (TARGETS ${PROJECT_NAME} RUNTIME DESTINATION .)
install (DIRECTORY html DESTINATION .)
SET(CPACK_GENERATOR "TGZ")
SET(CPACK_SYSTEM_NAME ${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}-${CMAKE_BUILD_TYPE})
SET(CPACK_PACKAGE_CONTACT "michel.promonet@free.fr") 
if(PROJECTVERSION)
    SET(CPACK_PACKAGE_VERSION "${PROJECTVERSION}")
endif()
INCLUDE(CPack)
