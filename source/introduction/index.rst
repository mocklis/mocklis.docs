============
Introduction
============


Mocklis is a mocking library for .net (currently C#) that

* creates fake implementations of interfaces
* that can be given specific behaviour
* in an intellisense-friendly way
* to be used as dependencies in components we want to test
* letting us verify that they are correctly interacted with
* without any use of reflection.


Let's go over these points one by one.

Fake implementations of interfaces
==================================

With Mocklis you take an interface that defines a dependency for a component we wish to test, for instance this IConnection interface (with some details such as `Message` and `MessageEventArgs` removed for brevity):

.. sourcecode:: csharp

    public interface IConnection
    {
        string ConnectionId { get; }
        event EventHandler<MessageEventArgs> Receive;
        Task Send(Message message);
    }

Then you create an empty partial class implementing this interface, and decorate it with the ``MocklisClass`` attribute.

.. sourcecode:: csharp

    [MocklisClass]
    public partial class MockConnection : IConnection;

This looks like it's missing interface implementations, and therefore shouldn't compile. By itself it does not, but Mocklis has created another file behind the scenes that provides the implementations for us.

There are two simple ways to see the generated source code. One is to right-click on the ``MockConnection`` class name and select ``Go To Implementation``.

.. image:: GoToImplementation.png

Then choose the generated file from the selection:

.. image:: GoToImplementation2.png

The other quick way to look at the generated file is to locate it under the Mocklis.MockGenerator "Analyzer" in the Solution Explorer:

.. image:: GeneratedFilesFromAnalyzers.png

Regardless of which way you get to the file, you will see the generated code in a read-only editor with a bar on the top allowing you to re-run the generator should you want to. This can sometimes be useful.

.. image:: GeneratedCode.png

You can see that the ``IConnection`` interface has been explicitly implemented, where the ``IConnection.ConnectionId`` property gets its value
from a `mock property` with the same name. The ``IConnection.Receive`` event forwards adds and removes to another `mock property`, and the
``IConnection.Send`` method forwards calls to yet another `mock property`.

Note that the `mock properties` are generally properties even if the members they support aren't: the ``IConnection.Send`` method is paired with a `mock property`
of type ``FuncMethodMock``, the ``IConnection.Receive`` event is paired with a `mock property` of type ``EventMock`` and so forth.

The practical upshot is that the ``IConnection`` interface is now implemented by the class, so that instances of the class can be used in places where
the ``IConnection`` interface is expected.

Can be given specific behaviour
===============================

If we just use the class that was written for us in a real test it might not work as expected. The default behaviour for a newly constructed mock is to do
as little as possible and return default values wherever asked, which is probably not what you wanted. The good news is that you can control this on a case
by case basis using so-called `steps`.

.. sourcecode:: csharp

    [Fact]
    public void ServiceCanCountMessages()
    {
        // Arrange
        var mockConnection = new MockConnection();
        mockConnection.ConnectionId.Return("TestConnectionId");
        mockConnection.Receive.Stored(out var registeredEvents);

        var service = new Service(mockConnection);

        // Act
        registeredEvents.Raise(mockConnection, new MessageEventArgs(new Message("Test")));

        // Assert
        Assert.Equal("TestConnectionId", service.ConId);
        Assert.Equal(1, service.ReceiveMessageCount);
    }

In this example, two steps were used. The ``Return`` step simply returns a value whenever the mock is used, and the ``Stored`` step tracks
values written to the mock. In this case we also obtained a reference to the store, which tracked attached event handlers letting us raise
an event on them for testing purposes. (Exercise for the reader: write some code for the `Service` class so that the test passes...)

This is of course just an introduction; see the :doc:`../reference/index` for a complete list of steps and other constructs used to control
how `Mocklis Classes` work.

Intellisense friendly
=====================

Intellisense is a great feature of modern code editors, and Mocklis is written to make the most of it. Your `Mocklis class` exposes `mock properties`
for members of implemented interfaces. These `mock properties` have extension methods for all of the different steps that they support, which allows
Visual Studio will list the available steps through intellisense.

.. image:: Intellisense.png

Thanks to the extension method approach this list would also include any bespoke steps that have been added, whether defined in your own
solution or in third party packages.

When mocking out method calls, all arguments are combined into a named value tuple (unless there's exactly one in which case that one is used),
which means that we get intellisense for using those parameters as well.

.. image:: intellisense2.png

Used as dependencies
====================

Since `Mocklis classes` implement interfaces explicitly, we don't risk a name clash with the `mock properties` (and indeed if possible, the `mock properties`
will be given the same name as the interface member it's paired with), and we can use the `Mocklis class` instance directly wherever the
interface is expected.

`Mocklis classes` can also implement more than one interface in cases where the component it acts as a stand-in for would implement more than
one interface. Common cases include where a class would implement a service interface and ``IDispose``, or an interface with property accessors
and ``INotifyPropertyChanged``. If you need to mock out an enumerable, your `Mocklis class` can mock both ``IEnumerable<T>`` and ``IEnumerator<T>``
at the same time.

However, this also means that `Mocklis classes` can not create mocks for virtual members of an (abstract) base class, as these can not be explicitly implemented.

Verify interactions
===================

There are a number of ways in which you can verify that the 'component under test' makes the right calls to your mocked dependency. There are a number of ways to do this using steps:
simple cases:

* If you have a method you don't expect to be called, you can use a ``Throw`` step to throw an exception which will hopefully bubble up through your code and fail the test.
* If you have a property, event or indexer you can use a ``Stored`` step and manually check that the right value was stored.
* If you have a method then you can use a ``Func`` or ``Action`` step and let that set a flag which you can later manully assert.
* You can use a ``Record`` step to record all interactions and check that the right interactions happened.

Mocklis also has a set of verification classes and interfaces that can be used to add checks to your `mock properties` and to verify
the contents of ``Stored`` steps in a declarative way. You create a ``VerificationGroup``, pass it to checks and verification steps,
and assert everything in one go.

Take for instance this, somewhat contrived, test:

.. sourcecode:: csharp

    [Fact]
    public void TestIndex()
    {
        // Arrange
        var vg = new VerificationGroup("Checks for indexer");
        var mockIndex = new MockIndex();
        mockIndex.Item
            .ExpectedUsage(vg, null, 1, 3)
            .StoredAsDictionary()
            .CurrentValuesCheck(vg, null, new[]
            {
                new KeyValuePair<int, string>(1, "one"),
                new KeyValuePair<int, string>(2, "two"),
                new KeyValuePair<int, string>(3, "thre")
            });

        var index = (IIndex) mockIndex;

        // Act
        index[1] = "one";
        index[2] = "two";
        index[3] = "three";

        // Assert
        vg.Assert(includeSuccessfulVerifications: true);
    }

This test will fail with the following output:

.. sourcecode:: none

    Mocklis.Verification.VerificationFailedException : Verification Failed.

    FAILED: Verification Group 'Checks for indexer':
    FAILED:   Usage Count: Expected 1 get(s); received 0 get(s).
    Passed:   Usage Count: Expected 3 set(s); received 3 set(s).
    FAILED:   Values check:
    Passed:     Key '1'; Expected 'one'; Current Value is 'one'
    Passed:     Key '2'; Expected 'two'; Current Value is 'two'
    FAILED:     Key '3'; Expected 'thre'; Current Value is 'three'

Note that all verifications are checked - it will not stop at the first failure. By default the assertion
will not show the Passed verifications (although the exception itself has a VerificationResult property,
so you can always get to it). If you want to include all verifications in the exception message you need
to pass true for the ``includeSuccessfulVerifications`` parameter, as was done in the sample above. Without
it you would only see the lines that failed.

The `null` values in the example are placeholders for strings that would be added to the relevant lines in
the result to help finding the culprit if the assertion failed.

Without reflection
==================

Maybe this point should have gone in first. Mocklis does not use reflection to find out information
about mocked interfaces, and it does not use emit or dynamic proxies to add implementations on the fly.
Furthermore the mock instance and the object used to 'program' the mock are the same thing.
There are pros and cons with this approach:

Pros
----

* The code is free from 'magic'. What you see is what you get, and you can set breakpoints in both your own and the generated code.

* You can easily extend Mocklis with your own steps, with whatever bespoke behaviour you might need.

* Running your tests is significantly faster than it would have been with on-the-fly generated dynamic proxies.

Cons
----

* The auto-generated bits of the code live separate from the rest of the code.

* The design only really works for interfaces and not for mocking members of virtual base classes.
