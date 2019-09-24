import os
import sys
import glob
import excons
import excons.tools.python

mscver = ARGUMENTS.get("mscver", "9.0")

env = excons.MakeBaseEnv()

out_dir = excons.OutputBaseDirectory()
out_bindir = out_dir + "/bin"
out_incdir = out_dir + "/include"
out_libdir = out_dir + "/lib"

openssl_static = (int(excons.GetArgument("openssl-static", "1")) != 0)

def LibSSLName():
   libname = "ssl"
   if sys.platform == "win32":
      libname = "lib" + libname
   return libname

def LibSSLPath():
   name = LibSSLName()
   if sys.platform == "win32":
      libname = name + ".lib"
   else:
      libname = "lib" + name + (".a" if openssl_static else excons.SharedLibraryLinkExt())
   return out_libdir + "/" + libname

def LibCryptoName():
   libname = "crypto"
   if sys.platform == "win32":
      libname = "lib" + libname
   return libname

def LibCryptoPath():
   name = LibCryptoName()
   if sys.platform == "win32":
      libname = name + ".lib"
   else:
      libname = "lib" + name + (".a" if openssl_static else excons.SharedLibraryLinkExt())
   return out_libdir + "/" + libname


if int(ARGUMENTS.get("build-openssl", "0")) != 0:
   target = ("no-shared" if openssl_static else "shared")
   if sys.platform != "win32":
      plat = ("darwin64-x86_64-cc" if sys.platform == "darwin" else "linux-x86_64")
      OpenSSLConfigure = "cd openssl ; ./Configure %s -fvisibility=hidden --prefix=%s --openssldir=%s %s ; cd .." % (target, out_dir, out_dir, plat)
      OpenSSLBuild = "cd openssl ; make install_dev ; cd .."
   else:
      od = out_dir.replace("/", "\\")
      OpenSSLConfigure = "cd openssl && perl ./Configure %s --prefix=%s --openssldir=%s VC-WIN64A-masm && cd .." % (target, od, od)
      OpenSSLBuild = "cd openssl && nmake install_dev && cd .."
   env.Command(["openssl/Makefile"], ["openssl/Configure"], OpenSSLConfigure)
   env.Command([LibSSLPath(), LibCryptoPath()], ["openssl/Makefile"], OpenSSLBuild)

else:
   excons.tools.python.RequireCython(env)

   prefix = "%s/%s/ssh2" % (excons.tools.python.ModulePrefix(),
                            excons.tools.python.Version())

   ssh2_static = (int(excons.GetArgument("libssh2-static", "0")) != 0)

   def LibSSH2Name():
      libname = "ssh2"
      if sys.platform == "win32":
         libname = "lib" + libname
      return libname

   def LibSSH2Path():
      name = LibSSH2Name()
      if sys.platform == "win32":
         libname = name + ".lib"
      else:
         libname = "lib" + name + (".a" if ssh2_static else excons.SharedLibraryLinkExt())
      return out_libdir + "/" + libname

   def RequireLibSSH2(env):
      env.Append(CPPPATH=[out_incdir])
      env.Append(LIBPATH=[out_libdir])
      excons.Link(env, LibSSH2Path(), static=ssh2_static, force=True, silent=True)
      if ssh2_static:
         excons.Link(env, LibSSLPath(), static=openssl_static, force=True, silent=True)
         excons.Link(env, LibCryptoPath(), static=openssl_static, force=True, silent=True)
         env.Append(LIBS=["ws2_32", "advapi32", "crypt32", "user32"])

   prjs = [{"name": "libssh2",
            "type": "cmake",
            "cmake-root": "libssh2",
            "cmake-opts": {"BUILD_SHARED_LIBS": "ON",
                           "CRYPTO_BACKEND": "OpenSSL",
                           "OPENSSL_ROOT_DIR": out_dir,
                           "OPENSSL_USE_STATIC_LIBS": ("TRUE" if openssl_static else "FALSE")},
            "cmake-cfgs": excons.CollectFiles(["libssh2"], patterns=["CMakeLists.txt"], recursive=True),
            "cmake-srcs": excons.CollectFiles(["libssh2/src"], patterns=["*.c"], recursive=True),
            "cmake-outputs": map(lambda x: out_incdir + "/" + os.path.basename(x), excons.glob("libssh2/include/*.h")) +
                             [LibSSH2Path()] +
                             [LibSSH2Path().replace("/lib/", "/bin/").replace(".lib", ".dll")]}]

   for item in glob.glob("ssh2/*.pyx"):
      modname, _ = os.path.splitext(os.path.basename(item))
      c, h = excons.tools.python.CythonGenerate(env, item, incdirs=[out_incdir], cte={"EMBEDDED_LIB": "1", "HAVE_AGENT_FWD": "0"})
      prj = {"name": modname,
             "type": "dynamicmodule",
             "ext": excons.tools.python.ModuleExtension(),
             "prefix": prefix,
             "srcs": [c],
             "custom": [RequireLibSSH2, excons.tools.python.SoftRequire]}
      prjs.append(prj)

   # Main target: ssh2

   deps = [x["name"] for x in prjs]
   insts = ["ssh2/__init__.py",
            "ssh2/_version.py"]
   if sys.platform == "win32" and not ssh2_static:
      dll = os.path.join(out_bindir + "/libssh2.dll")
      if os.path.isfile(dll):
         insts.append(dll)

   prjs.append({"name": "ssh2",
                "type": "install",
                "prefix": prefix,
                "deps": deps,
                "install": {prefix: insts}})

   excons.DeclareTargets(env, prjs)
