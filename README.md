# Notes about MAKE

## Append text to a variable

You have to use `:=`, instead of `=`:

	VAR := hello
	VAR := $(VAR)_world

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

## Test if a command line parameter is defined

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
	# Replace "$(LOCAL_SRC_DIRECTORY)" by "$(LOCAL_LIB_DIRECTORY)" (=> the ".o" files will be created into the directory "$(LOCAL_LIB_DIRECTORY)"). 
	LIB_SRC = $(subst $(LOCAL_SRC_DIRECTORY),$(LOCAL_LIB_DIRECTORY),$(patsubst %.c,%.o,$(wildcard ${LOCAL_SRC_DIRECTORY}/*.c)))

	# Build all "*.o" files from all "*.c" files.
	$(LOCAL_LIB_DIRECTORY)/%.o: $(LOCAL_SRC_DIRECTORY)/%.c
		$(CC) $(CFLAGS) -c -o $@ $<

	# Build the library from the list of "*.o" files.
	$(LOCAL_LIB_DIRECTORY)/libfunc.a: $(LIB_SRC)
		$(AR) $(ARFLAGS) $@ $^

## Create a header file at build time

Scenario: you want to insert the date of the compilation into the executable you are building.

One solution is to produce a header file that defines a constant which represents the date.
For example, we can think of the header file `src/data.h`:

    #ifndef DATE
    #define DATE "2020-6-11 10:11:22"
    #endif

This header file gets included in all executables sources. And these codes use the constant `DATE`.

To produce the header `src/date.h` we use a programme.
For example, we can think of the program `src/date.c`:

    #include <stdio.h>
    #include <time.h>
    
    int main(int argc, char *argv[])
    {
        if (2 != argc) {
            printf("Usage: %s <output file>\n", argv[0]);
            return 1;
        }
    
        time_t t = time(NULL);
        struct tm tm = *localtime(&t);
    
        FILE *fd = fopen(argv[1], "w");
        if (NULL == fd) {
            fprintf(stderr, "Cannot open the file <%s> for writing.\n", argv[1]);
            return 1;
        }
        fprintf(fd, "#ifndef DATE\n");
        fprintf(fd,"#define DATE \"%d-%02d-%02d %02d:%02d:%02d\"\n", tm.tm_year + 1900, tm.tm_mon + 1, tm.tm_mday, tm.tm_hour, tm.tm_min, tm.tm_sec);
        fprintf(fd, "#endif\n");
        fclose(fd);
        return 0;
    }

So, we need to:

* Compile `src/data.c` into `bin/date.exe`.
* Run `bin/date.exe src/date.h`.
* Compile all the executables that include `date.h`.

Declare the target `bin/date.exe`.

    bin/date.exe: src/date.c
	    ${CC} -Wall -o $@ $^

Declare the target `src/date.h`. We want the recipe for this target to be executed
**unconditionally** whenever `make` hits a target that depends on it.

See [https://www.gnu.org/software/make/manual/html_node/Force-Targets.html](https://www.gnu.org/software/make/manual/html_node/Force-Targets.html):

> If a rule has no prerequisites or recipe, and the target of the rule is a
> nonexistent file, then make imagines this target to have been updated
> whenever its rule is run. This implies that all targets depending on this
> one will always have their recipe run.
>
> => The file "src/date.h" will be (re)created every time "make" encounters a target that depends on "src/date.h".
    
    src/date.h: FORCE bin/date.exe
        $(info "Create date.h")
        bin/date.exe src/date.h
    
    FORCE:

Declare the target `bin/prog.exe`.

    bin/prog.exe: src/source1.c \
        src/source2.c \
        src/date.h
        ${CC} -Wall -o $@ $^

Please not that this target depends on `src/date.h`. `src/date.h` is (re)generated unconditionally.
Thus `bin/prog.exe` is (re)compiled unconditionally.
