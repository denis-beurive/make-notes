# Notes about MAKE

## $@, $<, $^...

	* `$@`: is the name of the file being generated.
	* `$<`: is the first prerequisite.
	* `$^`: is the list of all prerequisites.

Examples:

	argtable3.o: src/argtable3.c src/argtable3.h
		gcc -Wall -c -o $@ $<

	module.o: src/depend1.c src/depend2.c
		gcc -Wall -c -o $@ $^

## Get the path to the Makefile directory

	_DIR_ = $(dir $(realpath $(firstword $(MAKEFILE_LIST))))
	__DIR__ = $(_DIR_:/=)

## Test that a command line parameter is defined

	ifndef OS
	$(error "OS is not set! Usage: make OS=(mac|linux) <rule> ")
	endif

> The paremeter `OS` is set through the commend line: `make OS=...`.

## Test if an environment variable is set

	ifeq ("$(MODULE_INCLUDE_PATH)","")
	$(warning "**WARNING** you may need to set the environment MODULE_INCLUDE_PATH!")
	else
	INCLUDE_DIR := ${INCLUDE_DIR} -I$(MODULE_INCLUDE_PATH)
	endif

	ifeq ("$(MODULE_LIBRARY_PATH)","")
	$(warning "**WARNING** you may need to set the environment MODULE_LIBRARY_PATH!")
	else
	LIB_DIR := ${LIB_DIR} -L$(MODULE_LIBRARY_PATH)
	endif

## Conditional settings

	ifeq ($(OS),mac)
		CDEF =
		CC = gcc
	else
		CDEF = -DUSE_MTRACE
		CC = c99
	endif

> Test if `OS` is "`mac`" then ... else ...

## Get the list of all ".c" and build the list of all associated ".o"

	# Build the list of all associated ".o" files.
	LIB_SRC = $(subst $(LOCAL_SRC_DIRECTORY),$(LOCAL_LIB_DIRECTORY),$(patsubst %.c,%.o,$(wildcard ${LOCAL_SRC_DIRECTORY}/*.c)))

	# Build all "*.o" files from all "*.c" files.
	$(LOCAL_LIB_DIRECTORY)/%.o: $(LOCAL_SRC_DIRECTORY)/%.c
		$(CC) $(CFLAGS) -c -o $@ $<

	# Build the library from the list of "*.o" files.
	$(LOCAL_LIB_DIRECTORY)/libfunc.a: $(LIB_SRC)
		$(AR) $(ARFLAGS) $@ $^
