cmake_minimum_required(VERSION 3.10)
project(filament-samples C ASM)

if (FILAMENT_SKIP_SAMPLES)
    return()
endif()

set(ROOT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/..)
set(GENERATION_ROOT ${CMAKE_CURRENT_BINARY_DIR})
set(RESOURCE_DIR  "${GENERATION_ROOT}/generated/resources")
set(MATERIAL_DIR  "${GENERATION_ROOT}/generated/material")
set(TEXTURE_DIR "${GENERATION_ROOT}/generated/texture")
set(RESOURCE_BINS)

# ==================================================================================================
# Build materials
# ==================================================================================================

set(MATERIAL_SRCS
        materials/aiDefaultMat.mat
        materials/bakedColor.mat
        materials/bakedTexture.mat
        materials/pointSprites.mat
        materials/aoPreview.mat
        materials/groundShadow.mat
        materials/heightfield.mat
        materials/mirror.mat
        materials/sandboxCloth.mat
        materials/sandboxLit.mat
        materials/sandboxLitFade.mat
        materials/sandboxLitTransparent.mat
        materials/sandboxLitThinRefraction.mat
        materials/sandboxLitThinRefractionSsr.mat
        materials/sandboxLitSolidRefraction.mat
        materials/sandboxLitSolidRefractionSsr.mat
        materials/sandboxSpecGloss.mat
        materials/sandboxSubsurface.mat
        materials/sandboxUnlit.mat
        materials/texturedLit.mat)

if (CMAKE_CROSSCOMPILING)
    include(${IMPORT_EXECUTABLES})
endif()

file(MAKE_DIRECTORY ${MATERIAL_DIR})

foreach (mat_src ${MATERIAL_SRCS})
    get_filename_component(localname "${mat_src}" NAME_WE)
    get_filename_component(fullname "${mat_src}" ABSOLUTE)
    set(output_path "${MATERIAL_DIR}/${localname}.filamat")
    add_custom_command(
            OUTPUT ${output_path}
            COMMAND matc ${MATC_BASE_FLAGS} -o ${output_path} ${fullname}
            MAIN_DEPENDENCY ${mat_src}
            DEPENDS matc
            COMMENT "Compiling material ${mat_src} to ${output_path}"
    )
    list(APPEND RESOURCE_BINS ${output_path})
endforeach()

# ==================================================================================================
# Build sample resources
# ==================================================================================================

file(MAKE_DIRECTORY ${RESOURCE_DIR})

get_resgen_vars(${RESOURCE_DIR} resources)

add_custom_command(
        OUTPUT ${RESGEN_OUTPUTS}
        COMMAND resgen ${RESGEN_FLAGS} ${RESOURCE_BINS}
        DEPENDS resgen ${RESOURCE_BINS}
        COMMENT "Aggregating resources"
)

if (DEFINED RESGEN_SOURCE_FLAGS)
    set_source_files_properties(${RESGEN_SOURCE} PROPERTIES COMPILE_FLAGS ${RESGEN_SOURCE_FLAGS})
endif()

# CMake fails to invoke ar on Windows unless there is at least one C/C++ file in the library.
set(DUMMY_SRC "${RESOURCE_DIR}/dummy.c")
add_custom_command(OUTPUT ${DUMMY_SRC} COMMAND echo "//" > ${DUMMY_SRC})

add_library(sample-resources ${DUMMY_SRC} ${RESGEN_SOURCE})

# ==================================================================================================
# Invoke cmgen to build KTX files for the default IBL and skybox
# ==================================================================================================

set(CMGEN_ARGS --quiet --format=ktx --size=256 --extract-blur=0.1)

function(add_envmap SOURCE TARGET)
    set(source_envmap "${ROOT_DIR}/${SOURCE}")

    set(target_skybox   "${PROJECT_BINARY_DIR}/${TARGET}/${TARGET}_skybox.ktx")
    set(target_envmap   "${PROJECT_BINARY_DIR}/${TARGET}/${TARGET}_ibl.ktx")

    set(target_envmaps ${target_envmaps} ${target_skybox} PARENT_SCOPE)
    set(target_envmaps ${target_envmaps} ${target_envmap} PARENT_SCOPE)

    add_custom_command(OUTPUT ${target_skybox} ${target_envmap}
        COMMAND cmgen -x ${TARGET} ${CMGEN_ARGS} ${source_envmap}
        MAIN_DEPENDENCY ${source_envmap}
        DEPENDS cmgen
        COMMENT "Generating environment map ${target_envmap}")
endfunction()

add_envmap("third_party/environments/lightroom_14b.hdr" "default_env")

# ==================================================================================================
# Build suzanne resources
# ==================================================================================================

function(add_ktxfiles SOURCE TARGET EXTRA_ARGS)
    set(source_path "${ROOT_DIR}/${SOURCE}")
    set(target_path "${TEXTURE_DIR}/${TARGET}")
    set(MONKEY_FILES ${MONKEY_FILES} ${target_path} PARENT_SCOPE)
    add_custom_command(
        OUTPUT ${target_path}
        COMMAND mipgen --quiet ${EXTRA_ARGS} ${source_path} ${target_path}
        MAIN_DEPENDENCY ${source_path}
        DEPENDS mipgen
        COMMENT "Generating mip levels for ${source_path}")
endfunction()

function(add_pngfile SOURCE TARGET)
    set(source_path "${ROOT_DIR}/${SOURCE}")
    set(target_path "${TEXTURE_DIR}/${TARGET}")
    set(MONKEY_FILES ${MONKEY_FILES} ${target_path} PARENT_SCOPE)
    add_custom_command(
        OUTPUT ${target_path}
        COMMAND ${CMAKE_COMMAND} -E copy ${source_path} ${target_path}
        MAIN_DEPENDENCY ${source_path})
endfunction()

function(add_mesh SOURCE TARGET)
    set(source_mesh "${ROOT_DIR}/${SOURCE}")
    set(target_mesh "${RESOURCE_DIR}/${TARGET}")
    set(MONKEY_FILES ${MESH_BINS} ${target_mesh} PARENT_SCOPE)
    add_custom_command(
        OUTPUT ${target_mesh}
        COMMAND filamesh --compress ${source_mesh} ${target_mesh}
        MAIN_DEPENDENCY ${source_mesh}
        DEPENDS filamesh)
endfunction()

add_mesh("assets/models/monkey/monkey.obj" "suzanne.filamesh")

# Use a RGBA compression format for Metal and Vulkan support.
set (COMPRESSION "--compression=s3tc_rgba_dxt5")
add_ktxfiles("assets/models/monkey/albedo.png" "albedo_s3tc.ktx" "${COMPRESSION}")
add_ktxfiles("assets/models/monkey/roughness.png" "roughness.ktx" "${COMPRESSION};--grayscale")
add_ktxfiles("assets/models/monkey/metallic.png" "metallic.ktx" "${COMPRESSION};--grayscale")
add_ktxfiles("assets/models/monkey/ao.png" "ao.ktx" "${COMPRESSION};--grayscale")

add_pngfile("assets/models/monkey/normal.png" "normal.png")

get_resgen_vars(${RESOURCE_DIR} monkey)

add_custom_command(
        OUTPUT ${RESGEN_OUTPUTS}
        COMMAND resgen ${RESGEN_FLAGS} ${MONKEY_FILES}
        DEPENDS resgen ${MONKEY_FILES}
        COMMENT "Aggregating monkey resources"
)

if (DEFINED RESGEN_SOURCE_FLAGS)
    set_source_files_properties(${RESGEN_SOURCE} PROPERTIES COMPILE_FLAGS ${RESGEN_SOURCE_FLAGS})
endif()

add_library(suzanne-resources ${DUMMY_SRC} ${RESGEN_SOURCE})

# ==================================================================================================
# Resources for gltf_viewer
# ==================================================================================================

add_library(gltf-resources ${DUMMY_SRC})

set(GLTF_VIEWER_RESOURCE_FILES
    ${ROOT_DIR}/third_party/models/DamagedHelmet/DamagedHelmet.glb
    ${MATERIAL_DIR}/groundShadow.filamat
)

get_resgen_vars(${RESOURCE_DIR} gltf_viewer)

add_custom_command(
        OUTPUT ${RESGEN_OUTPUTS}
        COMMAND resgen ${RESGEN_FLAGS} ${GLTF_VIEWER_RESOURCE_FILES}
        DEPENDS resgen ${GLTF_VIEWER_RESOURCE_FILES}
)

if (DEFINED RESGEN_SOURCE_FLAGS)
    set_source_files_properties(${RESGEN_SOURCE} PROPERTIES COMPILE_FLAGS ${RESGEN_SOURCE_FLAGS})
endif()

target_sources(gltf-resources PRIVATE ${RESGEN_SOURCE})

# ==================================================================================================
# Compiler flags
# ==================================================================================================

if (MSVC)
    set(COMPILER_FLAGS $<$<CONFIG:Release>:/fp:fast>)
else()
    set(COMPILER_FLAGS
            -Wno-extern-c-compat
            $<$<NOT:$<PLATFORM_ID:Linux>>:-Wno-address-of-packed-member>
            $<$<CONFIG:Release>:-ffast-math>)
endif()

# ==================================================================================================
# Test apps
# ==================================================================================================

function(add_demo NAME)
    include_directories(${GENERATION_ROOT})
    add_executable(${NAME} ${NAME}.cpp)
    target_link_libraries(${NAME} PRIVATE sample-resources filamentapp)
    target_compile_options(${NAME} PRIVATE ${COMPILER_FLAGS})
endfunction()

if (NOT ANDROID)
    add_demo(animation)
    add_demo(depthtesting)
    add_demo(frame_generator)
    add_demo(gltf_viewer)
    add_demo(gltf_instances)
    add_demo(heightfield)
    add_demo(hellopbr)
    add_demo(hellotriangle)
    add_demo(lightbulb)
    add_demo(material_sandbox)
    add_demo(multiple_windows)
    add_demo(point_sprites)
    add_demo(rendertarget)
    add_demo(sample_cloth)
    add_demo(sample_full_pbr)
    add_demo(sample_normal_map)
    add_demo(shadowtest)
    add_demo(strobecolor)
    add_demo(suzanne)
    add_demo(texturedquad)
    add_demo(vbotest)
    add_demo(viewtest)

    # Sample app specific
    target_link_libraries(frame_generator PRIVATE imageio)
    target_link_libraries(gltf_viewer PRIVATE gltf-resources gltfio viewer)
    target_link_libraries(gltf_instances PRIVATE gltf-resources gltfio viewer)
    target_link_libraries(hellopbr PRIVATE filameshio suzanne-resources)
    target_link_libraries(multiple_windows PRIVATE filameshio suzanne-resources)
    target_link_libraries(rendertarget PRIVATE filameshio suzanne-resources)
    target_link_libraries(sample_cloth PRIVATE filameshio)
    target_link_libraries(sample_normal_map PRIVATE filameshio)
    target_link_libraries(suzanne PRIVATE filameshio suzanne-resources)

endif()

# ==================================================================================================
# Copy the MoltenVK dylibs and JSON on MacOS
# ==================================================================================================

if (APPLE AND NOT Vulkan_LIBRARY AND NOT FILAMENT_USE_SWIFTSHADER)
    message(STATUS "No Vulkan SDK was found, using prebuilt MoltenVK.")
    set(MOLTENVK_DIR "../third_party/moltenvk")
    configure_file(
            ${MOLTENVK_DIR}/libvulkan.1.dylib
            ${PROJECT_BINARY_DIR}/libvulkan.1.dylib COPYONLY)
    configure_file(
            ${MOLTENVK_DIR}/MoltenVK_icd.json
            ${PROJECT_BINARY_DIR}/MoltenVK_icd.json COPYONLY)
    configure_file(
            ${MOLTENVK_DIR}/libMoltenVK.dylib
            ${PROJECT_BINARY_DIR}/libMoltenVK.dylib COPYONLY)
    list(APPEND APP_SRCS
            ${PROJECT_BINARY_DIR}/libvulkan.1.dylib
            ${PROJECT_BINARY_DIR}/MoltenVK_icd.json
            ${PROJECT_BINARY_DIR}/libMoltenVK.dylib)
endif()

# ==================================================================================================
# Copy Assets
#
# This creates a structure like:
#
#    /lightbulb (etc)  sample app executable
#    /assets/fonts     copied from REPO/assets
#    /assets/models    copied from REPO/assets
#    /textures         copied from REPO/third_party
#
# ==================================================================================================

file(COPY ../third_party/textures DESTINATION ${PROJECT_BINARY_DIR})
add_custom_target(textures ALL DEPENDS textures)
add_dependencies(filament textures)

file(COPY ../assets DESTINATION ${PROJECT_BINARY_DIR}
        PATTERN "reference" EXCLUDE
        PATTERN "environments" EXCLUDE)
add_custom_target(assets ALL DEPENDS assets)
add_dependencies(filament assets)

add_custom_target(envs DEPENDS ${target_envmaps})
add_dependencies(filament envs)
