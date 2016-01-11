# CocoaPods

**NOTE:** This document doesn't provide a description of how CocoaPods
currently works (unfortunately!), but how I think it should work as this would
avoid so many problems I've been running into already and it would follow all
recommendations of Apple and does things exactly like Apple is doing them.

## Glossary

- **Modul**

  A module is a reusable piece of software. There are three kind of modules
	in OS X:

	- **Static Libraries**

	  They consist of a single file only, their extension is `.a` and they
		are linked into the binary at built time, that means it is not necessary to
		ship them together with the target (e.g. there's no need to embedded
		them into an application bundle).

	- **Dynamic Libraries**

    They consist of a single file only, their extension is `.dylib` and they
		need to be shipped together with the target as the final linking with
		the binary happens dynamically at runtime (they are typically embedded
		into the application bundle).

	- **Frameworks**

		They are bundles themselves (directories appearing as a single file),
		consisting of at least a property list (`Info.plist`) and a library.
		They can also contain further files like headers or a module map, as well
		as further embedded frameworks and dynamic libraries. Their extensions is
		`.framework`. The library is usually a dynamic one, in which case the whole
		framework needs to be shipped together with the target, yet not necessarily
		all files (the library and the property list are required, as well as any
		embedded frameworks and libraries, the rest isn't). Frameworks can also
		contain a Static library (that's very rare, but allowed), in that case the
		framework itself doesn't have to be shipped the application after linking.

 CocoaPods currently supports only Static Libraries (default and only works
 with C/Objective-C) and Frameworks with Dynamic Libraries inside (only when
 requested, required for Swift). Basically every pod is a module that is
 linked against and possible embedded into the application you build.

- **Private Headers**

  Private Headers of a pod are headers only required to build the pod itself,
	they are not required to later on use that pod within the target code.
	(**Careful:** *Xcode also calls certain headers "private" as well, but
	these are not the same as the headers that CocoaPods calls private.
	Private headers of CocoaPods would be named "project headers" in Xcode*)

- **Public Headers**

  Public Headers are headers that are required to use or integrate the module
	into a target.

- **Source Code**

  The Source Code is the code of the pod or of the target. Typically these are
	C (`.c`), Objective-C (`.m`) or Swift (`.swift`) files. For CocoaPods all
	files are considered source files, even header files (`.h`), which is
	technically correct as they contain source code, however when the term source
	code is used in the text below, it doesn't refer to header files.

- **Target**

  The Target is what is being build assisted with CocoaPods. It's usually an
	application but in rare cases my also be a deployable library or framework.

- **Umbrella Header**

  An Umbrella Header is a special public module header that includes/imports
	all the other public module headers. Instead of inculding/importing single
	public headers, one simply imports the umbrella header. By default it is
	named exactly like the module itself, os if the module is named `MyModule`,
	the umbrella headers would be named `MyModule.h`.

- **Module Map**

  A Module Map is a file that is written in a Module Map Language and it
	contains information about the module, sub-moduls, header files of the
	module, exported symbols, dependencies and how all this works together. In
	most cases the file defines the name of a module, the umbrella header and
	exports all symbols (wildcard) that this header defines. One speciality of
	a module map is that if a module map defines an umbrella header, every
	attempt to import/include any header of the module will cause the compiler
	to import/include the umbrella header instead.

- **User Imports/Includes**

	User Imports/Includes are imports/includes where the the compiler will
	search for headers in user search paths only. Typically these are used to
	import/include headers for the current target, e.g. private headers.

- **System Imports/Includes**

	System Imports/Includes are imports/includes where the the compiler will
	search for headers in system search paths only. Typically these are used to
	import/include headers from other modules and/or the operation system
	itself.



## Integrating Pods (for Developers)

If developers want to import/include a pod named XYZ, they'd typically do so
by importing its umbrella header, e.g.

    #import <XYZ/XYZ.h>

or in C code

    #include <XYZ/XYZ.h>

As these are headers from an external project, "angle brackets" are used
because they are not headers of the current target. The samples above work
for both, modules that are dynamic frameworks and modules that are static
libraries.

If frameworks are used, all pods should have a module map. As a result, one
can also import modules using the new syntax

    @import XYZ;

and in Swift one just uses

    import XYZ;


## Writing Pods (for Pod Developers)

Pod developers should use user imports/includes without a "namespace" in all
their source code and private headers. To include abc.h, they use

    #import "abc.h"

or

    #inlcude "abc.h"

**(NOT `"ModuleName/adc.h"`, `<abc.h>`, or `<ModuleName/abc.h>`!!!)**

Quotes because the header is part of the current target and not from an
external module.

*However, within all public headers imports/includes must look exactly as when
integrating pods! See above. That's because public headers will in fact become
a part of some target and there only these kind of imports/includes are
guaranteed to work.*

Pod developers don't have to write an umbrella header themselves. CocoaPods
automatically creates one for them if required. The name is the name of the
module itself (the pod name) and the content are import statements for all
public headers of the pod.

However, if developers want to have more control about the content of the
umbrella header, they can also write one themselves. Just having a public
header named like the module makes CocoaPods recognize this header as an
umbrella header. In that case CocoaPods may only write a module map for the
developer and uses the umbrella header "as is".

If developers need even more control, they can even write the module map
themselves and tell CocoaPods where to find it. In that case CocoaPods will
create neither a module map nor an umbrella header and that way the umbrella
header can also have a different name than the module (pod) itself, if that
is desired. In that case it's all entirely up to the pod developer.


## For the Developers of CocoaPods itself

When building the pod itself, all private headers must be in the users
header search path in Xcode - which they are by default if they are added to
the Xcode project itself.

Public headers or symlinks to public headers must exist in a directory named
after the module and the parent directory of that directory must be in the
system search paths to make the import scheme of above work.

E.g. you have a module XYZ with the public headers pubA.h and pubB.h and as
well as the private headers privK.h and privL.h, the directory structure could
look as follows:

    Pods/XYZ/Pod:
	    fileA.m
      fileB.m
			pubA.h
			pubB.h
			privK.h
			privL.h

		Pods/Headers/Public/XYZ/XYZ:
		  pubA.h --> ../../../XYZ/Pod/pubA.h
			pubB.h --> ../../../XYZ/Pod/pubB.h

Doing the same with the private headers is actually not required. One could do
the following:

		Pods/Headers/Private/XYZ:
		  privK.h --> ../../../XYZ/Pod/privK.h
      privL.h --> ../../../XYZ/Pod/privL.h

and then add "Pods/Headers/Private/XYZ" to the users header search path, but
it serves no meaningful purpose, at least not in a flat header hierarchy.

When building the pod itself, the directory `Pods/Headers/Public/XYZ` must be
in the system header search paths, otherwise imports like

      #import <XYZ/pubA.h>
			#import <XYZ/pubB.h>

will not work. If the pod wants to preserve some more complex import structure,
(`header_mappings_dir`), make sure it is replicated in
`Pods/Headers/Public/XYZ` to make these imports work as well!

**NOTE: replicate in `Pods/Headers/Public/XYZ`, NOT in
`Pods/Headers/Public/XYZ/XYZ` - if developers want the module name in the
import, they can can control that themselves by making a directory with that
name but they would have no way to get it out of the path if CocoaPods forces
it there and if they could choose import statements freely, they'd hardly
use `header_mappings_dir` in the first place!**

If there is a public header named `XYZ.h`, this header shall be treated at the
umbrella header and used unmodified. If no such header is found and umbrella
header is required, CocoaPods should create a public headers XYZ.h and for
every other public header add a

    #import <XYZ/header.h>

*Never use imports without namespace (`<header.h>`), user imports
(`"header.h"`), or import anything from any other module (like importing
`<Foundation/Foundation.h>` or similar), that is not required in an umbrella
header and if one of the public header needs that, it must import the
headers of other modules itself.*

If a module map is not generated but provided by the pod itself, an umbrella
header is not generated either. If a module map is generated, a headers named
after the module is searched and if found, used as the umbrella headers. If no
such headers is found, an umbrella header is generated together with the
module map.


### Behavior when Pods are Static Libs

When static libraries are used, there is no need to tag any header files in
pod project as "public". This will only cause them to be copied to the build
dir, cluttering that directory up and causing conflicts if two different pods
have an equally named public header file, as in the build dir the structure is
always flat and this directory is also within the header search paths in Xcode!

Instead, when building the actual target, add the public headers of all
integrated pods to the system header search path:

    = ($inherited) "Pods/Headers/Public/XYZ" ...

this is enough to make `#import <XYZ/header.h>` imports work (e.g. from within
the umbrella header), there are no name conflicts as all imports are prefixed
by the module name and thus **uniquely** identifying one specific header,
and finally linking the target against the static library from the build dir.

Targets don't need any access to private headers or implementation files of
a pod. So that's really all that is required and is much nicer than the
current behavior. Alternatively one could copy public headers to build dir but
then the public header target should be set per pod and it should be something
like `include/ModuleName`.


### Behavior when Pods are Static Libs

When building a framework, it is really required to mark public headers as
"public" in the Pod project to make sure Xcode copies them to the target
directory, which is not the build dir, but the headers dir within the framework
directory. Unlike in case of static libraries, there is no need to fiddle
around with any search paths at all, just linking against the framework will
make imports work as desired and expected.
