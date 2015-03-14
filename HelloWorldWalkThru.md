# Hello World Walk-Thru #

Lets take the most simple example possible - a function that takes no arguments and returns the string "Hello World!"

This page will walk you through creating such a plugin, step-by-step. The final code is available in the source tree under `examples/hello_world`.

## Writing HelloWorld ##

This is `helloworld.cc`.

```
#include <string>
#include "helloworld.h"

std::string HelloWorld::GetHw() {
  std::string hw;
  hw = "Hellow World";
  return hw;
}
```

And we'll have our associated `helloworld.h`:

```
#ifndef HELLOWORLD_H
#define HELLOWORLD_H

#include <string>

class HelloWorld {
 public:
  HelloWorld() {}
  std::string GetHw();
};
#endif  // HELLOWORLD_H
```

To get a JS binding to this C++, you simply need to add an IDL line to tell nixysa you want this to be part of your API. We'll call this `helloworld.idl`:

```
[binding_model=by_value, include="helloworld.h"] class HelloWorld {
  HelloWorld();
  std::string GetHw();
};
```

That's it! There are two stub files you must fill out in order to build. The first is a file that describes your plugin - typically called `plugin.cc`:

```
#include <npapi.h>

extern "C" {
  const char *NP_GetMIMEDescription(void) {
    return "application/HelloWorld::Hello World Test";
  }

  NPError NP_GetValue(NPP instance, NPPVariable variable, void *value) {
    switch (variable) {
      case NPPVpluginNameString:
        *static_cast<const char **>(value) = "Hello World";
        break;
      case NPPVpluginDescriptionString:
        *static_cast<const char **>(value) = "Hello World Plugin";
        break;
      default:
        return NPERR_INVALID_PARAM;
        break;
    }
    return NPERR_NO_ERROR;
  }
}
```

Next, nixysa uses SCons instead of Make. You can use an example `SConstruct` file, or I'll provide one here. The only lines you have to touch are `IDL_SOURCES`, `SOURCES` and the first argument to `env.SharedLibrary` which is the name of the sharedobject to create. Leave the rest. This is `SConstruct`.

```
import os
import sys

IDL_SOURCES=['helloworld.idl']
SOURCES=['helloworld.cc', 'plugin.cc']
STATIC_GLUE_SOURCES=['common.cc', 'npn_api.cc', 'static_object.cc', 'main.cc']

env = Environment(
    ROOT = '../nixysa',
    NIXYSA_DIR = '$ROOT/nixysa',
    STATIC_GLUE_DIR = '$NIXYSA_DIR/static_glue/npapi',
    NPAPI_DIR = '$ROOT/third_party/npapi/include',
    GLUE_DIR = 'glue',
    CPPPATH=['.', '$STATIC_GLUE_DIR', '$NPAPI_DIR', '$GLUE_DIR']
)
env.Append(ENV={'PYTHON': sys.executable})
if sys.platform == 'win32':
  env.Append(CODEGEN = 'codegen.bat',
             CPPDEFINES = ['WIN32', 'OS_WINDOWS'])
else:
  env.Append(CODEGEN = 'codegen.sh',
             CPPDEFINES = ['OS_LINUX'])

def NixysaEmitter(target, source, env):
  bases = [os.path.splitext(s.name)[0] for s in source] + ['globals']
  targets = ['$GLUE_DIR/%s_glue.cc' % b for b in bases]
  targets += ['$GLUE_DIR/%s_glue.h' % b for b in bases]
  targets += ['$GLUE_DIR/hash', '$GLUE_DIR/parsetab.py']
  return targets, source

NIXYSA_CMDLINE = ' '.join([env.File('$NIXYSA_DIR/$CODEGEN').abspath,
                           '--output-dir=$GLUE_DIR',
                           '--generate=npapi',
                           '$SOURCES'])

env['BUILDERS']['Nixysa'] = Builder(action=NIXYSA_CMDLINE,
                                    emitter=NixysaEmitter)

AUTOGEN_OUTPUT = env.Nixysa(IDL_SOURCES)
AUTOGEN_CC_FILES = [f for f in AUTOGEN_OUTPUT if f.suffix == '.cc']

env.SharedLibrary('helloworld', AUTOGEN_CC_FILES + SOURCES +
                  ['$STATIC_GLUE_DIR/' + f for f in STATIC_GLUE_SOURCES])
```

nixysa already knows how to convert a C++ string to a javascript string, and will handle everything else for you.

I chose to lay my tree out like this:

```
nixysa/
  examples/
  third-party/
  ...
src/
  helloworld.cc
  helloworld.idl
  helloworld.h
  ...
```

But you can change the paths in the SConstruct file if you wish to change that.

Now build by running `scons`, and drop libhelloworld.so into the `plugins` directory of your favorite browser.

We'll walk through some of the pieces that happen in a minute, first lets look at how you would use this.

```
<html>
<head>
<script type="text/javascript">

function init() {
  var plugin = document.getElementById("plugin");
  var hw = plugin.HelloWorld();
  if (!hw) {
    alert("no plugin");
  }

  alert(hw.getHw());
}
</script>
</head>

<body onload="init()">
<object type="application/HelloWorld" id="plugin" width="0" height="0"> </object>
</body>
</html>
```

Here we make the plugin load with an `object` tag that has the right MIME-type. It's important to remember that you need an init function `onload`, or the plugin won't be loaded when you try to instantiate your object.

In your init function, you pull a reference to the plugin and then instantiate your object. From here you can call methods on it.

**BE WARNED**: Nixysa does a variety of namespace mangling to allow you to use C++-style names in your C++ and still have Javascript-style names in your JS. All functions exported to JS will have the first character lower case and the rest camel-case. So `FooBar()` will become `fooBar()` and `foo_bar()` will become `fooBar()` as well.

When you call `hw.getHw()` above, you are calling a NPAPI backed function written by Nixysa which will then call your method, and return the value back to javascript.

Nixysa natively understands `bool`s, `int`s, `float`s, `double`s and `std::string`s.