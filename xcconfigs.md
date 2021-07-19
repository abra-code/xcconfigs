# xcconfigs

Xcconfig is an Xcode build configuration file. It is a plain text file with a simple basic syntax of **KEY = value**.

### Build Settings
Build settings are used by Xcode to control variables for different build phases, for example compilation, linking, script phases, code signing, etc.
Some of them are controlling private build phases, some might be translated to command line arguments for various build tools and others might be just raw arguments for tools.
Example settings:
```
CLANG_ENABLE_OBJC_ARC = YES
OTHER_LDFLAGS = -ObjC -framework CoreGraphics
CODE_SIGN_IDENTITY = iPhone Developer
INFOPLIST_FILE = MyAppInfo.plist
```

An Xcode project consists of targets which in turn have multiple build phases. Build settings are resolved per target. One set of build settings is a union of all key-value pairs and applies to all build phases in one target. The settings which don't apply to given build phase are ignored.

### Assigning xcconfigs in Xcode
Xcconfigs can be assigned in Xcode for the project itself and for each target. Each configuration (Debug or Release) may have a different xcconfig.
**TODO:** _add a screenshot_

### Inheritance Levels
In the simplest form target-level settings inherit and may override settings from project-level, which in turn inherit from immutable Xcode defaults for given platform.
Full inheritance structure is:

* Xcode built-in settings per platform (macOS, iOS, etc) - immutable
* Project-level xcconfig
* Project-level settings embedded in Xcode project
* Target-level xcconfig
* Target-level settings embedded in Xcode project

When building with xcodebuild, there are 2 additional levels:

* "-xcconfig" flag allows specifying an xcconfig file with overrides, e.g.:
	`xcodebuild -xcconfig path/to/override.xcconfig`

* you may pass KEY=VALUE straight as a parameter to xcodebuild, e.g.:
	`xcodebuild COMPILER_INDEX_STORE_ENABLE=NO`
	That method trumps all other methods on all levels, including "-xcconfig" config file and is the ultimate override

### Inclusions
Xcconfigs may include other xcconfigs with #include "other.xcconfig" directive. There are no header search paths for xcconfig inclusion. Which means that #include must be either relative or absolute.
* Relative #include files are found by Xcode in 2 locations:
	* relative to Xcode project
	* relative to the including xcconfig itself
* Absolute #includes are harder to maintain in any shared development and probably make sense only when xcconfigs are generated (along the generation of the xcode project itself)

### Variable Resolution Rules and Combining Values

Variables in xcconfigs can be single values, for example:
```
ALWAYS_SEARCH_USER_PATHS = NO
CLANG_WARN_INT_CONVERSION = YES
```
or a space-separated list of values, for example:
```
OTHER_CFLAGS = -ferror-limit=0 -fstack-protector -D MYDEFINE=1
LD_RUNPATH_SEARCH_PATHS = @executable_path/../Frameworks @loader_path/Frameworks
```

Variables can refer to other variables in form of  **$(KEY)** or **${KEY}**, for example:
```
FOO = one
BAR = $(FOO) two
```
Resolves to:
```
BAR = one two
```
Undefined variables are not causing errors but are replaced by empty strings:
```
BAR = $(BAZ) two
```
Resolves to:
```
BAR =  two
```
There is one important resolution rule which at first may seem unintuitive but is very powerful and allows great flexibility. It can be summarized by this fragment:
```
FOO = one
BAR = $(FOO) two
FOO = three
```
The  expectation would be that if the order matters it would resolve as `BAR = one two`. But actually xcconfigs subsitutes variables lazily and the later overriden value of FOO wins and therefore the resolution is:
```
BAR = three two
```
This is very useful because you can use some key with or without default value in a lower level xcconfig which will be provided or overriden later in target level

### $(inherited)

One special variable which can be referred to is **$(inherited)** as in:
```
FRAMEWORK_SEARCH_PATHS = $(inherited) some/additional/framework/location
```
That syntax is actually identical to:
```
FRAMEWORK_SEARCH_PATHS = $(FRAMEWORK_SEARCH_PATHS) some/additional/framework/location
```
The way it works is that $(inherited) is replaced with the previous value of the same key. In all revisions of Xcode up to Xcode 10 the inheritance was only allowed from one level to another as described in "Inheritance Levels" section above but not on the same level.
With the "New Build Sytem" Apple changed the rules and respects previous definition value even if it is not on previous level but just defined earlier.
* In Legacy Build System in the same xcconfig (same level):
```
	FOO = foo
	FOO = $(inherited) bar
```
resolves as:
```
	FOO = bar
```

* If we have FOO in project-level xcconfig as:
 ```
	FOO = foo
```
And in the next level, for example target-level xcconfig:
 ```
	FOO = $(inherited) bar
```
it resolves to :

```
	FOO = foo bar
```

* In New Build System the same construct in both cases resolves as:
```
	FOO = foo bar
```

### Conditionals

xcconfig value resolution engine does not allow having "if" statements. There are a couple of workarounds for this deficiency:

* KEY[scope=filter] = val
	There are limited number of predefined scopes which can be used:
	* arch= with values like arm64, x86_64
	* sdk= with values like iphonesimulator, iphoneos, macosx, watchos, watchsimulator
	* config = with values like "Debug" or "Release"
	
	Wildcards can be used in the filter strings, for example `arch=*64*` covers `x86_64`, `arm64`, `arm64e`, and `arm64_32`

* #include? "optional.xcconfig" - the question mark allows including some other xcconfig if it exists and does not complain if it is not found
* construct variable names with other variable values, for example:
```
	BUILD_TYPE_SWITCH = NEW
	//other possible value for BUILD_TYPE_SWITCH is OLD
	BUILD_NEW = new
	BUILD_OLD = old
	BUILD_TYPE = $(BUILD_$(BUILD_TYPE_SWITCH))
```
This resolves to:
`BUILD_TYPE = $(BUILD_NEW)` and eventually `BUILD_TYPE = new`
or 
`BUILD_TYPE = $(BUILD_OLD)` and eventually `BUILD_TYPE = old`

depending on whether `BUILD_TYPE_SWITCH` is defined to NEW or OLD

This way you can construct complex variables at any level which are switchable by some override value.

### No Line Continuation Support

In many languages the preprocessor supports "\\" at the end of line as a line continuation character, allowing vertical construction for a long list of values. Xcconfigs do not support this so your choices are:
* Put everything in one long horizontal line which may scroll for many pages like this:
```
OTHER_LDFLAGS = -framework MyFrameworkOne -framework MyFrameworkTwo -framework MyFrameworkThree -framework MyFrameworkFour -framework MyFrameworkFive
```
* Predefine variables which are merged later into the final one
```
LINK_1 = -framework MyFrameworkOne
LINK_2 = -framework MyFrameworkTwo
LINK_3 = -framework MyFrameworkThree
LINK_4 = -framework MyFrameworkFour
LINK_5[arch=arm64] = -framework MyFrameworkFive

OTHER_LDFLAGS = $(LINK_1) $(LINK_2) $(LINK_3) $(LINK_4) $(LINK_5)
```

That way the long concatenated line is not interesting with incrementing index but the individual `LINK_N` variables are now vertical for easy reading.
`LINK_1... LINK_NN` can also be hidden as predefined slots in a separate included xcconfig. You could have a 100 or more of such predefined slots and the unused ones would turn into empty strings in final concatenation.

7/19/2021 Update: the relase notes for Xcode 13 beta say the line continuation support has finally been added!
\
https://developer.apple.com/documentation/xcode-release-notes/xcode-13-beta-release-notes


### Build Settings in Script Phases

All build settings are translated into environment variables in script phases executed by Xcode. This has several important implications:
* you can define helper env vars in xcconfigs to control your scripts
* a change in xcconfig invalidates build script phase even if it does not logically apply to it. Environment variables are part of the hash considered for output invalidaiton
* large projects may exceed the maximum size allowed for environment and arguments for forked processes - this causes random unexplained script failures

### Investigating Resolved Values

If you would like to check and verify how the xcconfig and build settings in general were resolved by Xcode for given target you can do it in 2 ways:
* add a script phase to your target if you don't have one and look at the complete set of exported environment variables during the build
* use xcodebuild **-showBuildSettings** option in Terminal for for specified -project and -target 



