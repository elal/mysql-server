# Copyright (c) 2017, 2019, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License, version 2.0,
# as published by the Free Software Foundation.
#
# This program is also distributed with certain software (including
# but not limited to OpenSSL) that is licensed under separate terms,
# as designated in a particular file or component or in included license
# documentation.  The authors of MySQL hereby grant you an additional
# permission to link the program and your derivative works with the
# separately licensed software that they have included with MySQL.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License, version 2.0, for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA 02110-1301  USA

SET(PROTOCOL_FULL_LIB_NAME mysqlxmessages)
SET(PROTOCOL_LITE_LIB_NAME mysqlxmessages_lite)

SET(PROTOCOL_FULL_DIR ${MYSQLX_GENERATE_DIR}/protobuf)
SET(PROTOCOL_LITE_DIR ${MYSQLX_GENERATE_DIR}/protobuf_lite)
FILE(MAKE_DIRECTORY ${PROTOCOL_FULL_DIR} ${PROTOCOL_LITE_DIR})

INCLUDE(source_files.cmake)

SET(GENERATED_LITE_PROTO_FILES)
# Generate protobuf-lite versions of proto files.
FOREACH(PROTOBUF_MYSQLX_FILE ${MYSQLX_PROTOBUF_PROTO_FILES})
  GET_FILENAME_COMPONENT(MYSQLX_NAME_WD ${PROTOBUF_MYSQLX_FILE} NAME)
  LIST(APPEND GENERATED_LITE_PROTO_FILES
    ${PROTOCOL_LITE_DIR}/${MYSQLX_NAME_WD})
  ADD_CUSTOM_COMMAND(
    OUTPUT "${PROTOCOL_LITE_DIR}/${MYSQLX_NAME_WD}"
    COMMAND ${CMAKE_COMMAND}
            -D TARGET_DIR="${PROTOCOL_LITE_DIR}"
            -D PROTO_FILE="${PROTOBUF_MYSQLX_FILE}"
            -D PROTO_FILE_FLAGS="PROTOBUF_LITE"
            -P ${CMAKE_CURRENT_SOURCE_DIR}/transform_to_lite.cmake
    COMMENT "transform_to_lite ${PROTOBUF_MYSQLX_FILE}"
    DEPENDS ${PROTOBUF_MYSQLX_FILE})
  SET_SOURCE_FILES_PROPERTIES(
    "${PROTOCOL_LITE_DIR}/${MYSQLX_NAME_WD}"
    PROPERTIES GENERATED TRUE)
ENDFOREACH()

# Be sure to convert all proto files to protobuf_lite before running protoc
ADD_CUSTOM_TARGET(GenLiteProtos ALL DEPENDS ${GENERATED_LITE_PROTO_FILES})

INCLUDE(protobuf_compile.cmake)

MYSQLX_PROTOBUF_GENERATE_CPP(
  PROTO_SRCS PROTO_HDRS
  PROTO_LITE_SRCS PROTO_LITE_HDRS
  ${MYSQLX_PROTOBUF_PROTO_FILES}
)

MY_INCLUDE_SYSTEM_DIRECTORIES(PROTOBUF)
ADD_LIBRARY(${PROTOCOL_FULL_LIB_NAME} STATIC
  ${PROTO_SRCS}
)
# Generate it only if needed by other targets
SET_PROPERTY(TARGET ${PROTOCOL_FULL_LIB_NAME} PROPERTY EXCLUDE_FROM_ALL TRUE)

ADD_LIBRARY(${PROTOCOL_LITE_LIB_NAME} STATIC
  ${PROTO_LITE_SRCS}
)
# Generate it only if needed by other targets
SET_PROPERTY(TARGET ${PROTOCOL_LITE_LIB_NAME} PROPERTY EXCLUDE_FROM_ALL TRUE)

SET(XPROTOCOL_FLAGS "")
SET(PUBLIC_XPROTOCOL_FLAGS "")

IF(MY_COMPILER_IS_GNU_OR_CLANG)
  STRING_APPEND(PUBLIC_XPROTOCOL_FLAGS " -Wno-unused-parameter -Wno-undef"
    )

  STRING_APPEND(XPROTOCOL_FLAGS
    " -Wno-ignored-qualifiers -Wno-sign-compare -Wno-unused-variable -Wno-undef"
  )

  MY_CHECK_CXX_COMPILER_WARNING("-Wunused-but-set-parameter" HAS_WARN_FLAG)
  IF(HAS_WARN_FLAG)
    STRING_APPEND(PUBLIC_XPROTOCOL_FLAGS " ${HAS_WARN_FLAG}")
  ENDIF()

  MY_CHECK_CXX_COMPILER_WARNING("-Wextra-semi" HAS_WARN_FLAG)
  IF(HAS_WARN_FLAG)
    STRING_APPEND(PUBLIC_XPROTOCOL_FLAGS " ${HAS_WARN_FLAG}")
  ENDIF()
ENDIF()

IF(MSVC)
  IF(WIN32_CLANG)
    SET(XPROTOCOL_FLAGS "${XPROTOCOL_FLAGS} -Wno-sign-compare")
    SET(PUBLIC_XPROTOCOL_FLAGS "${PUBLIC_XPROTOCOL_FLAGS} -Wno-sign-compare")
  ELSE()
    SET(XPROTOCOL_FLAGS "${XPROTOCOL_FLAGS} /wd4018")
  ENDIF()
ENDIF(MSVC)

ADD_COMPILE_FLAGS(${PROTO_SRCS} ${PROTO_LITE_SRCS} COMPILE_FLAGS
        "${XPROTOCOL_FLAGS} ${PUBLIC_XPROTOCOL_FLAGS}"
)

SET(MYSQLX_PROTOCOL_FLAGS "${PUBLIC_XPROTOCOL_FLAGS}" PARENT_SCOPE)
SET(MYSQLX_PROTOCOL_FULL_LIB ${PROTOCOL_FULL_LIB_NAME} PARENT_SCOPE)
SET(MYSQLX_PROTOCOL_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR} PARENT_SCOPE)
SET(MYSQLX_PROTOCOL_LITE_LIB ${PROTOCOL_LITE_LIB_NAME} PARENT_SCOPE)
SET(MYSQLX_PROTOBUF_PROTO_DIR ${MYSQLX_PROTOBUF_PROTO_DIR} PARENT_SCOPE)
SET(MYSQLX_PROTOBUF_PROTO_FILES ${MYSQLX_PROTOBUF_PROTO_FILES} PARENT_SCOPE)

