include_directories(${mosquitto_SOURCE_DIR} ${mosquitto_SOURCE_DIR}/src
		${mosquitto_SOURCE_DIR}/lib ${OPENSSL_INCLUDE_DIR}
		${STDBOOL_H_PATH} ${STDINT_H_PATH})

set (MOSQ_SRCS
	conf.c
	context.c
	database.c
	lib_load.h
	logging.c
	loop.c
	../lib/memory_mosq.c ../lib/memory_mosq.h
	mosquitto.c
	mosquitto_broker.h
	net.c
	../lib/net_mosq.c ../lib/net_mosq.h
	persist.c persist.h
	read_handle.c read_handle_client.c read_handle_server.c
	../lib/read_handle_shared.c ../lib/read_handle.h
	subs.c
	security.c security_default.c
	../lib/send_client_mosq.c ../lib/send_mosq.h
	../lib/send_mosq.c ../lib/send_mosq.h
	send_server.c
	sys_tree.c
	../lib/time_mosq.c
	../lib/tls_mosq.c
	../lib/util_mosq.c ../lib/util_mosq.h
	websockets.c
	../lib/will_mosq.c ../lib/will_mosq.h)

option(INC_BRIDGE_SUPPORT
	"Include bridge support for connecting to other brokers?" ON)
if (${INC_BRIDGE_SUPPORT} STREQUAL ON)
	set (MOSQ_SRCS ${MOSQ_SRCS} bridge.c)
	add_definitions("-DWITH_BRIDGE")
endif (${INC_BRIDGE_SUPPORT} STREQUAL ON)


option(USE_LIBWRAP
	"Include tcp-wrappers support?" OFF)

if (${USE_LIBWRAP} STREQUAL ON)
	set (MOSQ_LIBS ${MOSQ_LIBS} wrap)
	add_definitions("-DWITH_WRAP")
endif (${USE_LIBWRAP} STREQUAL ON)

option(INC_DB_UPGRADE
	"Include database upgrade support? (recommended)" ON)

option(INC_MEMTRACK
	"Include memory tracking support?" ON)
if (${INC_MEMTRACK} STREQUAL ON)
	add_definitions("-DWITH_MEMORY_TRACKING")
endif (${INC_MEMTRACK} STREQUAL ON)

option(WITH_PERSISTENCE
	"Include persistence support?" ON)
if (${WITH_PERSISTENCE} STREQUAL ON)
	add_definitions("-DWITH_PERSISTENCE")
endif (${WITH_PERSISTENCE} STREQUAL ON)

option(WITH_SYS_TREE
	"Include $SYS tree support?" ON)
if (${WITH_SYS_TREE} STREQUAL ON)
	add_definitions("-DWITH_SYS_TREE")
endif (${WITH_SYS_TREE} STREQUAL ON)

option(WITH_WEBSOCKETS
	"Include websockets support?" OFF)
if (${WITH_WEBSOCKETS} STREQUAL ON)
	add_definitions("-DWITH_WEBSOCKETS")
endif (${WITH_WEBSOCKETS} STREQUAL ON)

if (WIN32 OR CYGWIN)
	set (MOSQ_SRCS ${MOSQ_SRCS} service.c)
endif (WIN32 OR CYGWIN)

add_definitions (-DWITH_BROKER)

add_executable(mosquitto ${MOSQ_SRCS})

set (MOSQ_LIBS ${MOSQ_LIBS} ${OPENSSL_LIBRARIES})

# Check for getaddrinfo_a
include(CheckLibraryExists)
check_library_exists(anl getaddrinfo_a  "" HAVE_GETADDRINFO_A)
if (HAVE_GETADDRINFO_A)
	add_definitions(-DHAVE_GETADDRINFO_A)
	set (MOSQ_LIBS ${MOSQ_LIBS} anl)
endif (HAVE_GETADDRINFO_A)



if (UNIX)
	if (APPLE)
		set (MOSQ_LIBS ${MOSQ_LIBS} dl m)
	else (APPLE)
		set (MOSQ_LIBS ${MOSQ_LIBS} dl m)
		find_library(LIBRT rt)
		if (LIBRT)
			set (MOSQ_LIBS ${MOSQ_LIBS} rt)
		endif (LIBRT)
	endif (APPLE)
endif (UNIX)

if (WIN32)
	set (MOSQ_LIBS ${MOSQ_LIBS} ws2_32)
endif (WIN32)

if (${WITH_WEBSOCKETS} STREQUAL ON)
	set (MOSQ_LIBS ${MOSQ_LIBS} websockets)
endif (${WITH_WEBSOCKETS} STREQUAL ON)

# Simple detect libuuid
if(NOT APPLE)
	FIND_PATH(UUID_HEADER uuid/uuid.h)
	if (UUID_HEADER)
                find_library ( LIBUUID NAMES uuid )
                add_library(uuid INTERFACE IMPORTED)
                set_property(TARGET uuid PROPERTY INTERFACE_LINK_LIBRARIES ${LIBUUID})

		add_definitions(-DWITH_UUID)
		set (MOSQ_LIBS ${MOSQ_LIBS} uuid)
	endif (UUID_HEADER)
endif(NOT APPLE)

target_link_libraries(mosquitto ${MOSQ_LIBS})

if (UNIX)
	if (APPLE)
		set_target_properties(mosquitto PROPERTIES LINK_FLAGS "-Wl,-exported_symbols_list -Wl,${mosquitto_SOURCE_DIR}/src/linker-macosx.syms")
	else (APPLE)
		set_target_properties(mosquitto PROPERTIES LINK_FLAGS "-Wl,-dynamic-list=${mosquitto_SOURCE_DIR}/src/linker.syms")
	endif (APPLE)
endif (UNIX)

install(TARGETS mosquitto RUNTIME DESTINATION "${SBINDIR}" LIBRARY DESTINATION "${LIBDIR}")
install(FILES mosquitto_plugin.h DESTINATION "${INCLUDEDIR}")

if (${WITH_TLS} STREQUAL ON)
	add_executable(mosquitto_passwd mosquitto_passwd.c)
	target_link_libraries(mosquitto_passwd "${OPENSSL_LIBRARIES}")
	install(TARGETS mosquitto_passwd RUNTIME DESTINATION "${BINDIR}" LIBRARY DESTINATION "${LIBDIR}")
endif (${WITH_TLS} STREQUAL ON)

if (UNIX)
	install(CODE "EXEC_PROGRAM(/sbin/ldconfig)")
endif (UNIX)

