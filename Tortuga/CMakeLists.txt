cmake_minimum_required(VERSION 3.22)

# Setup compiler settings
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)

# Define the build type
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Debug")
endif()

# Set the project name
set(CMAKE_PROJECT_NAME Tortuga)

# Enable compile command to ease indexing with e.g. clangd
set(CMAKE_EXPORT_COMPILE_COMMANDS TRUE)

# Core project settings
project(${CMAKE_PROJECT_NAME})
message("Build type: " ${CMAKE_BUILD_TYPE})

# Enable CMake support for ASM and C languages
enable_language(C ASM)

# Create an executable object type
add_executable(${CMAKE_PROJECT_NAME})

# Add STM32CubeMX generated sources
add_subdirectory(cmake/stm32cubemx)

# Link directories setup
target_link_directories(${CMAKE_PROJECT_NAME} PRIVATE
    # Add user defined library search paths
)

# Add sources to executable
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    # Add user sources here
)

# Add include paths
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    # Add user defined include paths
)

# Add project symbols (macros)
target_compile_definitions(${CMAKE_PROJECT_NAME} PRIVATE
    # Add user defined symbols
)

# Remove wrong libob.a library dependency when using cpp files
list(REMOVE_ITEM CMAKE_C_IMPLICIT_LINK_LIBRARIES ob)

# Add linked libraries
target_link_libraries(${CMAKE_PROJECT_NAME}
    stm32cubemx

    # Add user defined libraries
)


# --- BLOQUE LIBRERIAS AUTOMÁTICAS ---
# Si no funciona: CMake: Delete Cache and Reconfigure

set(RUTAS_DE_BUSQUEDA
    "${CMAKE_SOURCE_DIR}/Core"
    "${CMAKE_SOURCE_DIR}/Core/Src"
    "${CMAKE_SOURCE_DIR}"
)

set(LIBRERIAS_DIR "")

foreach(ruta ${RUTAS_DE_BUSQUEDA})
    if(EXISTS "${ruta}" AND NOT LIBRERIAS_DIR)
        file(GLOB contenido RELATIVE "${ruta}" "${ruta}/*")
        foreach(item ${contenido})
            if(IS_DIRECTORY "${ruta}/${item}")
                string(TOLOWER "${item}" item_lower)
                if(item_lower STREQUAL "librerias")
                    set(LIBRERIAS_DIR "${ruta}/${item}")
                    break()
                endif()
            endif()
        endforeach()
    endif()
endforeach()

# Solo se incluye si la carpeta existe (si no, se omite sin aviso)
if(LIBRERIAS_DIR)
    message(STATUS "Carpeta librerias: ${LIBRERIAS_DIR}")

    file(GLOB_RECURSE LIBRERIAS_SOURCES CONFIGURE_DEPENDS "${LIBRERIAS_DIR}/*.c")
    target_sources(${CMAKE_PROJECT_NAME} PRIVATE ${LIBRERIAS_SOURCES})

    file(GLOB_RECURSE LIBRERIAS_HEADERS CONFIGURE_DEPENDS "${LIBRERIAS_DIR}/*.h")
    set(LIBRERIAS_INCLUDE_DIRS "")
    foreach(header ${LIBRERIAS_HEADERS})
        get_filename_component(header_dir "${header}" DIRECTORY)
        list(APPEND LIBRERIAS_INCLUDE_DIRS "${header_dir}")
    endforeach()

    if(LIBRERIAS_INCLUDE_DIRS)
        list(REMOVE_DUPLICATES LIBRERIAS_INCLUDE_DIRS)
        target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ${LIBRERIAS_INCLUDE_DIRS})
    endif()
else()
    message(STATUS "No hay carpeta 'librerias': se omite (no es un error)")
endif()
#fin librerias


# --- BLOQUE SOPORTE FLOAT Y MATH (-lm) ---
target_link_options(${CMAKE_PROJECT_NAME} PRIVATE
    -u _printf_float
    -lm
)


# --- BLOQUE GENERACIÓN HEX Y BIN ---
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.bin
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${CMAKE_PROJECT_NAME}>
    COMMENT "Generando ${CMAKE_PROJECT_NAME}.hex y ${CMAKE_PROJECT_NAME}.bin"
)


# --- BLOQUE OPTIMIZACIÓN ---
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE -Og -g3)
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
    target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE -O2 -g0)
endif()