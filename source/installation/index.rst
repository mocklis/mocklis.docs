============
Installation
============

There are two things that you'll need to get hold of to run Mocklis.

Firstly there is a code generator that builds test double classes from interface definitions. The recommended way is to use a
Roslyn Analyzer + Code Fix supplied in the form of the NuGet package ``Mocklis.MockGenerator``.

Then there is a library of pluggable 'steps' which provide bite-sized behaviours to the test doubles, along with some supporting
code. This library is spread over a number of assemblies, most notably ``Mocklis.Core`` which contains the minimum amount of code
required to build the test doubles, and ``Mocklis`` which you use in your tests to add behaviour. There's also ``Mocklis.Experimental``
for steps whose design is still under development (read: steps that simply haven't been axed yet...) and ``Mocklis.Serilog2`` which
contains a logging provider for Serilog 2.x.

Note that usually you will start off by referencing ``Mocklis`` because it references ``Mocklis.Core`` which in turn references
``Mocklis.CodeGenerator``. To this you can then add any additional packages you need such as ``Mocklis.Serilog2`` or ``Mocklis.Experimental``.

You can add the packages to your projects with the NuGet browser in Visual Studio, just make sure you have 'include prerelease'
ticked since Mocklis is still in pre-release. Search for 'Mocklis' while on the Browse tab, and you should see the Mocklis
packages and be able to add them to your projects.

.. image:: nuget.png
