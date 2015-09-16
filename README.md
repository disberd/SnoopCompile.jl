# SnoopCompile

[![Build Status](https://travis-ci.org/timholy/SnoopCompile.jl.svg?branch=master)](https://travis-ci.org/timholy/SnoopCompile.jl)

SnoopCompile "snoops" on the Julia compiler, getting it to log the
functions and argument-types it's compiling.  By parsing the log file,
you can learn which functions are being precompiled, and even how long
each one takes to compile.  You can use the package to generate
"precompile lists" that reduce the amount of time needed for JIT
compilation in packages.

SnoopCompile is not recommended for Julia beginners, and even
experienced users may need several iterations to generate precompile
scripts that work.

## Usage

The easiest way to describe SnoopCompile is to show a snoop script, in this case for the `Images` package:

```jl
using SnoopCompile

### Log the compiles
# This only needs to be run once (to generate "/tmp/images_compiles.csv")

SnoopCompile.@snoop "/tmp/images_compiles.csv" begin
    include(Pkg.dir("Images", "test","runtests.jl"))
end

### Parse the compiles and generate precompilation scripts
# This can be run repeatedly to tweak the scripts

# IMPORTANT: we must have the module(s) defined for the parcelation
# step, otherwise we will get no precompiles for the Images module
using Images

data = SnoopCompile.read("/tmp/images_compiles.csv")

# The Images tests are run inside a module ImagesTest, so all
# the precompiles get credited to ImagesTest. Credit them to Images instead.
subst = Dict("ImagesTests"=>"Images")

# Blacklist helps fix problems:
# - MIME uses type-parameters with symbols like :image/png, which is
#   not parseable
blacklist = ["MIME"]

# Use these two lines if you want to create precompile functions for
# individual packages
pc, discards = SnoopCompile.parcel(data[end:-1:1,2], subst=subst, blacklist=blacklist)
SnoopCompile.write("/tmp/precompile", pc)
```

After the conclusion of this script, the `"/tmp/precompile"` folder will contain a number of `*.jl` files, organized by package.  These files could be added to a package like this:

```jl
module SomeModule

# All the usual commands that define the module go here

# ... followed by:

if VERSION >= v"0.4.0-dev+5512"
    include("precompile.jl")
    _precompile_()
end

end # module SomeModule
```

## `userimg.jl`

Currently, precompilation does not cache functions from other modules; as a consequence, your speedup in execution time might be smaller than you'd like. In such cases, one strategy is to generate a script for your `base/userimg.jl` file and build the packages (with precompiles) into julia itself.  Simply append/replace the last two lines of the above script with

```jl
# Use these two lines if you want to add to your userimg.jl
pc = SnoopCompile.format_userimg(data[end:-1:1,2], subst=subst, blacklist=blacklist)
SnoopCompile.write("/tmp/userimg_Images.jl", pc)
```

**Users are warned that there are substantial negatives associated with relying on a `userimg.jl` script**:
- Your julia build times become very long
- `Pkg.update()` will have no effect on packages that you've built into julia until you next recompile julia itself. Consequently, you may not get the benefit of enhancements or bug fixes.
- For a package that you sometimes develop, this strategy is very inefficient, because testing a change means rebuilding Julia as well as your package.
