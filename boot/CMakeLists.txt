## efisys.bin

# EFI platform ID, used in environ/CMakelists.txt for bootmgfw filename naming also.
if(ARCH STREQUAL "amd64")
    set(EFI_PLATFORM_ID "x64")
elseif(ARCH STREQUAL "i386")
    set(EFI_PLATFORM_ID "ia32")
elseif(ARCH STREQUAL "ia64")
    set(EFI_PLATFORM_ID "ia64")
elseif(ARCH STREQUAL "arm")
    set(EFI_PLATFORM_ID "arm")
elseif(ARCH STREQUAL "aarch64")
    set(EFI_PLATFORM_ID "aa64")
else()
    message(FATAL_ERROR "Unknown ARCH '" ${ARCH} "', cannot generate a valid UEFI boot filename.")
endif()

add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/efisys.bin
    COMMAND native-fatten ${CMAKE_CURRENT_BINARY_DIR}/efisys.bin -format 2880 EFIBOOT -boot ${CMAKE_CURRENT_BINARY_DIR}/freeldr/bootsect/fat.bin -mkdir EFI -mkdir EFI/BOOT -add $<TARGET_FILE:bootmgfw> EFI/BOOT/boot${EFI_PLATFORM_ID}.efi
    DEPENDS native-fatten fat bootmgfw
    VERBATIM)
add_custom_target(efisys DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/efisys.bin)

# Create an 'empty' directory (guaranteed to be empty) to be able to add
# arbitrary empty directories to the ISO image using mkisofs.
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/empty)

# Retrieve the full paths to the generated files of the 'isombr', 'isoboot', 'isobtrt' and 'efisys' targets
set(_isombr_file  ${CMAKE_CURRENT_BINARY_DIR}/freeldr/bootsect/isombr.bin)  # get_target_property(_isombr_file  isombr  LOCATION)
set(_isoboot_file ${CMAKE_CURRENT_BINARY_DIR}/freeldr/bootsect/isoboot.bin) # get_target_property(_isoboot_file isoboot LOCATION)
set(_isobtrt_file ${CMAKE_CURRENT_BINARY_DIR}/freeldr/bootsect/isobtrt.bin) # get_target_property(_isobtrt_file isobtrt LOCATION)
set(_efisys_file  ${CMAKE_CURRENT_BINARY_DIR}/efisys.bin) # get_target_property(_efisys_file  efisys  LOCATION)

# Create a mkisofs sort file to specify an explicit ordering for the boot files
# to place them at the beginning of the image (makes ISO image analysis easier).
# See mkisofs/schilytools/mkisofs/README.sort for more details.
# As the default file sort weight is '0', give the boot files sort weights >= 1.
# Note that it is sad that '-sort' does not work using grafted points, and as a
# result we need in particular to use the boot catalog file "path" mkisofs that
# mkisofs expects, that is, the boot catalog file name is appended to the first
# host-system path listed in the file list, whatever it is, and that does not
# work well if the first item is a graft point (and especially if it's a file
# and not a directory). To fix that, the trick is to use as the first file item
# the empty directory created earlier. This ensures that:
# - the boot catalog file path is meaningful;
# - since its contents are included by mkisofs in the root of the ISO image,
#   using the empty directory ensures that no extra unwanted files are added.
#
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/bootfiles.sort "\
${CMAKE_CURRENT_BINARY_DIR}/empty/boot.catalog 4
${_isoboot_file} 3
${_isobtrt_file} 2
${_efisys_file} 1
")

# ISO image identifier names
set(ISO_MANUFACTURER "ReactOS Foundation")  # For both the publisher and the preparer
set(ISO_VOLNAME      "ReactOS")             # For both the Volume ID and the Volume set ID


# Create user profile directories in the LiveImage
function(add_allusers_profile_dirs _target _rootdir)
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/All Users/Application Data=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/All Users/Documents/My Music=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/All Users/Documents/My Pictures=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/All Users/Documents/My Videos=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/All Users/Favorites=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/All Users/My Documents=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/All Users/Start Menu/Programs/StartUp=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/All Users/Templates=${CMAKE_CURRENT_BINARY_DIR}/empty")
endfunction()
function(add_user_profile_dirs _target _rootdir _username)
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Application Data=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Application Data/Microsoft/Internet Explorer/Quick Launch=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Cookies=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Desktop=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Favorites=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Local Settings/Application Data=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Local Settings/History=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Local Settings/Temporary Internet Files=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/My Music=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/My Pictures=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/My Videos=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/NetHood=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/PrintHood=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Recent=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/SendTo=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Start Menu/Programs=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Start Menu/Programs/Administrative Tools=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Start Menu/Programs/StartUp=${CMAKE_CURRENT_BINARY_DIR}/empty")
    set_property(TARGET ${_target} APPEND PROPERTY FILE_LIST "${_rootdir}/${_username}/Templates=${CMAKE_CURRENT_BINARY_DIR}/empty")
endfunction()


## BootCD
# create file list on generation phase
file(GENERATE 
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/bootcd.$<CONFIG>.lst
    CONTENT $<GENEX_EVAL:$<JOIN:$<TARGET_PROPERTY:bootcd,FILE_LIST>,\n>>)
# create target
add_custom_command(
    OUTPUT ${REACTOS_BINARY_DIR}/bootcd.iso
    COMMAND native-mkisofs -quiet -o ${REACTOS_BINARY_DIR}/bootcd.iso -iso-level 4
        -publisher ${ISO_MANUFACTURER} -preparer ${ISO_MANUFACTURER} -volid ${ISO_VOLNAME} -volset ${ISO_VOLNAME}
        -eltorito-boot loader/isoboot.bin -no-emul-boot -boot-load-size 4 -eltorito-alt-boot -eltorito-platform efi -eltorito-boot loader/efisys.bin -no-emul-boot -hide boot.catalog
        -sort ${CMAKE_CURRENT_BINARY_DIR}/bootfiles.sort
        -no-cache-inodes -graft-points -path-list ${CMAKE_CURRENT_BINARY_DIR}/bootcd.$<CONFIG>.lst
    COMMAND native-isohybrid -b ${_isombr_file} -t 0x96 ${REACTOS_BINARY_DIR}/bootcd.iso
    DEPENDS isombr native-isohybrid native-mkisofs $<TARGET_PROPERTY:bootcd,ISO_DEPENDENCIES>
    VERBATIM)
add_custom_target(bootcd DEPENDS ${REACTOS_BINARY_DIR}/bootcd.iso)

## BootCDRegTest
# create file list on generation phase
file(GENERATE 
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/bootcdregtest.$<CONFIG>.lst
    CONTENT $<GENEX_EVAL:$<JOIN:$<TARGET_PROPERTY:bootcdregtest,FILE_LIST>,\n>>)
# create target
add_custom_command(
    OUTPUT ${REACTOS_BINARY_DIR}/bootcdregtest.iso
    COMMAND native-mkisofs -quiet -o ${REACTOS_BINARY_DIR}/bootcdregtest.iso -iso-level 4
        -publisher ${ISO_MANUFACTURER} -preparer ${ISO_MANUFACTURER} -volid ${ISO_VOLNAME} -volset ${ISO_VOLNAME}
        -eltorito-boot loader/isobtrt.bin -no-emul-boot -boot-load-size 4 -eltorito-alt-boot -eltorito-platform efi -eltorito-boot loader/efisys.bin -no-emul-boot -hide boot.catalog
        -sort ${CMAKE_CURRENT_BINARY_DIR}/bootfiles.sort
        -no-cache-inodes -graft-points -path-list ${CMAKE_CURRENT_BINARY_DIR}/bootcdregtest.$<CONFIG>.lst
    COMMAND native-isohybrid -b ${_isombr_file} -t 0x96 ${REACTOS_BINARY_DIR}/bootcdregtest.iso
    DEPENDS isombr native-isohybrid native-mkisofs $<TARGET_PROPERTY:bootcdregtest,ISO_DEPENDENCIES>
    VERBATIM)
add_custom_target(bootcdregtest DEPENDS ${REACTOS_BINARY_DIR}/bootcdregtest.iso)

## LiveCD
# create file list on generation phase
file(GENERATE 
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/livecd.$<CONFIG>.lst
    CONTENT $<GENEX_EVAL:$<JOIN:$<TARGET_PROPERTY:livecd,FILE_LIST>,\n>>)
# create target
add_custom_command(
    OUTPUT ${REACTOS_BINARY_DIR}/livecd.iso
    COMMAND native-mkisofs -quiet -o ${REACTOS_BINARY_DIR}/livecd.iso -iso-level 4
        -publisher ${ISO_MANUFACTURER} -preparer ${ISO_MANUFACTURER} -volid ${ISO_VOLNAME} -volset ${ISO_VOLNAME}
        -eltorito-boot loader/isoboot.bin -no-emul-boot -boot-load-size 4 -eltorito-alt-boot -eltorito-platform efi -eltorito-boot loader/efisys.bin -no-emul-boot -hide boot.catalog
        -sort ${CMAKE_CURRENT_BINARY_DIR}/bootfiles.sort
        -no-cache-inodes -graft-points -path-list ${CMAKE_CURRENT_BINARY_DIR}/livecd.$<CONFIG>.lst
    COMMAND native-isohybrid -b ${_isombr_file} -t 0x96 ${REACTOS_BINARY_DIR}/livecd.iso
    DEPENDS isombr native-isohybrid native-mkisofs ${CMAKE_CURRENT_BINARY_DIR}/livecd.$<CONFIG>.lst $<TARGET_PROPERTY:livecd,ISO_DEPENDENCIES>
    VERBATIM)
add_custom_target(livecd DEPENDS ${REACTOS_BINARY_DIR}/livecd.iso)
# Create user profile directories
add_allusers_profile_dirs(livecd "Profiles")
add_user_profile_dirs(livecd "Profiles" "Default User")

## HybridCD
# create file list on generation phase
file(GENERATE 
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/hybridcd.$<CONFIG>.lst
    CONTENT $<GENEX_EVAL:$<JOIN:$<TARGET_PROPERTY:hybridcd,FILE_LIST>,\n>>)
# create target
add_custom_command(
    OUTPUT ${REACTOS_BINARY_DIR}/hybridcd.iso
    COMMAND native-mkisofs -quiet -o ${REACTOS_BINARY_DIR}/hybridcd.iso -iso-level 4
        -publisher ${ISO_MANUFACTURER} -preparer ${ISO_MANUFACTURER} -volid ${ISO_VOLNAME} -volset ${ISO_VOLNAME}
        -eltorito-boot loader/isoboot.bin -no-emul-boot -boot-load-size 4 -eltorito-alt-boot -eltorito-platform efi -eltorito-boot loader/efisys.bin -no-emul-boot -hide boot.catalog
        -sort ${CMAKE_CURRENT_BINARY_DIR}/bootfiles.sort
        -duplicates-once -no-cache-inodes -graft-points -path-list ${CMAKE_CURRENT_BINARY_DIR}/hybridcd.$<CONFIG>.lst
    COMMAND native-isohybrid -b ${_isombr_file} -t 0x96 ${REACTOS_BINARY_DIR}/hybridcd.iso
    DEPENDS isombr native-isohybrid native-mkisofs ${CMAKE_CURRENT_BINARY_DIR}/hybridcd.$<CONFIG>.lst $<TARGET_PROPERTY:hybridcd,ISO_DEPENDENCIES>
    VERBATIM)
add_custom_target(hybridcd DEPENDS ${REACTOS_BINARY_DIR}/hybridcd.iso)
# Create user profile directories
add_allusers_profile_dirs(hybridcd "livecd/Profiles")
add_user_profile_dirs(livecd "livecd/Profiles" "Default User")

add_cd_file(TARGET efisys FILE ${CMAKE_CURRENT_BINARY_DIR}/efisys.bin DESTINATION loader NO_CAB NOT_IN_HYBRIDCD FOR bootcd regtest livecd hybridcd)
add_cd_file(TARGET livecd FILE ${REACTOS_BINARY_DIR}/livecd.iso DESTINATION livecd FOR hybridcd)

add_subdirectory(freeldr)
add_subdirectory(bootdata)
add_subdirectory(environ)
