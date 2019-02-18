==========================
Frequently Asked Questions
==========================

ValueTuple
==========

*When I'm creating mocks for methods I often come across mock steps that take ValueTuple's as parameters, but I didn't have anything like that in my original
interfaces - what's going on?*

Mocklis strives to map members of interfaces into one of four formats (for events, indexers, methods and properties respectively). In the case of a method
the format is simply an ability to call the method with one parameter and one return type. If you have many parameters, they will be grouped together as
named members of a value tuple, and in the case where you don't have any parameters or a void return type, the 'empty' ValueTuple will be used. It is
essentially just a struct with no members, and is the closest thing to void that the .net framework contains.

For the lambda steps ('Func' and 'Action', and the instance versions thereof) there are overloads that require the parameters to be of type ValueTuple, and
the action versions require the return type to be a ValueTuple as well. This is simply to provide a nicer and terser syntax, but there is no way to prevent
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

When we are creating a mock for the SayHello method, we have four different (but ultimately equivalent) ways to go about it:

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

Of course the compiler infers the type parameters - they're just there for clarity. On the other hand for the 'ScaleByTheAnswer' method,
we only have one overload that works, since this method both accepts and returns stuff.

.. sourcecode:: csharp

    var misc = new Misc();

    misc.ScaleByTheAnswer.Func<int, int>(i =>
    {
        return i * 42;
    });

Again the compiler infers the type parameters, and you can of course shorten the lambda considerably. But in short: you may be presented
with mocklis steps that use ValueTuple, in which case it generally pays to look for a different step (or overload of the same step) that doesn't.

"Missing" mock
==============

*When my mock is called, it throws a MockMissingException. But I'm absolutely certain that I did provide a mock implementation. What's going on?*

The answer is that you probably didn't add a step that would handle the call instead of passing it on to another step. Assuming this is the case...

When a step is added that can forward the call to a 'next' step, it will default that next step to throw the Missing exception. It was a judgement
call whether to have next steps default to 'Missing' or to 'Dummy' implementations. The reason for going with 'Missing' was effectively that at least
you'd be told (in no uncertain terms, in the form of a MockMissingException) when you have an incomplete mock implementation.

The solution is to chain a next step that does what you want the mock to do, be it 'Dummy', 'Return' or anything else.

With an interface borrowed from the previous faq entry, here is a case which would throw the exception when used:

.. sourcecode:: csharp

    var misc = new Misc();

    misc.ScaleByTheAnswer.Log();

Log will log the call, and then forward to the next step which throws. Provide a next step as follows and it doesn't throw:

.. sourcecode:: csharp

    var misc = new Misc();

    misc.ScaleByTheAnswer.Log().Dummy();

And of course it doesn't have to be Dummy - looking at the name of the method an appropriate mock might be ``.Func(i => c * 42);``...

The modifier 'readonly' is not valid...
=======================================

*I have an interface that returns a 'ref readonly' value from a method/property/indexer (strike as appropriate), and I get a red squiggly in the generated
code saying that 'The modifier 'readonly' is not valid for explicit interface imlementation.' - what's going on?*

Resharper is going on... You will notice that the code compiles fine, and the behaviour is tracked here:
`https://youtrack.jetbrains.com/issue/RSRP-473141 <https://youtrack.jetbrains.com/issue/RSRP-473141>`_

Hopefully it's fixed by the time you read this...

Mocklis.Analyzer does not reference any other Mocklis package
=============================================================

*Why do I have to manually add references to both Mocklis.Analyzer and Mocklis.Core? Surely the former doesn't work without the latter!*

Yes and no. The Analyzer requires the existance of attributes and classes with the right name and namespace, but they don't strictly speaking have
to come from a NuGet package. You can copy the code straight from the Mocklis source into your own project and the Analyzer wouldn't be any the
wiser, indeed if you are writing Mocklis steps and spend a lot of time debugging the Mocklis source code this really is the way to go. If there is
interest, the Mocklis projects may well be made available in the form of Git Submodules to make this approach easier in the future.

If we enforced loading the NuGet package versions of the libraries whenever the Analyzer was added to a project this would no longer be possible.
