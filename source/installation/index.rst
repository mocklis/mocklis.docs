============
Installation
============

There are two things that you'll need to get hold of to run Mocklis.

Firstly there is a code generator that builds test double classes from interface definitions. The recommended way is to use a
Roslyn Analyzer + Code Fix supplied in the form of the NuGet package ``Mocklis.Analyzer``.

Then there is a library of pluggable 'steps' which provide bite-sized behaviours to the test doubles, along with some supporting
code. This library is spread over a number of assemblies, most notably ``Mocklis.Core`` which contains the minimum amount of code
required to build the test doubles, and ``Mocklis`` which you use in your tests to add behaviour. There's also ``Mocklis.Experimental``
for steps whose design is still under development (read: steps that simply haven't been axed yet...) and ``Mocklis.Serilog2`` which
contains a logging provider for Serilog 2.x.

Mocklis is pre-release software, and is not (at the time of writing) released to nuget.org. In order to use the NuGet packages
in a Visual Studio project you'll need to copy them to a location on your hard-drive and add that location as a NuGet source.
You can do this by editing your user-specific NuGet.Config file, which is located at %AppData%\\NuGet, or you could add it to
a NuGet.Config file that is local to a single solution. Let's say that you copied the Mocklis nupkg files to C:\\NuGet\\Mocklis;
then your NuGet.Config file should contain the following:

.. sourcecode:: none

    <?xml version="1.0" encoding="utf-8"?>
    <configuration>
    ...
        <packageSources>
        ...
            <add key="Mocklis" value="C:\NuGet\Mocklis" />
        ...
        </packageSources>
    ...
    </configuration>

Once set up you can add packages to projects with the NuGet browser in Visual Studio. Select your newly created NuGet source
from the source drop-down to the right and make sure you have 'include prerelease' ticked. You should see the Mocklis packages
and be able to add them to your projects. If you don't see your Mocklis source in the drop-down it's likely that the list of sources is
overridden by a different NuGet.Config file somewhere else on your disk. Check the
`NuGet documentation <https://docs.microsoft.com/en-us/nuget/consume-packages/configuring-nuget-behavior>`_ for details.

.. image:: nuget.png
