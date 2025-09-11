set(cmnrunsrc
)

set(runsrc_extra
)

add_library(tr32h STATIC
	${cmnrunsrc}
	${runsrc_extra}
	run.c
    trd.c
	
	bifgdum.c
	osnoui.c
	osstzprs.c
)


target_compile_definitions(tr32h PRIVATE
	DBG_OFF
	HTMLTADS
)

target_include_directories(tr32h PRIVATE
	${CMAKE_CURRENT_SOURCE_DIR}
)

if (WIN32)
	target_sources(tr32h PRIVATE
		msdos/oswin.c
        msdos/osdosnui.c
		msdos/ostzw32.c
	)
	target_include_directories(tr32h PRIVATE
		${CMAKE_CURRENT_SOURCE_DIR}/msdos
	)
	target_compile_definitions(tr32h PRIVATE
		T_WIN32
		MICROSOFT
	)
else()
	target_include_directories(tr32h PRIVATE
		${CMAKE_CURRENT_SOURCE_DIR}/unix
	)
	target_compile_definitions(tr32h PRIVATE
		UNIX
		LINUX_386
		OSANSI
	)
endif()

add_library(Tads::tr32h ALIAS tr32h)
