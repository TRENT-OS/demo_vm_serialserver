#
# Demo Serial Server VM
#
# Copyright (C) 2022-2024, HENSOLDT Cyber GmbH
# 
# SPDX-License-Identifier: BSD-2-Clause
#

cmake_minimum_required(VERSION 3.8.2)

project(demo_vm_minimal C)

#-------------------------------------------------------------------------------
# use the SDK
find_package(os-sdk REQUIRED)
os_sdk_set_defaults()

# Kernel settings
set(KernelArmHypervisorSupport ON CACHE BOOL "" FORCE)
# set(KernelArmVtimerUpdateVOffset OFF CACHE BOOL "" FORCE)
# set(KernelArmDisableWFIWFETraps ON CACHE BOOL "" FORCE)
set(VmPCISupport ON CACHE BOOL "" FORCE)
set(LibUSB OFF CACHE BOOL "" FORCE)
set(VmInitRdFile ON CACHE BOOL "" FORCE)
set(VmVirtioConsole ON CACHE BOOL "" FORCE)

include("${CMAKE_CURRENT_LIST_DIR}/plat/${PLATFORM}/plat.cmake" OPTIONAL)
os_sdk_setup()
CAmkESAddCPPInclude("plat/${PLATFORM}")
os_sdk_import_camkes_vm()
os_sdk_import_from_global_components(
    "components/FileServer"
    "components/SerialServer"
    "components/TimeServer"
    "components/VirtQueue"
)

#-------------------------------------------------------------------------------
# config
AddCamkesCPPFlag(cpp_flags CONFIG_VARS VmEmmc2NoDMA)

if(NOT KernelPlatformQEMUArmVirt)
    AddCamkesCPPFlag(cpp_flags CONFIG_VARS VmVUSB)
endif()

if(KernelPlatformTK1)
    AddCamkesCPPFlag(cpp_flags CONFIG_VARS Tk1DeviceFwd Tk1Insecure)
endif()


#-------------------------------------------------------------------------------
# VM Images

list(
    APPEND CMAKE_MODULE_PATH
    "${CMAKE_CURRENT_LIST_DIR}/camkes-vm-images"
    "${CMAKE_CURRENT_LIST_DIR}/camkes-vm-linux"
)
find_package(camkes-vm-images REQUIRED)
find_package(camkes-vm-linux REQUIRED)

include("${CAMKES_VM_LINUX_HELPERS_PATH}")

# images for the selected platform
set(VM_IMG_LINUX "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/linux")
set(VM_IMG_ROOTFS "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/rootfs.cpio.gz")
# set(VM_IMG_DTB "${CAMKES_VM_IMAGES_DIR}/${PLATFORM}/linux-dtb")

# # patch DTB to use proper 'initrd-start' and 'initrd-end'
# UpdateDtbFromInitrd(
#     "${VM_IMG_DTB}"
#     "${VM_IMG_ROOTFS}"
#     "0x4d700000" # initrd start
#     target_dtb_gen
#     VM_IMG_DTB_PATCHED
# )

# Setup our passthrough vm overlay
AddFileToOverlayDir(
    "inittab"
    ${CMAKE_CURRENT_LIST_DIR}/overlay_files/init_scripts/inittab_hvc0
    "etc"
    overlay
)

AddOverlayDirToRootfs(
    overlay
    ${VM_IMG_ROOTFS}
    "buildroot"
    "rootfs_install"
    output_overlayed_rootfs_location
    rootfs_target
    GZIP
)

message("--------------------------------${output_overlayed_rootfs_location}")
# DefineCAmkESVMFileServer() defines a CAmkESVMFileServer using the declared
# no explicit dependency to "rootfs_target" required, because implicit dependency on "output_overlayed_rootfs_location" is sufficient
DefineCAmkESVMFileServer(
    INSTANCE "fileServer"
    FILES
        "linux:${VM_IMG_LINUX}"
        "linux-initrd:${output_overlayed_rootfs_location}"
    # DEPENDS
    #     rootfs_target
)

# # fileserver images and dependencies. Images are placed into a CPIO archive.
# get_target_property(FS_FILES vm_fserver_config FILES)
# get_target_property(FS_DEPS vm_fserver_config DEPS)
# # Build CPIO archive given the defined kernel and rootfs images
# include(cpio)
# MakeCPIO("file_server_archive.o" "${FS_FILES}" DEPENDS "${FS_DEPS}")
# add_library(fileserver_cpio STATIC EXCLUDE_FROM_ALL file_server_archive.o)
# set_property(TARGET fileserver_cpio PROPERTY LINKER_LANGUAGE C)
# ExtendCAmkESComponentInstance(FileServer fileServer LIBS fileserver_cpio)


# CAmkESAddCPPInclude("${SDK_SEL4_CAMKES_DIR}/libs/sel4_projects_libs/libsel4vm/include/sel4vm")

#-------------------------------------------------------------------------------

#CAmkESAddImportPath(${KernelARMPlatform})

# Declare root server
#DeclareCAmkESRootserver(
#    vm_minimal.camkes
#    CPP_FLAGS
#        ${cpp_flags}
#    CPP_INCLUDES
#        ${CAMKES_VM_DIR}/components/VM_Arm
#)

os_sdk_create_CAmkES_system("vm_serial_server.camkes")
