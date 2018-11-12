# Connos scheme build

The motivation of defining the scheme is to build SDK by using conan build packages. It is a  yaml formated file, which provides more specific infomation about [conan](https://conan.io) package. It describe what's settings and options used for each package, for example if you build zlib with conan, in conanfile.py file, you can build with the composed of 'CPU arch' ,'build_type', 'share' or 'static'. It's very powerful, but if you want to build SDK and co-work with other packages, you must point out shared or static lib, that's infomation will be defeined in this scheme file.

Once you have a scheme.yml, you may not change too much on coanfile.py and [CPT](https://github.com/conan-io/conan-package-tools) build.py

in conanfile.py, you only need add config_scheme at lat of configure function

``` python
from conanos.build import config_scheme

class ZLibConan(ConanFile):
  ....
  def configure(self):
    ......
    config_scheme(self)

```

in build.py, you only need less then 10 line code for a package

``` python
from conanos.build import Main
if __name__=='__main__':
   Main('zlib')   
```

to run the build under some scheme you should set eviroment vars

* **CONANOS_SCHEME**   *the name of scheme*
* **CONANOS\_SCHEME\_REPO**   *the loacation of the scheme repositories*

If  **CONANOS\_SCHEME\_REPO** not set the default value  https://raw.githubusercontent.com/conanos/conanos.github.io/schemes would be used.

Finally the conanos will find scheme.yml at ${CONANOS\_SCHEME\_REPO}/${CONANOS_SCHEME}/scheme.yml
It works with [conanos](https://github.com/conanos/conanos.py) to build SDK according your designs, so you have to undertand [CPT](https://github.com/conan-io/conan-package-tools) well to work on it.

## Format

There are to section in the top level - *settings* ,*packages*

_settings_ tells the conan tools what's CPU(arch), Compiler (and compiler version), build_type (Debug or Reelase), shared or static lib to build for general. It's hierarchy from the general to spcification.

_packages_ tells more restrict about specific package on _settings_ and _options_

``` yaml
# example scheme.yml top level
settings:
   ......

packages:
  zlib:
  jpeg:
  .......
```

### Settings scetion
This section used to determine arch,build_type,lib type for package build . it composed of some "setting" unit, and list at  'compiler' and 'compiler.version' branchs. the "setting" unit like below
``` yaml
# setting unit
arch:
  - x86
  - x86_64
lib:
  - shared
  - static
build_type:
  - Debug
  - Release
```
You can see, there are 3 items in setting arch,lib and build_type. the setting item are list type, but if you have only one defined for the item ,you can simplify it with *key:value* pattern, for example some setting only support x86_64 build

```yaml
    arch: x86_64  # <-- we only build for x86_64 
```

An overview of the **settings** like this
``` yaml
settings:
  - *:
    [setting unit]

  - <compiler>: # compiler = gcc,msvc,clang,emcc
  	- *: 
      [setting unit]

  	- <compiler.version> # compiler.version is number to indicate the compiler's version
      [setting unit]

  - <compiler>:

  	- <compiler.version>
      [setting unit]
  	
```

In above example, you saw the star **\***, it use as general rule. The settings is hierarchical structure, you can infer the final build setting from the top down. Every star part **'- \*:'**, tell you if nothing configured later, use this setting item here. 

``` yaml
settings:
  - *:
    arch: 
    - x86_64
    - x86
  - msvc:
	  - *:
	    arch: x86
    - 15:
      arch: x86_64
  - gcc:
    - 7:
      arch: x86_64
	   
```
This example infer matrix as below ( msvc 14,15, gcc 6,7 ,clang 4 only)

| Compiler | Version | arch      | 
|----------|-------- |-----------| 
| msvc     | 15      | x86_64    |   
| msvc     | 14      | x86       |
| gcc      | 6       |x86_64,x86 |
| gcc      | 7       |x86_64     |
| clang    | 4       |x86_64,x86 |


### Packages section
In packages section, the first level is packages

``` yaml
packages:
  zlib:
    settings:
      ......
    options:
      .....
    deps:
      ....
  
  leptonica:
    ..... 
```
Every package defined on settings , options ,deps.

### Settings
The *settings* is further defination base on top level settings. a widely usage is somthing like this

in top level setting defined only build for shared lib. the below code pieces, indicate zlib will build both shared and static lib for gcc, other compiler only build shared lib.

``` yaml
settings:
   - *:
     lib: shared
packages:
  zlib:
    settings:
      - gcc:
        - *:
          lib:
            - shared
            - static
```

#### Options
the options part point out what's value of option which defianed in connafile.py, if not set in the scheme file the conanos build tool will use default value of conanfile.py

in conan build zlib connanfile.py
options defined as below
``` python
options = {"shared": [True, False], "fPIC": [True, False], "minizip": [True, False]}
default_options = "shared=False", "fPIC=True", "minizip=False"
```
if we defined package as below, the gcc build for zlib will set with options['minizip]=True, other build will use the default value 'False'
``` yaml
packages:
  zlib:
    options:
      minizip:
        True:
          - gcc:
```

you may guess the options structure like this

```yaml
options:
  <option name>:
    <option value>:
       [setting unit]
```


### Dependency

deps part figure out, wht's kind of lib to link (shared or static), if not use the default. 

**note**: the default value is not defined in conanfile.py, instead it's infer value in the schemel.yml

example,

if we have two packageA and packageB, and both of them use zlib.

if we use below scheme, the packageA will explicted to link shared zlib. packageB will link to static because we infer the zlib build in this scheme will be static according global settings

```yaml
settings:
  - *:
    lib: static
deps:
   packageA: 
     deps:
       zlib: shared
   packageB:

```
