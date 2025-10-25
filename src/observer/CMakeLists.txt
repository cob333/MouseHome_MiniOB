MESSAGE(STATUS "This is CMAKE_CURRENT_SOURCE_DIR dir " ${CMAKE_CURRENT_SOURCE_DIR})

INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})

find_package(FLEX REQUIRED)
find_package(BISON REQUIRED)

set(PARSER_WORKDIR ${CMAKE_CURRENT_SOURCE_DIR}/sql/parser)
set(LEXER_SOURCES ${PARSER_WORKDIR}/lex_sql.cpp)
set(LEXER_HEADERS ${PARSER_WORKDIR}/lex_sql.h)
set(PARSER_SOURCES ${PARSER_WORKDIR}/yacc_sql.cpp)
set(PARSER_HEADERS ${PARSER_WORKDIR}/yacc_sql.hpp)

flex_target(lex ${PARSER_WORKDIR}/lex_sql.l "${LEXER_SOURCES}" DEFINES_FILE "${LEXER_HEADERS}")
bison_target(yacc ${PARSER_WORKDIR}/yacc_sql.y "${PARSER_SOURCES}" DEFINES_FILE "${PARSER_HEADERS}")
add_flex_bison_dependency(lex yacc)

FILE(GLOB_RECURSE ALL_SRC *.cpp *.c)
SET(MAIN_SRC ${CMAKE_CURRENT_SOURCE_DIR}/main.cpp)
MESSAGE("MAIN SRC: " ${MAIN_SRC})
SET(LIB_SRC ${LIB_SRC} ${LEXER_SOURCES} ${PARSER_SOURCES})
FOREACH (F ${ALL_SRC})

    IF (NOT ${F} STREQUAL ${MAIN_SRC})
        SET(LIB_SRC ${LIB_SRC} ${F})
        MESSAGE("Use " ${F})
    ENDIF()

ENDFOREACH (F)

SET(LIBEVENT_STATIC_LINK TRUE)
#FIND_PACKAGE(Libevent CONFIG REQUIRED)
# 手动查找 libevent 库
find_library(Libevent_CORE_LIB 
    NAMES event_core libevent_core
    PATHS "/usr/lib/x86_64-linux-gnu"
    NO_DEFAULT_PATH
)

find_library(Libevent_PTHREADS_LIB 
    NAMES event_pthreads libevent_pthreads
    PATHS "/usr/lib/x86_64-linux-gnu" 
    NO_DEFAULT_PATH
)

if(Libevent_CORE_LIB AND Libevent_PTHREADS_LIB)
    message(STATUS "Found libevent core: ${Libevent_CORE_LIB}")
    message(STATUS "Found libevent pthreads: ${Libevent_PTHREADS_LIB}")
    
    # 创建导入目标
    add_library(libevent::core SHARED IMPORTED)
    set_target_properties(libevent::core PROPERTIES
        IMPORTED_LOCATION "${Libevent_CORE_LIB}"
        INTERFACE_INCLUDE_DIRECTORIES "/usr/include"
    )
    
    add_library(libevent::pthreads SHARED IMPORTED)
    set_target_properties(libevent::pthreads PROPERTIES
        IMPORTED_LOCATION "${Libevent_PTHREADS_LIB}"
        INTERFACE_INCLUDE_DIRECTORIES "/usr/include"
    )
    
    set(Libevent_FOUND TRUE)
else()
    message(FATAL_ERROR "libevent not found")
endif()

FIND_PACKAGE(jsoncpp CONFIG REQUIRED)

SET(LIBRARIES common pthread dl libevent::core libevent::pthreads)

# 指定目标文件位置
SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)
MESSAGE("Binary directory:" ${EXECUTABLE_OUTPUT_PATH})
SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
MESSAGE("Archive directory:" ${LIBRARY_OUTPUT_PATH})

ADD_EXECUTABLE(observer ${MAIN_SRC})
TARGET_LINK_LIBRARIES(observer observer_static)

ADD_LIBRARY(observer_static STATIC ${LIB_SRC})

SET_TARGET_PROPERTIES(observer_static PROPERTIES OUTPUT_NAME observer)
TARGET_LINK_LIBRARIES(observer_static ${LIBRARIES} oblsm)

# Target 必须在定义 ADD_EXECUTABLE 之后， programs 不受这个限制
# TARGETS和PROGRAMS 的默认权限是OWNER_EXECUTE, GROUP_EXECUTE, 和WORLD_EXECUTE，即755权限， programs 都是处理脚本类
# 类型分为RUNTIME／LIBRARY／ARCHIVE, prog
INSTALL(TARGETS observer observer_static 
    RUNTIME DESTINATION bin
    ARCHIVE DESTINATION lib)
