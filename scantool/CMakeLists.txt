
#Based on the L0 and L2 options selected (see root CMakeLists.txt)
#generate the "zone" string lists for diag_config.c.in, and
#select appropriate source files

#generate zone 1 and 3 lines with L0 names
foreach (L0NAME IN LISTS L0LIST)
	if (USE_L0_${L0NAME})
		set (DIAG_CONFIG_ZONE1 "${DIAG_CONFIG_ZONE1}extern const struct diag_l0 diag_l0_${L0NAME};\n")
		set (DIAG_CONFIG_ZONE3 "${DIAG_CONFIG_ZONE3}\t&diag_l0_${L0NAME},\n")
		set (DL0_SRCS ${DL0_SRCS} "diag_l0_${L0NAME}.c")
		message(STATUS "Adding l0 driver ${L0NAME}")
	endif()
endforeach()

#generate zone 2 and 4 lines with L2 names
foreach (L2NAME IN LISTS L2LIST)
	if (USE_L2_${L2NAME})
		set (DIAG_CONFIG_ZONE2 "${DIAG_CONFIG_ZONE2}extern const struct diag_l2_proto diag_l2_proto_${L2NAME};\n")
		set (DIAG_CONFIG_ZONE4 "${DIAG_CONFIG_ZONE4}\t&diag_l2_proto_${L2NAME},\n")
		set (DL2_SRCS ${DL2_SRCS} "diag_l2_${L2NAME}.c")
		message(STATUS "Adding l2 driver ${L2NAME}")
	endif()
endforeach()

#integrate cppcheck checks if the binary is present.
find_program (CPPCHECK_BIN NAMES cppcheck)
if (NOT (CPPCHECK_BIN MATCHES "NOTFOUND"))
       # cppcheck is found in the path so use it.
       find_program (CMAKE_C_CPPCHECK NAMES cppcheck)
       list (
        APPEND CMAKE_C_CPPCHECK
            "--enable=warning"
			"--enable=style"
			"--enable=performance"
			"--enable=information"
            "--suppress=missingInclude"
			"--suppress=missingSystemInclude"
			"--suppress=unmatchedSuppression"
			"--suppress=variableScope"
            "--error-exitcode=1"
            "--inline-suppr"
       )
	   if (WIN32)
	      list (
             APPEND CMAKE_C_CPPCHECK 
             "--platform=win32A"
			 "-D_WIN32"
			 #
			 # This is to prevent cppcheck to include windows.h.
			 # The header leads to problems with the borland compilers.
			 #
			 "-D_WINDOWS_"
		  )
	   else ()
	      list (
		     APPEND CMAKE_C_CPPCHECK
		     "--platform=unix32"
		     "-D__unix__"
	      )
	   endif ()
endif ()

#and now generate diag_config.c ! (output in the build directory)
configure_file ( diag_config.c.in diag_config.c)


### select conditional source files

if(WIN32)
	set (OS_DIAGTTY "diag_tty_win.c")
	set (OS_DIAGOS "diag_os_win.c")
	set (FREEDIAG_RC "freediag.rc")
	set (DIAG_TEST_RC "diag_test.rc")
	enable_language (RC)
else()
	set (OS_DIAGTTY "diag_tty_unix.c")
	set (OS_DIAGOS "diag_os_unix.c")
	set (FREEDIAG_RC "")
	set (DIAG_TEST_RC "")
endif()


### build source file lists (useful later)
set (LIBDIAG_SRCS ${DL0_SRCS} ${DL2_SRCS}
	${CMAKE_CURRENT_BINARY_DIR}/diag_config.c
	${OS_DIAGTTY} ${OS_DIAGOS}
	diag_l0.c diag_l1.c diag_l2.c diag_l3.c
	diag_l3_saej1979.c diag_l3_iso14230.c diag_l3_vag.c
	diag_l7_d2.c diag_l7_kwp71.c
	diag_general.c diag_dtc.c diag_cfg.c)
set (LIBDYNO_SRCS dyno.c)
set (DIAGTEST_SRCS diag_test.c ${DIAG_TEST_RC})
set (CLI_SRCS scantool_cli.c scantool_diag.c scantool_set.c
	scantool_debug.c)
set (SCANTOOL_SRCS scantool.c
	scantool_test.c scantool_vag scantool_850.c scantool_dyno.c
	scantool_obd.c scantool_aif.c ${FREEDIAG_RC})

#and GLOB all the headers. This is *only* so that the headers end up in
#the file list of IDE projects (at least Code::blocks)
#but could be removed without impacting the build process
file (GLOB SCANTOOL_HEADERS *.h)



### set CURFILE
#for the various debugging fprintf()s, __FILE__ gives a super long path
#but we're really only interested in the filename; the following defines
# CURFILE=xyz for each source file.
# See http://stackoverflow.com/questions/237542/learning-the
# -source-codes-filename-at-compile-time/22161316

foreach (F IN LISTS LIBDIAG_SRCS;LIBDYNO_SRCS;
	DIAGTEST_SRCS;CLI_SRCS;SCANTOOL_SRCS)
	get_filename_component (BNAME ${F} NAME)
	set_source_files_properties (${F} PROPERTIES
		COMPILE_DEFINITIONS "CURFILE=\"${BNAME}\"")
endforeach()


### Now define targets.

# libdiag and libdyno (required for binaries)

add_library(diag STATIC ${LIBDIAG_SRCS})

if (NOT WIN32)
	#link to libmath (m); not required on win32 (msvcrt provides sin() etc)
	#link to pthread (win* uses native API)
	target_link_libraries(diag m ${CMAKE_THREAD_LIBS_INIT})
else()
	# WIN needs winmm for timeBeginPeriod()
	target_link_libraries(diag winmm)
endif ()

#if required, link with -lrt (see top-level CMakeLists.txt)
target_link_libraries(diag ${LIBRT})

add_library(dyno STATIC ${LIBDYNO_SRCS})

add_library(freediagcli STATIC ${CLI_SRCS})


# diag_test binary
if (BUILD_DIAGTEST)
	add_executable(${DIAG_TEST_PROGNAME} ${DIAGTEST_SRCS})
	target_link_libraries(diag_test diag)
	install(TARGETS diag_test DESTINATION ${BIN_DESTDIR})
endif ()

# freediag binary

add_executable(${SCANTOOL_PROGNAME}  ${SCANTOOL_SRCS} ${SCANTOOL_HEADERS})

target_link_libraries(freediagcli diag)
if (HAVE_LIBREADLINE)
	#this was optional
	target_link_libraries(freediagcli readline)
endif ()

target_link_libraries(freediag dyno freediagcli)

install(TARGETS freediag DESTINATION ${BIN_DESTDIR})


### misc install & copy targets

#install carsim .db files and sample .ini file
file (GLOB CARSIM_EXAMPLES *.db)
install (FILES ${CARSIM_EXAMPLES} DESTINATION ${EXAMPLES_DESTDIR})

if (USE_INIFILE)
	install (FILES freediag.ini DESTINATION ${EXAMPLES_DESTDIR})
endif ()
