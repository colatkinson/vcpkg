cmake_minimum_required(VERSION 3.0)
project(glib C)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 11)

if(MSVC)
    add_compile_options(/utf-8)
endif()
set(GLIB_DLL_SUFFIX 2)
set(GLIB_LIB_SUFFIX 2.0)

if(CMAKE_BUILD_TYPE STREQUAL Debug)
    add_definitions(-DG_ENABLE_DEBUG)
endif()

if(BUILD_SHARED_LIBS)
    add_definitions(-DDLL_EXPORT)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

# find dependencies

# zlib
find_package(ZLIB REQUIRED)
# pcre
find_path(PCRE_INCLUDE_DIR pcre.h)
if(CMAKE_BUILD_TYPE STREQUAL Debug)
    set(PCRE_SUFFIX d)
endif()
find_library(PCRE_LIBRARY NAMES pcre${PCRE_SUFFIX} pcre)
# libiconv
find_package(unofficial-iconv REQUIRED)
# libffi
find_path(FFI_INCLUDE_DIR ffi.h)
find_library(FFI_LIBRARY NAMES ffi libffi)
get_filename_component(LIB_DIR "${FFI_LIBRARY}" DIRECTORY)
if(APPLE)
    find_library(COREFOUNDATION_LIBRARY CoreFoundation)
    find_library(FOUNDATION_LIBRARY Foundation)
    find_library(CORESERVICES_LIBRARY CoreServices)
    link_libraries(${CORESERVICES_LIBRARY} ${COREFOUNDATION_LIBRARY} ${FOUNDATION_LIBRARY})
endif()

if(WIN32 OR APPLE)
    # libintl(gettext)
    find_path(LIBINTL_INCLUDE_DIR libintl.h)
    find_library(LIBINTL_LIBRARY NAMES intl libintl)
else()
    set(LIBINTL_INCLUDE_DIR)
    set(LIBINTL_LIBRARY)
endif()

if(WIN32)
    set(LIBRESOLV_LIBRARY)
else()
    find_library(LIBRESOLV_LIBRARY NAMES resolv libresolv)
endif()

#prepare config files
if(WIN32)
    configure_file(config.h.win32 ${CMAKE_BINARY_DIR}/config/config.h COPYONLY)
    configure_file(glib/glibconfig.h.win32 ${CMAKE_BINARY_DIR}/config/glib/glibconfig.h COPYONLY)
    configure_file(gmodule/gmoduleconf.h.win32 ${CMAKE_BINARY_DIR}/config/gmodule/gmoduleconf.h COPYONLY)
    configure_file(gio/gnetworking.h.win32 ${CMAKE_BINARY_DIR}/config/gio/gnetworking.h COPYONLY)
    add_definitions(-DHAVE_CONFIG_H)
else()
    file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/config)
    set(ENV{LIBFFI_LIBS} "${FFI_LIBRARY}")
    set(ENV{PCRE_LIBS} "${PCRE_LIBRARY}")
    set(ENV{LIBFFI_CFLAGS} "-I${FFI_INCLUDE_DIR}")
    set(ENV{PCRE_CFLAGS} "-I${PCRE_INCLUDE_DIR}")
    set(ENV{MSGFMT} "/bin/echo")
    set(ENV{GMSGFMT} "/bin/echo")
    if(NOT EXISTS "${CMAKE_BINARY_DIR}/config/config.h")
        string(TOUPPER UPPER_CONFIG "${CMAKE_BUILD_TYPE}")
        set(CXXFLAGS "${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_${UPPER_CONFIG}} -I${PCRE_INCLUDE_DIR}")
        set(CFLAGS "${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_${UPPER_CONFIG}} -I${PCRE_INCLUDE_DIR}")
        if(BUILD_SHARED_LIBS)
            set(LDFLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${CMAKE_SHARED_LINKER_FLAGS_${UPPER_CONFIG}} -L${LIB_DIR}")
        else()
            set(LDFLAGS "${CMAKE_STATIC_LINKER_FLAGS} ${CMAKE_STATIC_LINKER_FLAGS_${UPPER_CONFIG}} -L${LIB_DIR}")
        endif()

        string(STRIP "${CXXFLAGS}" CXXFLAGS)
        string(STRIP "${CFLAGS}"   CFLAGS)
        string(STRIP "${LDFLAGS}"  LDFLAGS)
        execute_process(
            COMMAND "${CMAKE_SOURCE_DIR}/configure"
                --disable-libelf
                --disable-libmount
                "CPPFLAGS=${CXXFLAGS}"
                "CFLAGS=${CFLAGS}"
                "LDFLAGS=${LDFLAGS}"
            WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/config
            RESULT_VARIABLE res
        )
        if(NOT res EQUAL 0)
            message(FATAL_ERROR "Configure failed.")
        endif()
    endif()
    add_definitions("-DGIO_MODULE_DIR=\"gio/modules\"")
endif()

include_directories(${CMAKE_BINARY_DIR}/config ${CMAKE_BINARY_DIR}/config/glib ${CMAKE_BINARY_DIR}/config/gio ${CMAKE_BINARY_DIR}/config/gmodule)
include_directories(. glib)

# main module
set(GLIB_SOURCES
    ./glib/deprecated/gallocator.c
    ./glib/deprecated/gcache.c
    ./glib/deprecated/gcompletion.c
    ./glib/deprecated/grel.c
    ./glib/deprecated/gthread-deprecated.c
    ./glib/docs.c
    ./glib/garray.c
    ./glib/gasyncqueue.c
    ./glib/gatomic.c
    ./glib/gbacktrace.c
    ./glib/gbase64.c
    ./glib/gbitlock.c
    ./glib/gbookmarkfile.c
    ./glib/gbytes.c
    ./glib/gcharset.c
    ./glib/gchecksum.c
    ./glib/gconvert.c
    ./glib/gdataset.c
    ./glib/gdate.c
    ./glib/gdatetime.c
    ./glib/gdir.c
    ./glib/genviron.c
    ./glib/gerror.c
    ./glib/gfileutils.c
    ./glib/ggettext.c
    ./glib/ghash.c
    ./glib/ghmac.c
    ./glib/ghook.c
    ./glib/ghostutils.c
    ./glib/giochannel.c
    ./glib/giowin32.c
    ./glib/gkeyfile.c
    ./glib/glib-init.c
    ./glib/glib-private.c
    ./glib/glist.c
    ./glib/gmain.c
    ./glib/gmappedfile.c
    ./glib/gmarkup.c
    ./glib/gmem.c
    ./glib/gmessages.c
    ./glib/gnode.c
    ./glib/gnulib/asnprintf.c
    ./glib/gnulib/printf-args.c
    ./glib/gnulib/printf-parse.c
    ./glib/gnulib/printf.c
    ./glib/gnulib/vasnprintf.c
    ./glib/goption.c
    ./glib/gpattern.c
    ./glib/gpoll.c
    ./glib/gprimes.c
    ./glib/gprintf.c
    ./glib/gqsort.c
    ./glib/gquark.c
    ./glib/gqueue.c
    ./glib/grand.c
    ./glib/gregex.c
    ./glib/gscanner.c
    ./glib/gsequence.c
    ./glib/gshell.c
    ./glib/gslice.c
    ./glib/gslist.c
    ./glib/gspawn-win32.c
    ./glib/gstdio.c
    ./glib/gstrfuncs.c
    ./glib/gstring.c
    ./glib/gstringchunk.c
    ./glib/gtestutils.c
    ./glib/gthread-win32.c
    ./glib/gthread.c
    ./glib/gthreadpool.c
    ./glib/gtimer.c
    ./glib/gtimezone.c
    ./glib/gtranslit.c
    ./glib/gtrashstack.c
    ./glib/gtree.c
    ./glib/gunibreak.c
    ./glib/gunicollate.c
    ./glib/gunidecomp.c
    ./glib/guniprop.c
    ./glib/gurifuncs.c
    ./glib/gutf8.c
    ./glib/gutils.c
    ./glib/guuid.c
    ./glib/gvariant-core.c
    ./glib/gvariant-parser.c
    ./glib/gvariant-serialiser.c
    ./glib/gvariant.c
    ./glib/gvarianttype.c
    ./glib/gvarianttypeinfo.c
    ./glib/gversion.c
    ./glib/gwakeup.c
    ./glib/gwin32.c
    ./glib/libcharset/localcharset.c
)
add_library(glib ${GLIB_SOURCES})
target_compile_definitions(glib PRIVATE GLIB_COMPILATION G_LOG_DOMAIN="GLib" LIBDIR="")
target_link_libraries(glib PRIVATE ${PCRE_LIBRARY} unofficial::iconv::libiconv unofficial::iconv::libcharset ${LIBINTL_LIBRARY})
if(WIN32)
    target_compile_definitions(glib PRIVATE USE_SYSTEM_PCRE)
    target_link_libraries(glib PRIVATE ws2_32 winmm advapi32 ole32 shell32)
else()
    set(THREADS_PREFER_PTHREAD_FLAG ON)
    find_package(Threads REQUIRED)
    target_link_libraries(glib PRIVATE Threads::Threads ${CMAKE_DL_LIBS})
endif()
target_include_directories(glib PRIVATE ${PCRE_INCLUDE_DIR} ${ICONV_INCLUDE_DIR})
target_include_directories(glib PUBLIC $<BUILD_INTERFACE:${LIBINTL_INCLUDE_DIR}> $<INSTALL_INTERFACE:include>)
list(APPEND GLIB_TARGETS glib)
if(NOT GLIB_SKIP_HEADERS)
    install(FILES glib/glib.h glib/glib-object.h ${CMAKE_BINARY_DIR}/config/glib/glibconfig.h DESTINATION include)

    file(GLOB GLIB_HEADERS glib/*.h)
    list(FILTER GLIB_HEADERS EXCLUDE REGEX "/glib.h\$|/glib-object.h\$|private.h\$")
    install(FILES ${GLIB_HEADERS} DESTINATION include/glib)

    file(GLOB GLIB_DEP_HEADERS glib/deprecated/*.h)
    install(FILES ${GLIB_DEP_HEADERS} DESTINATION include/glib/deprecated)
endif()

# gthread
add_library(gthread gthread/gthread-impl.c)
target_compile_definitions(gthread PRIVATE G_LOG_DOMAIN="GThread")
target_link_libraries(gthread PRIVATE glib ${LIBINTL_LIBRARY})
target_include_directories(gthread PRIVATE ${LIBINTL_INCLUDE_DIR})
list(APPEND GLIB_TARGETS gthread)

# gobject
set(GOBJECT_SOURCES
    ./gobject/gatomicarray.c
    ./gobject/gbinding.c
    ./gobject/gboxed.c
    ./gobject/gclosure.c
    ./gobject/genums.c
    ./gobject/gmarshal.c
    ./gobject/gobject.c
    ./gobject/gparam.c
    ./gobject/gparamspecs.c
    ./gobject/gsignal.c
    ./gobject/gsourceclosure.c
    ./gobject/gtype.c
    ./gobject/gtypemodule.c
    ./gobject/gtypeplugin.c
    ./gobject/gvalue.c
    ./gobject/gvaluearray.c
    ./gobject/gvaluetransform.c
    ./gobject/gvaluetypes.c
)
add_library(gobject ${GOBJECT_SOURCES})
target_compile_definitions(gobject PRIVATE GOBJECT_COMPILATION G_LOG_DOMAIN="GLib-GObject")
target_link_libraries(gobject PRIVATE gthread glib ${FFI_LIBRARY})
target_include_directories(gobject PRIVATE ${FFI_INCLUDE_DIR} PUBLIC $<INSTALL_INTERFACE:include>)
list(APPEND GLIB_TARGETS gobject)
if(NOT GLIB_SKIP_HEADERS)
    file(GLOB GOBJECT_HEADERS gobject/*.h gobject/gobjectnotifyqueue.c)
    list(FILTER GOBJECT_HEADERS EXCLUDE REGEX "private.h\$")
    install(FILES ${GOBJECT_HEADERS} DESTINATION include/gobject)
endif()

# gmodule
add_library(gmodule gmodule/gmodule.c)
target_compile_definitions(gmodule PRIVATE G_LOG_DOMAIN="GModule")
target_link_libraries(gmodule PRIVATE glib ${LIBINTL_LIBRARY})
target_include_directories(gmodule PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/gmodule>)
target_include_directories(gmodule PRIVATE ${LIBINTL_INCLUDE_DIR} PUBLIC $<INSTALL_INTERFACE:include>)
list(APPEND GLIB_TARGETS gmodule)
if(NOT GLIB_SKIP_HEADERS)
    install(FILES gmodule/gmodule.h DESTINATION include)
endif()

# gio subdirs
if(NOT WIN32)
    file(GLOB XDGMIME_SOURCES gio/xdgmime/*.c)
    add_library(xdgmime ${XDGMIME_SOURCES})
    target_compile_definitions(xdgmime PRIVATE -DXDG_PREFIX=_gio_xdg)
    list(APPEND GLIB_TARGETS xdgmime)
endif()

if(NOT WIN32 AND NOT APPLE)
    file(GLOB INOTIFY_SOURCES gio/inotify/*.c)
    add_library(inotify ${INOTIFY_SOURCES})
    target_link_libraries(inotify PRIVATE gmodule)
    target_compile_definitions(inotify PRIVATE -DG_LOG_DOMAIN=\"GLib-GIO\" -DGIO_COMPILATION -DG_DISABLE_DEPRECATED)
    list(APPEND GLIB_TARGETS inotify)
endif()

if(APPLE)
    file(GLOB KQUEUE_SOURCES gio/kqueue/*.c)
    add_library(kqueue ${KQUEUE_SOURCES})
    target_link_libraries(kqueue PRIVATE gmodule)
    target_compile_definitions(kqueue PRIVATE -DG_LOG_DOMAIN=\"GLib-GIO\" -DGIO_COMPILATION -DG_DISABLE_DEPRECATED)
    list(APPEND GLIB_TARGETS kqueue)
endif()

# gio
set(GIO_SOURCES
    ./gio/gappinfo.c
    ./gio/gasynchelper.c
    ./gio/gasyncinitable.c
    ./gio/gasyncresult.c
    ./gio/gbufferedinputstream.c
    ./gio/gbufferedoutputstream.c
    ./gio/gbytesicon.c
    ./gio/gcancellable.c
    ./gio/gcharsetconverter.c
    ./gio/gcontextspecificgroup.c
    ./gio/gconverter.c
    ./gio/gconverterinputstream.c
    ./gio/gconverteroutputstream.c
    ./gio/gcredentials.c
    ./gio/gdatagrambased.c
    ./gio/gdatainputstream.c
    ./gio/gdataoutputstream.c
    ./gio/gdrive.c
    ./gio/gdummyfile.c
    ./gio/gdummyproxyresolver.c
    ./gio/gdummytlsbackend.c
    ./gio/gemblem.c
    ./gio/gemblemedicon.c
    ./gio/gfile.c
    ./gio/gfileattribute.c
    ./gio/gfileenumerator.c
    ./gio/gfileicon.c
    ./gio/gfileinfo.c
    ./gio/gfileinputstream.c
    ./gio/gfilemonitor.c
    ./gio/gfilenamecompleter.c
    ./gio/gfileoutputstream.c
    ./gio/gfileiostream.c
    ./gio/gfilterinputstream.c
    ./gio/gfilteroutputstream.c
    ./gio/gicon.c
    ./gio/ginetaddress.c
    ./gio/ginetaddressmask.c
    ./gio/ginetsocketaddress.c
    ./gio/ginitable.c
    ./gio/ginputstream.c
    ./gio/gioerror.c
    ./gio/giomodule.c
    ./gio/gioscheduler.c
    ./gio/giostream.c
    ./gio/gloadableicon.c
    ./gio/gmount.c
    ./gio/gmemoryinputstream.c
    ./gio/gmemoryoutputstream.c
    ./gio/gmountoperation.c
    ./gio/gnativevolumemonitor.c
    ./gio/gnativesocketaddress.c
    ./gio/gnetworkaddress.c
    ./gio/gnetworking.c
    ./gio/gnetworkmonitor.c
    ./gio/gnetworkmonitorbase.c
    ./gio/gnetworkservice.c
    ./gio/goutputstream.c
    ./gio/gpermission.c
    ./gio/gpollableinputstream.c
    ./gio/gpollableoutputstream.c
    ./gio/gpollableutils.c
    ./gio/gpollfilemonitor.c
    ./gio/gproxy.c
    ./gio/gproxyaddress.c
    ./gio/gproxyaddressenumerator.c
    ./gio/gproxyresolver.c
    ./gio/gresolver.c
    ./gio/gresource.c
    ./gio/gresourcefile.c
    ./gio/gseekable.c
    ./gio/gsimpleasyncresult.c
    ./gio/gsimpleiostream.c
    ./gio/gsimplepermission.c
    ./gio/gsocket.c
    ./gio/gsocketaddress.c
    ./gio/gsocketaddressenumerator.c
    ./gio/gsocketclient.c
    ./gio/gsocketconnectable.c
    ./gio/gsocketconnection.c
    ./gio/gsocketcontrolmessage.c
    ./gio/gsocketinputstream.c
    ./gio/gsocketlistener.c
    ./gio/gsocketoutputstream.c
    ./gio/gsubprocesslauncher.c
    ./gio/gsubprocess.c
    ./gio/gsocketservice.c
    ./gio/gsrvtarget.c
    ./gio/gsimpleproxyresolver.c
    ./gio/gtask.c
    ./gio/gtcpconnection.c
    ./gio/gtcpwrapperconnection.c
    ./gio/gthreadedsocketservice.c
    ./gio/gthemedicon.c
    ./gio/gthreadedresolver.c
    ./gio/gtlsbackend.c
    ./gio/gtlscertificate.c
    ./gio/gtlsclientconnection.c
    ./gio/gtlsconnection.c
    ./gio/gtlsdatabase.c
    ./gio/gtlsfiledatabase.c
    ./gio/gtlsinteraction.c
    ./gio/gtlspassword.c
    ./gio/gtlsserverconnection.c
    ./gio/gdtlsconnection.c
    ./gio/gdtlsclientconnection.c
    ./gio/gdtlsserverconnection.c
    ./gio/gunionvolumemonitor.c
    ./gio/gvfs.c
    ./gio/gvolume.c
    ./gio/gvolumemonitor.c
    ./gio/gzlibcompressor.c
    ./gio/gzlibdecompressor.c
    ./gio/gioenumtypes.c
    ./gio/glistmodel.c
    ./gio/gliststore.c
    ./gio/gapplication.c
    ./gio/gapplicationcommandline.c
    ./gio/gapplicationimpl-dbus.c
    ./gio/gactiongroup.c
    ./gio/gactionmap.c
    ./gio/gsimpleactiongroup.c
    ./gio/gremoteactiongroup.c
    ./gio/gactiongroupexporter.c
    ./gio/gdbusactiongroup.c
    ./gio/gaction.c
    ./gio/gpropertyaction.c
    ./gio/gsimpleaction.c
    ./gio/gmenumodel.c
    ./gio/gmenu.c
    ./gio/gmenuexporter.c
    ./gio/gdbusmenumodel.c
    ./gio/gnotification.c
    ./gio/gnotificationbackend.c
    ./gio/gdbusutils.c
    ./gio/gdbusaddress.c
    ./gio/gdbusauthobserver.c
    ./gio/gdbusauth.c
    ./gio/gdbusauthmechanism.c
    ./gio/gdbusauthmechanismanon.c
    ./gio/gdbusauthmechanismexternal.c
    ./gio/gdbusauthmechanismsha1.c
    ./gio/gdbuserror.c
    ./gio/gdbusconnection.c
    ./gio/gdbusmessage.c
    ./gio/gdbusnameowning.c
    ./gio/gdbusnamewatching.c
    ./gio/gdbusproxy.c
    ./gio/gdbusprivate.c
    ./gio/gdbusintrospection.c
    ./gio/gdbusmethodinvocation.c
    ./gio/gdbusserver.c
    ./gio/gdbusinterface.c
    ./gio/gdbusinterfaceskeleton.c
    ./gio/gdbusobject.c
    ./gio/gdbusobjectskeleton.c
    ./gio/gdbusobjectproxy.c
    ./gio/gdbusobjectmanager.c
    ./gio/gdbusobjectmanagerclient.c
    ./gio/gdbusobjectmanagerserver.c
    ./gio/gtestdbus.c
    ./gio/ghttpproxy.c
    ./gio/glocalfile.c
    ./gio/glocalfileenumerator.c
    ./gio/glocalfileinfo.c
    ./gio/glocalfileinputstream.c
    ./gio/glocalfilemonitor.c
    ./gio/glocalfileoutputstream.c
    ./gio/glocalfileiostream.c
    ./gio/glocalvfs.c
    ./gio/gsocks4proxy.c
    ./gio/gsocks4aproxy.c
    ./gio/gsocks5proxy.c
    ./gio/thumbnail-verify.c
    ./gio/gdbusdaemon.c
    ./gio/gdbus-daemon-generated.c
    ./gio/gwin32registrykey.c
    ./gio/gwin32mount.c
    ./gio/gwin32volumemonitor.c
    ./gio/gwin32inputstream.c
    ./gio/gwin32outputstream.c
    ./gio/gwin32appinfo.c
    ./gio/gcontenttype-win32.c
    ./gio/gregistrysettingsbackend.c
    ./gio/win32/gwinhttpfile.c
    ./gio/win32/gwinhttpfileinputstream.c
    ./gio/win32/gwinhttpfileoutputstream.c
    ./gio/win32/gwinhttpvfs.c
    ./gio/win32/gwin32fsmonitorutils.c
    ./gio/win32/gwin32filemonitor.c
    ./gio/gvdb/gvdb-reader.c
    ./gio/gdelayedsettingsbackend.c
    ./gio/gkeyfilesettingsbackend.c
    ./gio/gmemorysettingsbackend.c
    ./gio/gnullsettingsbackend.c
    ./gio/gsettingsbackend.c
    ./gio/gsettingsschema.c
    ./gio/gsettings-mapping.c
    ./gio/gsettings.c
)

if(NOT WIN32)
    file(GLOB GIO_UNIX_SOURCES "gio/gunix*.c" "gio/g*notificationbackend.c" "gio/g*portal*.c")
    list(APPEND GIO_SOURCES ${GIO_UNIX_SOURCES})
    list(APPEND GIO_SOURCES
        "gio/gcontenttype.c"
        "gio/gfiledescriptorbased.c"
        "gio/gnetworkmonitornm.c"
        "gio/xdp-dbus.c"
    )
    list(FILTER GIO_SOURCES EXCLUDE REGEX "/gwin32[^/]+\$|win32/[^/]+\$|win32.c\$|gregistrysettingsbackend.c\$")
    if(APPLE)
        set_property(SOURCE
            gio/gcocoanotificationbackend.c
            gio/gosxappinfo.c
            gio/gnextstepsettingsbackend.c
            PROPERTY COMPILE_FLAGS "-x objective-c")
        list(APPEND GIO_SOURCES
            "gio/gnextstepsettingsbackend.c"
            "gio/gosxappinfo.c"
        )
    else()
        list(APPEND GIO_SOURCES
            "gio/gnetworkmonitornetlink.c"
            "gio/gdesktopappinfo.c"
        )
        list(FILTER GIO_SOURCES EXCLUDE REGEX "gcocoanotificationbackend.c\$")
    endif()
endif()
if(NOT GLIB_SKIP_HEADERS)
    file(GLOB GIO_HEADERS gio/*.h)
    list(FILTER GIO_HEADERS EXCLUDE REGEX "private.h\$")
    install(FILES ${GIO_HEADERS} ${CMAKE_BINARY_DIR}/config/gio/gnetworking.h DESTINATION include/gio)
endif()
add_library(gio ${GIO_SOURCES})
target_compile_definitions(gio PRIVATE GIO_COMPILATION G_LOG_DOMAIN="GLib-GIO")
target_link_libraries(gio PRIVATE glib gmodule gobject ZLIB::ZLIB ${LIBRESOLV_LIBRARY} ${LIBINTL_LIBRARY})
target_include_directories(gio PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/gio> $<INSTALL_INTERFACE:include/gio>)
if(WIN32)
    target_link_libraries(gio PRIVATE ws2_32 shlwapi dnsapi iphlpapi advapi32 shell32)
elseif(APPLE)
    target_link_libraries(gio PRIVATE xdgmime kqueue)
else()
    target_link_libraries(gio PRIVATE xdgmime inotify)
endif()
list(APPEND GLIB_TARGETS gio)

foreach(GTARGET ${GLIB_TARGETS})
    set_target_properties(${GTARGET} PROPERTIES
        OUTPUT_NAME ${GTARGET}-${GLIB_DLL_SUFFIX}
        ARCHIVE_OUTPUT_NAME ${GTARGET}-${GLIB_LIB_SUFFIX})
endforeach()

macro(add_glib_tool TOOL_NAME)
    add_executable(${TOOL_NAME} ${ARGN})
    if(WIN32)
        target_link_libraries(${TOOL_NAME} glib ${LIBINTL_LIBRARY} shell32)
    else()
        target_link_libraries(${TOOL_NAME} glib ${LIBINTL_LIBRARY})
    endif()
    target_compile_definitions(${TOOL_NAME} PRIVATE GLIB_COMPILATION)
    list(APPEND GLIB_TOOLS ${TOOL_NAME})
endmacro()

macro(add_gio_tool TOOL_NAME)
    add_executable(${TOOL_NAME} ${ARGN})
    target_link_libraries(${TOOL_NAME} PRIVATE glib gio gobject gmodule ${LIBINTL_LIBRARY})
    target_include_directories(${TOOL_NAME} PRIVATE gmodule gio)
    target_compile_definitions(${TOOL_NAME} PRIVATE GIO_COMPILATION)
    list(APPEND GLIB_TOOLS ${TOOL_NAME})
endmacro()

if(NOT GLIB_SKIP_TOOLS)
    configure_file(gobject/glib-mkenums.in ${CMAKE_SOURCE_DIR}/gobject/glib-mkenums @ONLY) # uses GLIB_VERSION
    install(FILES gobject/glib-mkenums DESTINATION tools/glib)

    configure_file(gio/gdbus-2.0/codegen/gdbus-codegen.in ${CMAKE_SOURCE_DIR}/gio/gdbus-2.0/codegen/gdbus-codegen COPYONLY)
    install(FILES gio/gdbus-2.0/codegen/gdbus-codegen DESTINATION tools/glib)
    file(GLOB CODEGEN_SOURCES gio/gdbus-2.0/codegen/*.py)
    install(FILES ${CODEGEN_SOURCES} DESTINATION tools/glib/codegen)

    add_gio_tool(gdbus gio/gdbus-tool.c)
    add_gio_tool(gio-querymodules gio/gio-querymodules.c)
    file(GLOB GIO_TOOL_SOURCES gio/gio-tool*.c)
    add_gio_tool(gio-tool ${GIO_TOOL_SOURCES})
    set_target_properties(gio-tool PROPERTIES OUTPUT_NAME gio)
    add_gio_tool(glib-compile-resources gio/glib-compile-resources.c gio/gvdb/gvdb-builder.c)
    add_gio_tool(glib-compile-schemas gio/glib-compile-schemas.c gio/gvdb/gvdb-builder.c)
    add_gio_tool(gresource gio/gresource-tool.c)
    add_gio_tool(gsettings gio/gsettings-tool.c)

    if(CMAKE_SIZEOF_VOID_P EQUAL 4)
        set(WIN win32)
    else()
        set(WIN win64)
    endif()

    add_glib_tool(glib-genmarshal gobject/glib-genmarshal.c)
    if(WIN32)
        add_glib_tool(gspawn-${WIN}-helper WIN32 glib/gspawn-win32-helper.c)
        add_glib_tool(gspawn-${WIN}-helper-console glib/gspawn-win32-helper-console.c)
    endif()

    install(TARGETS ${GLIB_TOOLS} RUNTIME DESTINATION tools/glib)
endif()

install(
    TARGETS ${GLIB_TARGETS}
    EXPORT glib
    RUNTIME DESTINATION bin
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
)
install(
    EXPORT glib
    NAMESPACE unofficial::glib::
    FILE unofficial-glib-targets.cmake
    DESTINATION share/unofficial-glib
)
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/unofficial-glib-config.in.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/cmake/unofficial-glib-config.cmake
    @ONLY
)
install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/cmake/unofficial-glib-config.cmake
    DESTINATION share/unofficial-glib
)

message(STATUS "Link-time dependencies:")
message(STATUS "  " ${ZLIB_LIBRARIES})
message(STATUS "  " ${PCRE_LIBRARY})
message(STATUS "  " ${ICONV_LIBRARY})
message(STATUS "  " ${CHARSET_LIBRARY})
message(STATUS "  " ${FFI_LIBRARY})
message(STATUS "  " ${LIBINTL_LIBRARY})
