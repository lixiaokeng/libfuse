project('libfuse3', ['c'], version: '3.10.4',
        meson_version: '>= 0.42',
        default_options: [
            'buildtype=debugoptimized',
            'cpp_std=c++11',
            'warning_level=2',
        ])


platform = host_machine.system()
if platform == 'darwin'
  error('libfuse does not support OS-X.\n' +
        'Take a look at http://osxfuse.github.io/ instead')
elif platform == 'cygwin' or platform == 'windows'
  error('libfuse does not support Windows.\n' +
        'Take a look at http://www.secfs.net/winfsp/ instead')       
endif

#
# Feature detection
#
cfg = configuration_data()
cc = meson.get_compiler('c')

# Default includes when checking for presence of functions and
# struct members
include_default = '''
#include <stdio.h>
#include <stdlib.h>
#include <stddef.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
'''
args_default = [ '-D_GNU_SOURCE' ]

cfg.set_quoted('PACKAGE_VERSION', meson.project_version())

# Test for presence of some functions
test_funcs = [ 'fork', 'fstatat', 'openat', 'readlinkat', 'pipe2',
               'splice', 'vmsplice', 'posix_fallocate', 'fdatasync',
               'utimensat', 'copy_file_range', 'fallocate' ]
foreach func : test_funcs
    cfg.set('HAVE_' + func.to_upper(),
        cc.has_function(func, prefix: include_default, args: args_default))
endforeach
cfg.set('HAVE_SETXATTR', 
        cc.has_function('setxattr', prefix: '#include <sys/xattr.h>'))
cfg.set('HAVE_ICONV', 
        cc.has_function('iconv', prefix: '#include <iconv.h>'))

# Test if structs have specific member
cfg.set('HAVE_STRUCT_STAT_ST_ATIM',
         cc.has_member('struct stat', 'st_atim',
                       prefix: include_default,
                       args: args_default))
cfg.set('HAVE_STRUCT_STAT_ST_ATIMESPEC',
         cc.has_member('struct stat', 'st_atimespec',
                       prefix: include_default,
                       args: args_default))

# Write the test results into config.h (stored in build directory)
configure_file(output: 'config.h',
               configuration : cfg)

#
# Compiler configuration
#
add_project_arguments('-D_REENTRANT', '-DHAVE_CONFIG_H', '-Wno-sign-compare',
                      '-Wstrict-prototypes', '-Wmissing-declarations', '-Wwrite-strings',
                      '-fno-strict-aliasing', language: 'c')
add_project_arguments('-D_REENTRANT', '-DHAVE_CONFIG_H', '-D_GNU_SOURCE',
                     '-Wno-sign-compare', '-Wmissing-declarations',
                     '-Wwrite-strings', '-fno-strict-aliasing', language: 'cpp')

# Some (stupid) GCC versions warn about unused return values even when they are
# casted to void. This makes -Wunused-result pretty useless, since there is no
# way to suppress the warning when we really *want* to ignore the value.
code = '''
__attribute__((warn_unused_result)) int get_4() {
    return 4;
}
int main(void) {
    (void) get_4();
    return 0;
}'''
if not cc.compiles(code, args: [ '-O0', '-Werror=unused-result' ])
     message('Compiler warns about unused result even when casting to void')
     add_project_arguments('-Wno-unused-result', language: 'c')
endif

# gcc-10 and newer support the symver attribute which we need to use if we
# want to support LTO
# recent clang and gcc both support __has_attribute (and if they are too old
# to have __has_attribute, then they are too old to support symver)
# other compilers might not have __has_attribute, but in those cases
# it is safe for this check to fail and for us to fallback to the old _asm_
# method for symver. Anyway the attributes not supported by __has_attribute()
# unfortunately return true giving a false positive. So let's try to build
# using __attribute__ ((symver )) and see the result.
code = '''
__attribute__ ((symver ("test@TEST")))
void foo(void) {
}

int main(void) {
    return 0;
}'''
if cc.compiles(code, args: [ '-O0', '-c', '-Werror'])
     message('Compiler supports symver attribute')
     add_project_arguments('-DHAVE_SYMVER_ATTRIBUTE', language: 'c')
else
     message('Compiler does not support symver attribute')
endif

# '.' will refer to current build directory, which contains config.h
include_dirs = include_directories('include', 'lib', '.')

# Common dependencies
thread_dep = dependency('threads') 

#
# Read build files from sub-directories
#
subdirs = [ 'lib', 'include']
if get_option('utils') and not platform.endswith('bsd') and platform != 'dragonfly'
  subdirs += [ 'util', 'doc' ]
endif

if get_option('examples')
  subdirs += 'example'
endif

if get_option('tests')
  subdirs += 'test'
endif

foreach n : subdirs
    subdir(n)
endforeach
