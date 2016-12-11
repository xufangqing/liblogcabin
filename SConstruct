from distutils.version import LooseVersion as Version
import fnmatch
import re
import sys
import os
import shutil
import subprocess

# Python 2.6 doesn't have subprocess.check_output
try:
    subprocess.check_output
except AttributeError:
    def check_output_compat(*popenargs, **kwargs):
        # This function was copied from Python 2.7's subprocess module.
        # This function only is:
        # Copyright (c) 2003-2005 by Peter Astrand <astrand@lysator.liu.se>
        # Licensed to PSF under a Contributor Agreement.
        # See http://www.python.org/2.4/license for licensing details.
        if 'stdout' in kwargs:
            raise ValueError('stdout argument not allowed, it will be overridden.')
        process = subprocess.Popen(stdout=subprocess.PIPE, *popenargs, **kwargs)
        output, unused_err = process.communicate()
        retcode = process.poll()
        if retcode:
            cmd = kwargs.get("args")
            if cmd is None:
                cmd = popenargs[0]
            raise subprocess.CalledProcessError(retcode, cmd, output=output)
        return output
    subprocess.check_output = check_output_compat

_VERSION = '0.0.1-alpha.0'

opts = Variables('Local.sc')

opts.AddVariables(
    ("CC", "C Compiler"),
    ("CPPPATH", "The list of directories that the C preprocessor "
                "will search for include directories", []),
    ("CXX", "C++ Compiler"),
    ("CXX_FAMILY", "C++ compiler family (gcc or clang)", "auto"),
    ("CXX_VERSION", "C++ compiler version", "auto"),
    ("CXXFLAGS", "Options that are passed to the C++ compiler", []),
    ("PROTOCXXFLAGS", "Options that are passed to the C++ compiler "
                      "for ProtoBuf files", []),
    ("GTESTCXXFLAGS", "Options that are passed to the C++ compiler "
                      "for compiling gtest", []),
    ("LINKFLAGS", "Options that are passed to the linker", []),
    ("AS", "Assembler"),
    ("LIBPATH", "Library paths that are passed to the linker", []),
    ("LINK", "Linker"),
    ("BUILDTYPE", "Build type (RELEASE or DEBUG)", "DEBUG"),
    ("VERBOSE", "Show full build information (0 or 1)", "0"),
    ("NUMCPUS", "Number of CPUs to use for build (0 means auto).", "0"),
    ("VERSION", "Override version string", _VERSION)
)

env = Environment(options = opts,
                  tools = ['default', 'protoc', 'packaging'],
                  ENV = os.environ)
Help(opts.GenerateHelpText(env))

# Needed for Clang Static Analyzer's scan-build tool
env["CC"] = os.getenv("CC") or env["CC"]
env["CXX"] = os.getenv("CXX") or env["CXX"]
for k, v in os.environ.items():
    if k.startswith("CCC_"):
        env["ENV"][k] = v

def detect_compiler():
    reflags = re.IGNORECASE|re.MULTILINE
    output = subprocess.check_output([env['CXX'], '-v'],
                                     stderr=subprocess.STDOUT)
    m = re.search(r'gcc version (\d+\.\d+\.\d+)',
                  output, reflags)
    if m is not None:
        env['CXX_FAMILY'] = 'gcc'
        env['CXX_VERSION'] = m.group(1)
        return
    m = re.search(r'clang version (\d+\.\d+\.\d+)',
                  output, reflags)
    if m is not None:
        env['CXX_FAMILY'] = 'clang'
        env['CXX_VERSION'] = m.group(1)
        return

if env['CXX_FAMILY'].lower() == 'auto':
    try:
        detect_compiler()
        print 'Detected compiler %s %s' % (env['CXX_FAMILY'],
                                           env['CXX_VERSION'])
    except BaseException as e:
        print 'Could not detect compiler: %s' % e
        pass

CXX_STANDARD = 'c++11'

if (env['CXX_FAMILY'] == 'gcc' and
    Version(env['CXX_VERSION']) < Version('4.7')):
    CXX_STANDARD = 'c++0x'

if env['CXX_FAMILY'] == 'gcc':
    env.Prepend(CXXFLAGS = [
        "-Wall",
        "-Wextra",
        "-Wcast-align",
        "-Wcast-qual",
        "-Wconversion",
        "-Weffc++",
        "-Wformat=2",
        "-Wmissing-format-attribute",
        "-Wno-non-template-friend",
        "-Wno-unused-parameter",
        "-Woverloaded-virtual",
        "-Wwrite-strings",
        "-DSWIG", # For some unknown reason, this suppresses some definitions
                  # in headers generated by protobuf 2.6 (but not 2.5) that we
                  # don't use and that cause warnings with -Weffc++.
    ])
elif env['CXX_FAMILY'] == 'clang':
    # I couldn't find a descriptive list of warnings for clang, so it's easier
    # to enable them all with -Weverything and then disable the problematic
    # ones.
    env.Prepend(CXXFLAGS = [
        '-Wno-c++98-compat-pedantic',
        '-Wno-covered-switch-default',
        '-Wno-deprecated',
        '-Wno-disabled-macro-expansion',
        '-Wno-documentation-unknown-command',
        '-Wno-exit-time-destructors',
        '-Wno-float-equal',
        '-Wno-global-constructors',
        '-Wno-gnu-zero-variadic-macro-arguments',
        '-Wno-missing-noreturn',
        '-Wno-missing-prototypes',
        '-Wno-missing-variable-declarations',
        '-Wno-packed',
        '-Wno-padded',
        '-Wno-reserved-id-macro',
        '-Wno-shadow',
        '-Wno-shift-sign-overflow',
        '-Wno-switch-enum',
        '-Wno-undef',
        '-Wno-unknown-warning-option',
        '-Wno-unused-macros',
        '-Wno-unused-member-function',
        "-Wno-unused-parameter",
        '-Wno-used-but-marked-unused',
        '-Wno-vla',
        '-Wno-vla-extension',
        '-Wno-weak-vtables',
    ])

    # Clang 3.4 is known to emit warnings without -Wno-unreachable-code:
    if Version(env['CXX_VERSION']) < Version('3.5'):
        env.Prepend(CXXFLAGS = ['-Wno-unreachable-code'])

    env.Prepend(CXXFLAGS = ['-Weverything'])

env.Prepend(CXXFLAGS = [
    "-std=%s" % CXX_STANDARD,
    "-fno-strict-overflow",
    "-fPIC",
])
env.Prepend(PROTOCXXFLAGS = [
    "-std=%s" % CXX_STANDARD,
    "-fno-strict-overflow",
    "-fPIC",
])
env.Prepend(GTESTCXXFLAGS = [
    "-std=%s" % CXX_STANDARD,
    "-fno-strict-overflow",
    "-fPIC",
])

if env["BUILDTYPE"] == "DEBUG":
    env.Append(CPPFLAGS = [ "-g", "-DDEBUG" ])
elif env["BUILDTYPE"] == "RELEASE":
    env.Append(CPPFLAGS = [ "-DNDEBUG", "-O2" ])
else:
    print "Error BUILDTYPE must be RELEASE or DEBUG"
    sys.exit(-1)

if env["VERBOSE"] == "0":
    env["CCCOMSTR"] = "Compiling $SOURCE"
    env["CXXCOMSTR"] = "Compiling $SOURCE"
    env["SHCCCOMSTR"] = "Compiling $SOURCE"
    env["SHCXXCOMSTR"] = "Compiling $SOURCE"
    env["ARCOMSTR"] = "Creating library $TARGET"
    env["LINKCOMSTR"] = "Linking $TARGET"

env.Append(CPPPATH = '#')
env.Append(CPPPATH = '#/build')

# Define protocol buffers builder to simplify SConstruct files
def Protobuf(env, source):
    # First build the proto file
    cc = env.Protoc(os.path.splitext(source)[0] + '.pb.cc',
                    source,
                    PROTOCPROTOPATH = ["."],
                    PROTOCPYTHONOUTDIR = ".",
                    PROTOCOUTDIR = ".")[1]
    # Then build the resulting C++ file with no warnings
    return env.StaticObject(cc,
                            CXXFLAGS = env['PROTOCXXFLAGS'] + ['-Ibuild'])
env.AddMethod(Protobuf)

def CopySources(env):
    parent, cur = os.path.split(os.getcwd())
    srcDir = os.path.join(parent, "..", "..", "src", "liblogcabin", cur)
    for file in os.listdir(srcDir):
        fullPath = os.path.join(srcDir, file)
        if file != "SConscript" and (os.path.isfile(fullPath)):
           if os.path.exists(os.path.join(os.getcwd(), file)):
              os.remove(os.path.join(os.getcwd(), file))
           shutil.copy(fullPath, os.getcwd())

env.AddMethod(CopySources)

def GetNumCPUs():
    if env["NUMCPUS"] != "0":
        return int(env["NUMCPUS"])
    if os.sysconf_names.has_key("SC_NPROCESSORS_ONLN"):
        cpus = os.sysconf("SC_NPROCESSORS_ONLN")
        if isinstance(cpus, int) and cpus > 0:
            return 2*cpus
        else:
            return 2
    return 2*int(os.popen2("sysctl -n hw.ncpu")[1].read())

env.SetOption('num_jobs', GetNumCPUs())

object_files = {}
Export('object_files')

Export('env')
SConscript('src/liblogcabin/Core/SConscript', variant_dir='build/liblogcabin/Core')
SConscript('src/liblogcabin/Client/SConscript', variant_dir='build/liblogcabin/Client')
SConscript('src/liblogcabin/Event/SConscript', variant_dir='build/liblogcabin/Event')
SConscript('src/liblogcabin/Protocol/SConscript', variant_dir='build/liblogcabin/Protocol')
SConscript('src/liblogcabin/Raft/SConscript', variant_dir='build/liblogcabin/Raft')
SConscript('src/liblogcabin/RPC/SConscript', variant_dir='build/liblogcabin/RPC')
SConscript('src/liblogcabin/Storage/SConscript', variant_dir='build/liblogcabin/Storage')
SConscript('test/SConscript', variant_dir='build/test')

library = env.StaticLibrary("build/liblogcabin",
                  (object_files['Core'] +
                   object_files['Event'] +
                   object_files['Protocol'] +
                   object_files['Raft'] +
                   object_files['RPC'] +
                   object_files['Storage']))
env.Default(library)

def RecursiveGlob(pathname):
    matches = []
    for root, dirnames, filenames in os.walk(pathname):
        for filename in fnmatch.filter(filenames, '*.h'):
            matches.append(os.path.join(root, filename))
    return matches

ib = env.Alias("install-lib", env.Install("/usr/local/lib", library))
headers = RecursiveGlob("build")
hdr_inst = [env.Install(os.path.dirname('/usr/local/include/liblogcabin/'+h.replace("build/","")), h) for h in headers]
ih = env.Alias("install-headers", hdr_inst)
env.Alias("install", [ib, ih])
