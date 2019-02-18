=================
Extending Mocklis
=================

How Mocklis works
=================

An interface in C# can contain four different types of members: events, methods, properties and indexers. However
each of them is just syntactic suger over one or two method calls. In Mocklis we represent each of them with a
generic interface which encapsulates these method calls.

.. sourcecode:: csharp

    public interface IEventStep<in THandler> where THandler : Delegate
    {
        void Add(IMockInfo mockInfo, THandler value);
        void Remove(IMockInfo mockInfo, THandler value);
    }

    public interface IIndexerStep<in TKey, TValue>
    {
        TValue Get(IMockInfo mockInfo, TKey key);
        void Set(IMockInfo mockInfo, TKey key, TValue value);
    }

    public interface IMethodStep<in TParam, out TResult>
    {
        TResult Call(IMockInfo mockInfo, TParam param);
    }

    public interface IPropertyStep<TValue>
    {
        TValue Get(IMockInfo mockInfo);
        void Set(IMockInfo mockInfo, TValue value);
    }

Ignoring the IMockInfo parameter for the moment, these represent what the different member types do, except these
are modelled as if indexers always have `one` key, methods always `one` parameter and `one` result. Thanks to the
value tuple feature we can pretend that multiple values are one.

Now it's just a question of transforming any interface member into one of these four standard forms, and this is
done by the code generated my Mocklis.

Each member of each interface we mock out is itself implemented explicitly (read: out of the way), and a `mock property`
is added with the same name (if possible, otherwise a sequence number is added). The mock property is called from the
explicit member implementation, where any transformations also take place.

See for instance the following interface / mock implementation pair:

.. sourcecode:: csharp

    public interface ISample
    {
        double MonthlyPayment(double loanSize, double interestRate, int numberOfMonths);
        bool Parse(string text, out int value);
    }

    [MocklisClass]
    public class MockSample : ISample
    {
        public MockSample()
        {
            MonthlyPayment = new FuncMethodMock<(double loanSize, double interestRate, int numberOfMonths), double>(this, "MockSample", "ISample", "MonthlyPayment", "MonthlyPayment");
            Parse = new FuncMethodMock<string, (bool returnValue, int value)>(this, "MockSample", "ISample", "Parse", "Parse");
        }

        public FuncMethodMock<(double loanSize, double interestRate, int numberOfMonths), double> MonthlyPayment { get; }

        double ISample.MonthlyPayment(double loanSize, double interestRate, int numberOfMonths) => MonthlyPayment.Call((loanSize, interestRate, numberOfMonths));

        public FuncMethodMock<string, (bool returnValue, int value)> Parse { get; }

        bool ISample.Parse(string text, out int value)
        {
            var tmp = Parse.Call(text);
            value = tmp.value;
            return tmp.returnValue;
        }
    }

When MonthlyPayment is called, it wraps the parameters sent to it into a single argument to the `Call` method on the `FuncMethodMock` property.

With out parameters we need to work a little harder. The mock property now has only one in parameter but two out parameters (again wrapped in
a value tuple). It's up to the generated code to juggle the pieces to the right places. Ref parameters are *both* sent as parameters to the
mock property, and received back in the result.

If there are no parameters, something akin to `void` would be useful. The closest thing we have is the non-generic `ValueTuple` struct, which
to all intents an purposes is an empty declaration.

If you look at the constructor of the example above, each of the mock properties take a couple of parameters, a reference to the mock instance
itself, and a couple of strings with the name of the mock class, the names of the interface and member, and the name of the mock property (which
often but not always is the same as the name of the member). This is exactly what can be found in the IMockInfo interface that is on every
call on every I-membertype-Step interface. Steps can take advantage of this information if they want to.

Ok - so the system under test uses a member on a mocked-out interface. The mocklis class implement the interface and forwards the call on to
a mock parameter of the right type. Then what?

The mock parameter implements another interface. The method version looks like this, and the others are similar.

.. sourcecode:: csharp

    public interface ICanHaveNextMethodStep<out TParam, in TResult>
    {
        [EditorBrowsable(EditorBrowsableState.Never)]
        TStep SetNextStep<TStep>(TStep step) where TStep : IMethodStep<TParam, TResult>;
    }

This means that anything implementing IMethodStep can be sent to anything implementing ICanHaveNextMethodStep as its 'next' step. Since this new
step is returned, we could add another step after that, provided the step we added also implements the ICanHaveNextMethodStep interface.

A step therefore accepts calls, potentially does something, and potentially forwards on to subsequent steps.

Part of the contract for a non-final step is that if they aren't assigned any furthes steps to pass on calls to,
they should behave as if they were given a `Missing` step. The following would return the value 120 once,
and from then on act as if the mock wasn't configured.

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.TotalLinesOfCode
        .ReturnOnce(120);

The exception for the second call would look something like:

    *Mocklis.Core.MockMissingException: No mock implementation found for getting the value of Property 'ISample.TotalLinesOfCode'. Add one using 'TotalLinesOfCode' on your 'MockSample' instance.*

If we take another look at this last code sample, we notice that we do not call `SetNextStep` anywhere. In fact you will very rarely (if ever) see
these calls in your test code. The reason is that they're hidden in extension methods looking something along these lines:

.. sourcecode:: csharp

    public static ICanHaveNextPropertyStep<TValue> ReturnOnce<TValue>(
        this ICanHaveNextPropertyStep<TValue> caller,
        TValue value)
    {
        return caller.SetNextStep(new ReturnOncePropertyStep<TValue>(value));
    }

Every step in Mocklis is paired with one or more such extension methods. They are normally exectly this straightforward - pass on any parameters
to the step constructor, and chain in the new step via the `SetNextStep` method. They sometimes return the step itself as an out parameter, and
in the case of `final` steps (where the extension method would normally have the return type `void`) we have the opportunity to return something
else. For the `Stored` property steps, an IStoredProperty interface is returned which we can use to modify the stored value directly or add
validation checks.

As a last note, since method calls can have zero (or more) parameters and a void (or non-void) return type, we end up with effectively four different
types of methods - nothing->nothing, nothing->something, something->nothing and something->something. To keep the mock class a little more readable
there are therefore four different method mock types, that all implement the `ICanHaveNextMethodStep` interface. There are also cases where the steps
themselves come in different flavours depending on whether there are parameters and/or return types. The trick used by Mocklis is to represent a
missing type with ValueTuple, but that means that there might be more than one valid step to use.


Writing new steps
=================

The best way to learn about writing steps is to look at the source code for existing steps. But in the interest of documentation, here's a sample.

Disclaimer: This is a silly example. You would never write test code that depends on the time of day. It was chosen because you can be absolutely
certain that this step won't ever clash with anything in the Mocklis libraries themselves. (We sincerely hope...)

Phase 1: Write a step
---------------------

If you are writing a `final` step, implement the I-memberType-Step interface. You just need to implement this interface and you're done.

If you are writing a non-final step, consider (as in it is very strongly recommended) subclass the memberType-StepWithNext class, and override
the I-memberType-Step members as you see fit. If you don't override them you'll just forward the call on, and if you override them you can
use the `base` implementation to forward the call on.

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

If you wanted to use the new step as is, you would have to create an instance of it and feed to the `SetNextStep` method of the previous
step. To enable the fluent syntax you'll need to add the step as an extension method on the IPropertyStepCaller interface.

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

Notice the naming convention: The EndOfDayPropertyStep is added as an extension method named EndOfDay, taking an IPropertyStepCaller as its `this`
parameter. An EndOfDayMethodStep would be added as an extension method also named EndOfDay. There is no risk of a naming clash, as the parameter
types will differ.

Now you can use your new step:

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.TotalLinesOfCode
        .EndOfDay();

With the obvious (well - depending on what time it is) result:

    *System.Exception: It's late - start considering calling it a day.*

Phase 3: Generalise
-------------------

The last phase is to look at your newly created step and consider whether it can be used in other situations. You should extend
the step to the different member types if possible.

In some cases the way a step works could depend on the complete state of the mock instance. In these cases you should add new
steps with the same name as your existing ones, but prefixed with `Instance`. For this version you pass on the IMockInfo.Instance
to the construct you have that uses the instance. Look at the existing `Lamdba` steps for the quintessential implementation, however
the `Record` and `If` steps also have instance versions.

If you work with steps for methods, you might need to consider having different versions depending on whether your
methods take parameters or not, and whether they return things or not. For the `lamdba` steps there are two `FuncMethodStep` classes,
and two `ActionMethodStep` classes.

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

Verification is one of the least 'polished' parts of Mocklis (and that's saying something...)

The idea is to create a tree of binary checks that can be verified in one go. When verified, a read-only (and recursive) data structure is created,
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

Verifications can either be written as steps. These steps implement the `IVerifiable` interface, and the
extension method takes a VerficationGroup as a parameter and attach the created step to that group.

Let's say we're creating a Method step to check that the method has indeed been called. Subclass MedialMethodStep,
override Call to set a flag that it has been called, and implement IVerifiable to return a VerificationResult.

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

But we may want to check some condition without it being a step in its own right. All the `Stored` steps
(which would be property, indexer and event) implement an interface to directly access what is being
stored. An implementation of IVerifiable that is not a `step` in its own right is called a `check`, and
writing one is straightforward:

Create a class, have it implement IVerifiable. Let the constructor take as input anything it needs to
verify that the condition for the verification has been met. In the case of the `CurrentValuePropertyCheck`
that checks that a `stored` property step has the right value this includes:

* The IStoredProperty to check the value of.
* A string that allows us to give the verification a name to identify it by. This is genarally a recommended thing to do.
* The expected value
* An equality comparer to check that the value is right, where the default null will be replaced with `EqualityComparer.Default`.

Then the Verify method checks the condition and returns one or more verification results.

The extension method is slighly different from the one used for steps. For one thing there is no chaining
going on through a `SetNextStep` method. Just use the interface exposed as a `this` parameter, add a VerificationGroup, use the
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

This has got to be a very rare occurrance. Given that the log steps are mainly there to aid in debugging your mocks, the default
behaviour to just write log statements to the console is normally good enough.

If you need to write them to somewhere else, passing an instance of the `TextWriterContext` to the step linked up to an
appropriate `TextWriter` is also a possibility.

However if you need to do more advanced stuff, such as logging mocklis events as structured data (to things like Serilog)
you can implement the ILogContext. This interface has individual methods for all different logging calls made by Mocklis.
Implementing it should be a straightforward, if boring, exercise, and you can always look at the source code for the
`Mocklis.Serilog2` for an example of how it can be done.
