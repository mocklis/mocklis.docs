==========================
Frequently Asked Questions
==========================

ValueTuple
==========

*When I'm creating mocks for methods I often come across mock steps that take ``ValueTuple`` as parameters, but I didn't have anything like that in my original
interfaces - what's going on?*

Mocklis strives to map members of interfaces into one of four formats (for events, indexers, methods and properties respectively). In the case of a method
the format is simply an ability to call the method with one parameter and one return type. If you have many parameters, they will be grouped together as
named members of a value tuple, and in the case where you don't have any parameters or a void return type, the 'empty' ``ValueTuple`` will be used. It is
essentially just a struct with no members, and is the closest thing to a `void` type that the .net framework contains.

For the lambda steps (``Func`` and ``Action``, and the instance versions thereof) there are overloads that require the parameters to be of type ``ValueTuple``, and
the action versions require the return type to be a ``ValueTuple`` as well. This is simply to provide a nicer and terser syntax, but there is no way to prevent
intellisense to suggest the fuller version as well. To make it a little bit more concrete, consider the following interface:

.. sourcecode:: csharp

    public interface IMisc
    {
        void SayHello();
        int ScaleByTheAnswer(int p);
    }

    [MocklisClass]
    public class Misc : IMisc
    {
       . . .
    }

When we are creating a mock for the ``SayHello`` method, we have four different (but ultimately equivalent) ways to go about it:

.. sourcecode:: csharp

    var misc = new Misc();

    misc.SayHello.Action(() =>
    {
        Console.WriteLine("Hello");
    });

    misc.SayHello.Action<ValueTuple>(_ =>
    {
        Console.WriteLine("Hello");
    });

    misc.SayHello.Func<ValueTuple>(() =>
    {
        Console.WriteLine("Hello");
        return ValueTuple.Create();
    });

    misc.SayHello.Func<ValueTuple, ValueTuple>(_ =>
    {
        Console.WriteLine("Hello");
        return ValueTuple.Create();
    });

Of course the compiler infers the type parameters - they're just there for clarity. On the other hand for the ``ScaleByTheAnswer`` method,
we only have one overload that works, since this method both accepts and returns stuff.

.. sourcecode:: csharp

    var misc = new Misc();

    misc.ScaleByTheAnswer.Func<int, int>(i =>
    {
        return i * 42;
    });

Again the compiler infers the type parameters, and you can of course shorten the lambda considerably. But in short: you may be presented
with steps that use ``ValueTuple``, in which case it generally pays to look for a different step (or overload of the same step) that doesn't.

"Missing" mock
==============

*When my mock is called, it throws a ``MockMissingException``. But I'm absolutely certain that I did provide a mock implementation. What's going on?*

Update: This now only happens when the MocklisClass attribute was declared with a ``VeryStrict = true`` parameter. The new behaviour (since version
0.2.0-alpha) for both lenient and strict (as in not 'very strict') mocks is that all steps assume an implicit ``Dummy`` step for all further extension
points. In 'very strict' mode there is an implicit ``Missing`` step instead and an exception will be thrown.

You can think of the 'very strict' mode as 'treat warnings as errors'. It's a bit of a pain but it can help find issues with your mocks.

The solution is to chain a next step that does what you want the mock to do, be it a ``Dummy`` step, a ``Return`` step or anything else.

With an interface borrowed from the previous faq entry, here is a case which would throw the exception when used:

.. sourcecode:: csharp

    var misc = new Misc();

    misc.ScaleByTheAnswer.Log();

The ``Log`` step will log the call, and then forward to the 'default' next step which (perhaps surprisingly) throws. Provide a next step as follows and it doesn't throw:

.. sourcecode:: csharp

    var misc = new Misc();

    misc.ScaleByTheAnswer.Log().Dummy();

And of course it doesn't have to be ``Dummy();`` - looking at the name of the method an appropriate mock might be ``.Func(i => i * 42);``...
