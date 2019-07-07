=================
Extending Mocklis
=================



Writing new steps
=================

The best way to learn about writing steps is to look at the source code for existing steps. But in the interest of documentation, here's a sample.

Disclaimer: This is a silly example. You would never write test code that depends on the time of day. It was chosen because you can be absolutely
certain that this step won't ever clash with anything in the Mocklis libraries themselves. (We sincerely hope...)

Phase 1: Write a step
---------------------

If you are writing a 'final' step, implement the I-memberType-Step interface. You just need to implement this interface and you're done.

If you are writing a non-final step, consider (as in: it is very strongly recommended) subclassing the memberType-StepWithNext class, and override
the I-memberType-Step members as you see fit. Otherwise you need to implement strictness checks yourself, and the base class will give you
overridable members to plug in your specific functionality where the base implementation will forward to a next step.
Let's say we're writing a step to nudge our overworked developers to go home by starting to throw exceptions after 5 o'clock.

Let's also say we're writing this for a property. We'll end up with something like this:

.. sourcecode:: csharp

    public class EndOfDayPropertyStep<TValue> : PropertyStepWithNext<TValue>
    {
        private readonly int _cutOffHour;

        public EndOfDayPropertyStep(int cutOffHour)
        {
            _cutOffHour = cutOffHour;
        }

        private void ThrowIfLate()
        {
            if (DateTime.Now.Hour >= _cutOffHour)
            {
                throw new Exception("It's late - start considering calling it a day.");
            }
        }

        public override TValue Get(IMockInfo mockInfo)
        {
            ThrowIfLate();
            return base.Get(mockInfo);
        }

        public override void Set(IMockInfo mockInfo, TValue value)
        {
            ThrowIfLate();
            base.Set(mockInfo, value);
        }
    }

Phase 2: Write an extension method
----------------------------------

If you wanted to use the new step as is, you would have to create an instance of it and feed to the ``SetNextStep`` method of the previous
step. To enable the fluent syntax you'll need to add the step as an extension method on the ``IPropertyStepCaller`` interface.

.. sourcecode:: csharp

    public static class EndOfDayStepExtensions
    {
        public static IPropertyStepCaller<TValue> EndOfDay<TValue>(
            this IPropertyStepCaller<TValue> caller,
            int? cutOffHour = null)
        {
            return caller.SetNextStep(new EndOfDayPropertyStep<TValue>(cutOffHour ?? 17));
        }
    }

Notice the naming convention: The ``EndOfDayPropertyStep`` is added as an extension method named ``EndOfDay``, taking an ``IPropertyStepCaller`` as its 'this'
parameter. An ``EndOfDayMethodStep`` would be added as an extension method also named ``EndOfDay``. There is no risk of a naming clash, as the parameter
types will differ.

Now you can use your new step:

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.TotalLinesOfCode
        .EndOfDay()
        .Return(50);

With the obvious (well - depending on what time it is) result:

.. sourcecode:: none

    System.Exception: It's late - start considering calling it a day.

Phase 3: Generalise
-------------------

The last phase is to look at your newly created step and consider whether it can be used in other situations. You should extend
the step to the different member types if possible.

In some cases the way a step works could depend on the complete state of the mock instance. In these cases you should add new
steps with the same name as your existing ones, but prefixed with 'Instance'. For this version you pass on the ``IMockInfo.Instance``
to the construct you have that uses the instance. Look at the existing ``Lamdba`` steps for the quintessential implementation, however
the ``Record`` and ``If`` steps also have instance versions.

If you work with steps for methods, you might need to consider having different versions depending on whether your
methods take parameters or not, and whether they return things or not. For the ``lamdba`` steps there are two ``FuncMethodStep`` classes,
and two ``ActionMethodStep`` classes.

.. sourcecode:: csharp

    public class FuncMethodStep<TParam, TResult> : IMethodStep<TParam, TResult>
    {
    }

    public class FuncMethodStep<TResult> : IMethodStep<ValueTuple, TResult>
    {
    }

    public class ActionMethodStep<TParam> : IMethodStep<TParam, ValueTuple>
    {
    }

    public class ActionMethodStep : IMethodStep<ValueTuple, ValueTuple>
    {
    }

Note how the ones that don't funnel data constrict either TParam and/or TResult to be of type ValueTuple (read: *void* or *unit* depending on how you were brought up).
While more than one of these might be eligible for use in a given scenario, the design goal is that there should always be one that doesn't require the user
to pass manually created ValueTuple instances.

Writing new verifications
=========================

The idea behind Mocklis' verifications is to create a tree of binary checks that can be verified in one go. When verified, a read-only data structure is created,
that contains information about all the verifications and whether they were successful or not.

A verification implements the IVerifiable interface:

.. sourcecode:: csharp

    public interface IVerifiable
    {
        IEnumerable<VerificationResult> Verify();
    }

...where a truncated version of the VerificationResult struct is as follows:

.. sourcecode:: csharp

    public struct VerificationResult
    {
        public string Description { get; }
        public IReadOnlyList<VerificationResult> SubResults { get; }
        public bool Success { get; }

        public VerificationResult(string description, bool success)
        {
            Description = description;
            SubResults = Array.Empty<VerificationResult>();
            Success = success;
        }

        public VerificationResult(string description, IEnumerable<VerificationResult> subResults)
        {
            Description = description;
            if (subResults is ReadOnlyCollection<VerificationResult> readOnlyCollection)
            {
                SubResults = readOnlyCollection;
            }
            else
            {
                SubResults =
                    new ReadOnlyCollection<VerificationResult>(
                        subResults?.ToArray() ?? Array.Empty<VerificationResult>());
            }

            Success = SubResults.All(sr => sr.Success);
        }
    }

The first constructor is for leaf nodes, and the second is for branch nodes. Note that if any leaf node fails,
all branch nodes up to the root from that leaf node will have failed as well. Therefore if the root succeeds,
we can be sure that all leaf nodes will have as well.

Verifications can either be written as steps. These steps implement the ``IVerifiable`` interface, and the
extension method takes a VerficationGroup as a parameter and attach the created step to that group.

Let's say we're creating a Method step to check that the method has indeed been called. Subclass ``MethodStepWithNext``,
override ``Call`` to set a flag that it has been called, and implement ``IVerifiable`` to return a ``VerificationResult``.

.. sourcecode:: csharp

    public override TResult Call(IMockInfo mockInfo, TParam param)
    {
        _hasBeenCalled = true;
        return base.Call(mockInfo, param);
    }

    public IEnumerable<VerificationResult> Verify()
    {
        var text = "Method should be called, " +
            (_hasBeenCalled ? "and it has." : "but it hasn't.");
        yield return new VerificationResult(text, _hasBeenCalled);
    }

Then we add the step to the verification group in its extension method:

.. sourcecode:: csharp

    public static IMethodStepCaller<TParam, TResult> HasBeenCalled<TParam, TResult>(
            this IMethodStepCaller<TParam, TResult> caller,
            VerificationGroup collector)
        {
            var step = new HasBeenCalledMethodStep<TParam, TResult>();
            collector.Add(step);
            return caller.SetNextStep(step);
        }

But we may want to check some condition without it being a step in its own right. All the ``Stored`` steps
(which would be property, indexer and event) implement an interface to directly access what is being
stored. An implementation of ``IVerifiable`` that is not a step in its own right is called a 'check', and
writing one is straightforward:

Create a class, have it implement ``IVerifiable``. Let the constructor take as input anything it needs to
verify that the condition for the verification has been met. In the case of the ``CurrentValuePropertyCheck``
that checks that a ``Stored`` property step has the right value this includes:

* The ``IStoredProperty`` to check the value of.
* A string that allows us to give the verification a name to identify it by. This is genarally a recommended thing to do.
* The expected value.
* An equality comparer to check that the value is right, where the default null will be replaced with ``EqualityComparer.Default``.

Then the ``Verify`` method checks the condition and returns one or more verification results.

The extension method is slighly different from the one used for steps. For one thing there is no chaining
going on through a ``SetNextStep`` method. Just use the interface exposed as a 'this' parameter, add a ``VerificationGroup``, use the
former to create the check instance and the latter to make the check available from the group. Then it can just return
the access interface again if we want to attach more checks.

Something like this:

.. sourcecode:: csharp

    public static IStoredProperty<TValue> CurrentValueCheck<TValue>(
        this IStoredProperty<TValue> property,
        VerificationGroup collector,
        string name,
        TValue expectedValue,
        IEqualityComparer<TValue> comparer = null)
    {
        collector.Add(new CurrentValuePropertyCheck<TValue>(property, name, expectedValue, comparer));
        return property;
    }

Writing a new logging context
=============================

This has got to be a very rare occurrance. Given that the ``Log`` steps are mainly there to aid in debugging your mocks, the default
behaviour to just write log statements to the console is normally good enough.

If you need to write them to somewhere else, such as to an xUnit ``ITestOutputHelper``,
you can pass an ``Action<string>`` to the ``WriteLineLogContext`` constructor and pass that to the ``Log`` steps.

However if you need to do more advanced stuff, such as logging mock interactions as structured data you can create a bespoke
implementation of the ``ILogContext`` interface. This interface has individual methods for all different logging calls made by Mocklis.
Implementing it should be a straightforward, if boring, exercise, and you can always look at the source code for the
`Mocklis.Serilog2` for an example of how it can be done.
