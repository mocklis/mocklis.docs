============
Installation
============

There are basically two things that you'll need to get hold of to run Mocklis. Firstly there is a code generator that builds
test double classes from interface definitions. Then there is a library of pluggable 'steps' which provide bite-sized
behaviours to the test doubles, along with some supporting code. This library is spread over a number of assemblies, most
notably ``Mocklis.core`` which contains everything needed to compile test doubles, and ``Mocklis`` which you use in your tests
to add behaviour. Ok - so at the moment there are *only* those two...

Code generation
===============

What you install in your development environment depends on what your development environment is. The deepest integration
is with Visual Studio, where the code generator is implemented as a `Roslyn <https://github.com/dotnet/roslyn>`_ Refactoring.
Mocklis also ships a command-line version which you point to a solution file to generate code. It is also possible to write
Mocklis-compatible code manually, but this doesn't really give you the development experience we're aiming for. A 
`.NET Core Global Tool <https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools>`_ is also a very possible future 
addition.

Visual Studio
-------------

To install Mocklis for Visual Studio you need to get hold of a Mocklis.vsix file, or build it from sources. In the latter
case you need a Visual Studio with the `Visual Studio extension development` Workload enabled - use the Visual Studio
installer to get this done. Then you should be able to build the ``Mocklis.vsix`` project, which will create a vsix package.

*Note: If you run the vsix project instead of building it, it will start up a second instance of Visual Studio with the extension
enabled, with the original Visual Studio having a debugger attached. Super useful for development work in the code generation
projects.*

Then it is as simple as running the vsix-file and Visual Studio will be extended with Mocklis code generation capabilities.

Command-line
------------

If you want the command line version, you have to build it from sources. Once done, you run ``Mocklis.cli path/to/yourproject.sln``
and all available test doubles will be rewritten. Arguably this experience leaves quite a bit to be desired, but that
is where we currently are.

Mocklis Libraries
=================

Once the Mocklis libraries reach some sort of beta quality mark, they will start to appear on NuGet, and then it is just a
case of adding them as packages to your projects. In the meantime there are a few more hoops to go through. The options as
they currently stand are:

Use NuGet packages from own source
----------------------------------

Even though the NuGet packages for Mocklis are not on `NuGet <https://www.nuget.org>`_ (yet) you can still get hold of them. Or you
can build them yourself from the sources, by executing the following commands from the solution ('src') folder:

.. sourcecode:: none

    dotnet build Mocklis.Core\Mocklis.Core.csproj -c Debug
    dotnet pack Mocklis.Core\Mocklis.Core.csproj -c Debug
    dotnet build Mocklis\Mocklis.csproj -c Debug
    dotnet pack Mocklis\Mocklis.csproj -c Debug

This will land you with two `.nupkg` Packages. Just drop them in a folder and point your NuGet in Visual Studio at that folder.
You will need to enable 'prerelease' packages to your search when adding the NuGet packages to your projects to find them.

Also you'll need to enable 'SourceLink', with a fallback to Git Credentials Manager, and disable 'Just My Code' if you want to
be able to step into the Mocklis Source code.

Copy projects or assemblies from Mocklis project
------------------------------------------------

The other option that you have is to download the source code, and just copy the Mocklis and Mocklis.Core projects to your own
solution and reference them there. This gives you a slightly better debugging experience, but is fiddlier, particularly if the
packages were to be updated.
