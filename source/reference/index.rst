=========
Reference
=========

The reference section consists of four parts: standard steps, verifications, code generation and experimental
stuff.

Standard Steps
==============

Find below a discussion of all the steps in the standard Mocklis package that you can use. The list is, perhaps
unfortunately, in alphabetic order. This means that some of the more common steps are listed towards the end,
but it makes a little bit more sense as a reference section in this way.

Conditional steps
-----------------

The conditional steps are steps that either branch out or cut short the invocation of a mocked member
based on some condition.

The ``If`` steps branch to a different chain of steps if a given condition holds, with the option of
joining the original remaining steps. In their basic form the decision is based on just information passed
to the step, parameters to a method or key to an indexer. The ``InstanceIf`` step lets you make the decision
based on the state of the whole mocked instance, and the ``IfAdd``, ``IfRemove``, ``IfGet`` and ``IfSet`` versions branch
only for those actions of an event, property or indexer.

Here's a mock setup for an indexer. Note the underlying name used for the indexer, which is used since it would
not be well-formed C# to name a property `this[]`. The underlying name is usually `Item` - which is the reason
why it's not possible (unless the indexer name has been changed via the ``IndexerNameAttribute``) to have
a method named ``Item`` in a class with an indexer.

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.Item
        .If(g => g % 2 == 0, null, b => b.Return("Even"))
        .Return("Odd");

The ``null`` argument is for a parameter that decides when to use the if-branch when writing to the indexer.

``If`` steps provide you with a `joinpoint` representing the non-if branch (called 'ElseBranch'). In the following
sample we have a property, where both the getter and setter are connected to the same ``Stored`` step. Only calls
to the setter are logged to the console, however.

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.TotalLinesOfCode
        .IfSet(b => b.Log().Join(b.ElseBranch))
        .Stored(200);

The last sample for a conditional step uses the ``OnlySetIfChanged`` conditional which only exists for properties
and indexers. When an attempt to 'set' a value is made, the step will first try to 'get' the value, check if it's
actually changed, and only 'set' the new value if it has.

.. sourcecode:: csharp

    var mock = new MockSampleWithNotifyPropertyChanged();
    mock.PropertyChanged
        .Stored(out var pch);
    mock.TotalLinesOfCode
        .OnlySetIfChanged()
        .RaisePropertyChangedEvent(pch)
        .Stored();

The combination of the ``OnlySetIfChanged``, ``RaisePropertyChangedEvent`` and ``Stored`` steps is so common that there
is a shorthand: ``StoredWithChangeNotification``

.. sourcecode:: csharp

    var mock = new MockSampleWithNotifyPropertyChanged();
    mock.PropertyChanged
        .Stored(out var pch);
    mock.TotalLinesOfCode
        .StoredWithChangeNotification(pch);


Dummy steps
-----------

The ``Dummy`` steps will do as little as possible without throwing an exception. For a property or indexer, the
step will do nothing for a setter, and return a default value for a getter. For an event, adding or removing
an event handler do absolutely nothing, and for a method, it will not do anything with the parameters, and return
default values for anything that needs returning, including out and ref parameters.

Note also that ``Dummy`` steps are final - you cannot add anything to follow them.


Join steps
----------

We've already met the ``Join`` step in the sample code for ``If`` above, where it allows us to take any step (with
the right form - that is member type and type parameters) and use as the next step. The missing piece is a method
to designate a step as such a target, which is where the ``JoinPoint`` comes in.

Let's say that we want to connect two properties to the same ``Stored`` step. The solution is to add a ``JoinPoint``
step just before the ``Stored`` step.

.. sourcecode:: csharp

    var mockDishes = new MockDishes();
    mockDishes.Vichyssoise.JoinPoint(out var soup).Stored();
    mockDishes.Revenge.Join(soup);

    IDishes dishes = mockDishes;

    dishes.Vichyssoise = "Best served cold";
    Console.WriteLine(dishes.Revenge);

Note that any step would do for a ``Join``, as long as we can get hold of it. The following would work equally well, taking
the ``Stored`` step itself and using that as a join point:

.. sourcecode:: csharp

    var mockDishes = new MockDishes();
    mockDishes.Vichyssoise.Stored(out var soup);
    mockDishes.Revenge.Join(soup);

Lambda steps
------------

These steps are constructed with either an ``Action`` or a ``Func``, and when they are called the ``Action`` or ``Func`` will be
run. In the case of ``Func`` the result of the call will be returned.

The names always contain the word ``Action`` or the word ``Func``, but they are further qualified for non-method steps. Property
and indexer steps are called ``GetFunc`` and ``SetAction`` while event steps are called ``AddAction`` and ``RemoveAction``.

The lambda steps (and some of the other steps) have 'instance' versions where the current instance of the mock
is passed as an additional parameter. This parameter is always untyped (well, passed as object), so you'll need
to cast it to one of the mocked interfaces (or the mocking class itself) for it to be of any use. These steps have the names of
their non-instance counterparts prefixed with the word ``Instance`` (so that ``InstanceSetAction`` would exist as a
property step to give an example).

Here's an example where a ``Send`` method takes a message of some reference type and returns a ``Task``:

.. sourcecode:: csharp

    var mockConnection = new MockConnection();
    mockConnection.Send.Func(m => m == null
        ? Task.FromException(new ArgumentNullException())
        : Task.CompletedTask);

Log steps
---------

``Log`` steps are your quintessential debugging steps. They won't do anything except write out anything that
passes through them, by default to the console althought this can be tailored to your specific needs.

Therefore you can just add in a ``.Log()`` if you need to figure out what happens with a given mock. Note that they are best
added early in a mock step chain if you want to get a faithful representation of what's being called from the code you
are testing, as steps can short-circuit calls or make calls of their own down the chain.

The :doc:`../getting-started/index` makes extensive use of ``Log`` steps.

If you're working with Xunit as your test framework, you probably know that you cannot write to the Console and expect
the strings written to be part of the test output, and that instead your test class accepts an ``ITestOutputHelper``
on the constructor. The recommended approach is to have your test class implement the ``ILogContextProvider`` interface.

.. sourcecode:: csharp

    public class Tests : ILogContextProvider
    {
        public ILogContext LogContext { get; }

        protected Tests(ITestOutputHelper testOutputHelper)
        {
            LogContext = new WriteLineLogContext(testOutputHelper.WriteLine);
        }
    }

Then you can replace all your calls to `Log()` with `Log(this)`, and the lines will be written to the ``ITestOutputHelper``.

There is also a specific LogContext for Serilog 2.x if you add the ``Mocklis.Serilog2`` NuGet package. Create a ``SerilogContext``
with an ``ILogger`` that has a test-framework compatible sink, and set LogContext to that as in the example above.

Miscellaneous steps
-------------------

Stuff that couldn't really be placed in an existing category, and would have constituted a 'one-step-only' category if
pushed...

Currently this (possibly expanding) category contains just the ``RaisePropertyChangedEvent`` step you saw in the last example
of the Conditional steps category.

Missing steps
-------------

When one of these steps is invoked, it will throw a ``MockMissingException`` with information about the `mock property` itself.

The exception thrown could look something like this:

    *Mocklis.Core.MockMissingException: No mock implementation found for getting value of Property 'ISample.TotalLinesOfCode'. Add one using 'TotalLinesOfCode' on your 'MockSample' instance.*

Record steps
------------

These steps will keep track of all the calls that have been made to them, so that you can assert in your tests that the
right interactions have happened.

Each of the record steps will cater for one type of interaction only (method call, indexer get, indexer set, property
get, property set, event add or event remove), and it will take a ``Func`` that transforms whatever is seen by the step
to something that you want to store. They also provide the 'ledger' with recorded data as an out parameter.

There is currently no mechanism for letting record steps share these 'ledgers' with one another.

.. sourcecode:: csharp

    [Fact]
    public void RecordAddedEventHandlers()
    {
        // Arrange
        var mockSamples = new MockSampleWithNotifyPropertyChanged();
        mockSamples.PropertyChanged.RecordBeforeAdd(out var handlingTypes, h => h.Target?.GetType());

        // Act
        ((INotifyPropertyChanged)mockSamples).PropertyChanged += OnPropertyChanged;

        // Assert
        Assert.Equal(new[] { typeof(RecordSamples) }, handlingTypes);
    }

Repetition steps
----------------

The ``Times`` steps look a little like conditional steps in that they add a separate step chain that can be taken. They
differ from the if-step in that they cannot join back to the normal path, and that the separate path will only be used
a given number of times.

In the current version a get or a set both count as a usage from the same pool for property and indexer mocks, as do
adds and removes for an event mock.

For a sample see the next section, return steps.

Return steps
------------

Arguably the most important step of them all. The ``Return`` step, only useable in cases where some sort of return value is
expected, will simply return a value.

There are three versions, one that just returns a given value once, and passes calls on to subsequent steps on later calls,
one that returns items from a list one by one, and one that returns the same value over and over.

Here's code that shows how to use these, and the repetition step:

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.GuessTheSequence
        .Times(2, m => m.Return(1))
        .ReturnOnce(int.MaxValue) // should really be infinity for this sequence
        .ReturnEach(5, 6)
        .Return(3);

    var systemUnderTest = (ISample)mock;

    Assert.Equal(1, systemUnderTest.GuessTheSequence);
    Assert.Equal(1, systemUnderTest.GuessTheSequence);
    Assert.Equal(int.MaxValue, systemUnderTest.GuessTheSequence);
    Assert.Equal(5, systemUnderTest.GuessTheSequence);
    Assert.Equal(6, systemUnderTest.GuessTheSequence);
    Assert.Equal(3, systemUnderTest.GuessTheSequence);
    Assert.Equal(3, systemUnderTest.GuessTheSequence);
    Assert.Equal(3, systemUnderTest.GuessTheSequence);
    Assert.Equal(3, systemUnderTest.GuessTheSequence);

Stored steps
------------

If the ``Return`` steps are the most used steps, the ``Stored`` steps are definitely the first runners up. These steps are defined
for properties, playing backing field to the mocked property. They are also defined for indexers, where the backing structure
is a dictionary which has the default return value for all non-set keys.

When creating a ``Stored`` step for a property you can give it an initial value, and for both properties and indexers you can use
verifications to check that the stored value has been set correctly by the components that are under test.

``Stored`` steps are also used with events where the steps act as storage for added event handlers. If you have a reference to the
``Stored`` step you can raise events on these handlers, simply by calling ``Invoke`` on the stored value. Alternatively,
if your handler type is a generic ``EventHandler<>`` or one of a handful of very common event
handler types including ``PropertyChangedEventHandler`` and the basic ``EventHandler``, the Mocklis library
provides you with ``Raise`` extension methods. These can be found in the ``Mocklis.Verification`` namespace.

.. sourcecode:: csharp

    [Fact]
    public void RaiseEvent()
    {
        var mock = new MockSample();
        mock.MyEvent.Stored<EventArgs>(out var eventStep);
        bool hasBeenCalled = false;

        ISample sample = mock;
        sample.MyEvent += (s, e) => hasBeenCalled = true;

        eventStep.Raise(null, EventArgs.Empty);
        // equivalent: eventStep.EventHandler?.Invoke(null, EventArgs.Empty);
        Assert.True(hasBeenCalled);
    }

For indexers the step is called ``StoredAsDictionary`` as it holds different values for different keys. It will return a default
value rather than throw if an empty slot is read from.

Throw steps
-----------

Super easy - with these steps you provide a ``Func`` that creates an exception. When called, the step will call
this ``Func`` and throw the exception it returns.


Verification steps
------------------

Verification steps are steps that track some condition that can be checked and asserted against.

``ExpectedUsage`` steps take a verification group as a parameter, along with the number of time they
expect the mocked member to be called (which are tracked individually for getters, setters, adds, removes and plain method calls).

To get access to all steps and checks (see next section) for verifications you need to have the namespace ``Mocklis.Verification``
in scope via a using statement at the top of your file.

Verifications
=============

If steps provide a means of creating behaviour for the system under test, verifications provide a means of checking that those
behaviours have been used in the right way by the system under test.

Verifications come in two flavours. As normal steps they check data as it passes through them:

.. sourcecode:: csharp

    var vg = new VerificationGroup();
    var mock = new MockSample();
    mock.DoStuff
        .ExpectedUsage(vg, "DoStuff",  1);

    ...

    vg.Assert();

... and also as 'checks' that verify some condition of an existing step:

.. sourcecode:: csharp

    [Fact]
    public void JustChecks()
    {
        var vg = new VerificationGroup();
        var mock = new MockSample();
        mock.TotalLinesOfCode
            .Stored(50)
            .CurrentValueCheck(vg, "TLC", 60);

        ISample sample = mock;
        sample.TotalLinesOfCode = 60;

        vg.Assert();
    }

These are the only verifications in the framework at the moment. The expected usage steps work for all different member types,
and track the different access methods independently. The current value checks exist for properties and indexers only, where
the latter takes a list of key-value pairs to check.

To check that verifications have been met, call ``Assert`` on the top-most verification group, as done in the last example.

Mocklis Code Generation
=======================

An interface in C# can contain four different types of members: events, methods, properties and indexers. However
each of them is just syntactic suger over one or two method calls. In Mocklis we represent each with a
generic interface that encapsulates these method calls.

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

Ignoring the IMockInfo parameter for the moment, these represent what the different member types do, except they
are modelled as if indexers always have *one* key, methods always *one* parameter and *one* result. Thanks to the
value tuple feature we can pretend that multiple values are one.

Now it's just a question of transforming any interface member into one of these four standard forms, and this is
done by the code generated my Mocklis.

In the next few sections we'll go over the different things that Mocklis can generate for us. There are a number
of corner cases in particular to do with naming, and we won't go over all of them here. For a reasonably complete
set of cases see the ``Mocklis.MockGenerator.Tests`` project in the Mocklis source code.

Event mocks
-----------

The simplest (as in has the fewest special cases) thing to implement is events. An event has to be of a delegate type
and is always represented by an Add/Remove pair of methods, which is exactly what the IEventStep interface models.

Let's say that we have an interface with an event.

.. sourcecode:: csharp

    public interface ITestClass
    {
        event EventHandler MyEvent;
    }

The generated code for such an interface consists of three parts. The explicitly implemented event itself just forwards adds
and removes to an EventMock, which is itself creted in the constructor and exposed as a property.

.. sourcecode:: csharp

    [MocklisClass]
    public class TestClass : ITestClass
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        public TestClass()
        {
            MyEvent = new EventMock<EventHandler>(this, "TestClass", "ITestClass", "MyEvent", "MyEvent", Strictness.Lenient);
        }

        public EventMock<EventHandler> MyEvent { get; }

        event EventHandler ITestClass.MyEvent { add => MyEvent.Add(value); remove => MyEvent.Remove(value); }
    }

That's really all there is to it. A common question is how to raise events. The fact that an event itself has little to do with
raising events is a common C# 'gotcha'. The event is only about combining event handlers through the ``add`` and ``remove`` accessors.
To raise an event you need to call `Invoke` on the resulting combined handler. In Mocklis you need to add a ``Stored`` step to an
event in order to correctly remember and combine handlers so you have something to raise the event on. Then you can use the handler
exposed by the ``Stored`` step. See the documentation for a ``Stored`` step above for a complete example.

Events can be generic-ish. For some reason it's not possible to have an event of type parameter type, even if that type parameter is
contrained to delegate type. But you can have an event of a generic delegate type, such as EventHandler<T>.

Property mocks
--------------

Like an event, a property has two accessor methods. In this case one to get a value, and one to set a value. Unlike an event you do
not need to use both of them, as a property can be readonly or writeonly. The generated `mock property` doesn't make a distinction, and
the generated code for an interface with three string propertis with different access looks like this:

.. sourcecode:: csharp

    [MocklisClass]
    class TestClass : ITestClass
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        public TestClass()
        {
            ReadOnly = new PropertyMock<string>(this, "TestClass", "ITestClass", "ReadOnly", "ReadOnly", Strictness.Lenient);
            WriteOnly = new PropertyMock<string>(this, "TestClass", "ITestClass", "WriteOnly", "WriteOnly", Strictness.Lenient);
            ReadWrite = new PropertyMock<string>(this, "TestClass", "ITestClass", "ReadWrite", "ReadWrite", Strictness.Lenient);
        }

        public PropertyMock<string> ReadOnly { get; }

        string ITestClass.ReadOnly => ReadOnly.Value;

        public PropertyMock<string> WriteOnly { get; }

        string ITestClass.WriteOnly { set => WriteOnly.Value = value; }

        public PropertyMock<string> ReadWrite { get; }

        string ITestClass.ReadWrite { get => ReadWrite.Value; set => ReadWrite.Value = value; }
    }

Properties can be generic. Properties can also be of `restricted type`, in which case the generated code will fall back to
virtual methods and you'll need to subclass and override to add behaviour rather than using steps.

.. sourcecode:: csharp

    public interface ITestClass
    {
        Span<string> SpanProperty { get; set; }
    }

    [MocklisClass]
    class TestClass : ITestClass
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        protected virtual Span<string> SpanProperty()
        {
            throw new MockMissingException(MockType.VirtualPropertyGet, "TestClass", "ITestClass", "SpanProperty", "SpanProperty");
        }

        protected virtual void SpanProperty(Span<string> value)
        {
            throw new MockMissingException(MockType.VirtualPropertySet, "TestClass", "ITestClass", "SpanProperty", "SpanProperty");
        }

        Span<string> ITestClass.SpanProperty { get => SpanProperty(); set => SpanProperty(value); }
    }

Indexer mocks
-------------

Indexers are, loosely speaking, properties with a parameter list, so most of the discussion for properties goes for indexers as
well. Even thought indexers are declared with the `this` keyword, they have an internal name (that can be changed with the
`IndexerName` attribute), and the default name is `Item`.

In mocklis an indexer has a getter with *one* parameter type, and a return type, and a setter with *one* parameter type and a value type,
but indexers can have more than one parameter. Here Mocklis uses `ValueTypes` to turn multiple types into one.

.. sourcecode:: csharp

    public interface ITestClass
    {
        string this[int row, int col] { get; set; }
    }

    [MocklisClass]
    class TestClass : ITestClass
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        public TestClass()
        {
            Item = new IndexerMock<(int row, int col), string>(this, "TestClass", "ITestClass", "this[]", "Item", Strictness.Lenient);
        }

        public IndexerMock<(int row, int col), string> Item { get; }

        string ITestClass.this[int row, int col] { get => Item[(row, col)]; set => Item[(row, col)] = value; }
    }

Note that the `mock property` uses the internal name of the indexer, it's not possible to expose a `this[]` property. Otherwise anything
that goes for a property also goes for an indexer.

Method mocks
------------

We're discussing methods last because these have the largest number of different cases, even thought the ``IMethodStep`` interface only
has one member. As for the indexer, condensing multiple parameters into one is done using `ValueTuples`. We cannot encode whether a
parameter is `in`, `out` or `ref` in the value tuple, so instead an `out` parameter is added to the return type, and a `ref` parameter
is added both as a normal parameter an as part of the return type. This means that we can have multiple return types, and again these
are combined into a single `ValueTuple`.

The canonical example is the `TryParse`. Notice that the mock takes a string and returns a (bool, int) pair. By naming the individual
types in the `ValueTuple` we get intellisense, and the name `returnValue` is given to the value returned from the mocked method.

.. sourcecode:: csharp

    public interface ITestClass
    {
        bool TryParse(string text, out int result);
    }

    [MocklisClass]
    class TestClass : ITestClass
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        public TestClass()
        {
            TryParse = new FuncMethodMock<string, (bool returnValue, int result)>(this, "TestClass", "ITestClass", "TryParse", "TryParse", Strictness.Lenient);
        }

        public FuncMethodMock<string, (bool returnValue, int result)> TryParse { get; }

        bool ITestClass.TryParse(string text, out int result)
        {
            var tmp = TryParse.Call(text);
            result = tmp.result;
            return tmp.returnValue;
        }
    }

Method mocks can also return values by reference, and like `out` or `ref` parameters, the information that the value is to be returned `by ref` isn't part
of the return type and thus cannot be encoded in the type parameters of the mock itself. Mocklis can handle this by treating the mock as if it was a normal
`non-ref` method, and then wrap the return value in an object so that we can return a reference at the last minute. Granted, this doesn't provide the performance
benefit that is sometimes looked for when using `ref` return values, but for tests this is usually good enough.

.. sourcecode:: csharp

    public interface ITestClass<in TKey, out TValue> where TValue : new()
    {
        ref readonly int GetRef();
    }

    [MocklisClass]
    class TestClass<T, U> : ITestClass<T, U> where U : new()
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        public TestClass()
        {
            GetRef = new FuncMethodMock<int>(this, "TestClass", "ITestClass", "GetRef", "GetRef", Strictness.Lenient);
        }

        public FuncMethodMock<int> GetRef { get; }

        ref readonly int ITestClass<T, U>.GetRef() => ref ByRef<int>.Wrap(GetRef.Call());
    }

The default is to treat `ref readonly` returns in this manner, while using the virtual method fallback for `ref` returns. This can be controlled with
attribute parameters as discussed in the :doc:`../usage/index` section.

Since method calls can have zero (or more) parameters and a void (or non-void) return type, we end up with four different
types of methods: nothing->nothing, nothing->something, something->nothing and something->something. To keep the mock class a little more readable
there are four different method mock types (in the example above `FuncMethodMock` was used) that all implement the ``ICanHaveNextMethodStep``
interface. There are also cases where the steps themselves come in different flavours depending on whether there are parameters and/or return types.
The trick used by Mocklis is to represent a missing type with ``ValueTuple``, but this also means that there might be more than one valid step to use.

As for properties and indexers, methods can use type parameters introduced by the interface they're defined in. But methods can also introduce type parameters
of their own. Since these type parameters won't be closed we cannot create a mock property directly with them - we would need to have individual properties of all
possible combinations of types. This is clearly impractical, so instead we create them as needed and store them in a dictionary keyed on the actual types
used for each instance.

The generated code therefore contains a mock factory method instead of a mock property.

.. sourcecode:: csharp

    public interface ITestClass
    {
        string Write<T>(T param);
    }

    [MocklisClass]
    class TestClass : ITestClass
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        private readonly TypedMockProvider _write = new TypedMockProvider();

        public FuncMethodMock<T, string> Write<T>()
        {
            var key = new[] { typeof(T) };
            return (FuncMethodMock<T, string>)_write.GetOrAdd(key, keyString => new FuncMethodMock<T, string>(this, "TestClass", "ITestClass", "Write" + keyString, "Write" + keyString + "()", Strictness.Lenient));
        }

        string ITestClass.Write<T>(T param) => Write<T>().Call(param);
    }

Constructors
------------

The mocks are initialised in the constructor. For a `MocklisClass` that doesn't derive from another class (that is to say that derives directly from `object`)
a default constructor will be added if there are mocks to initialise. The constructors are `protected` if the `MocklisClass` is declared as
abstract, and `public` otherwise.

If the `MocklisClass` does derive from another class, all public and protected constructors from that base class will be given a corresponding constructor
in the `MocklisClass`, passing on parameters as necessary.

If you look at the constructors in the examples given, each of the `mock properties` take a couple of parameters, a reference to the mock instance
itself, and a couple of strings with the name of the mock class, the names of the interface and member, and the name of the `mock property` (which
often but not always is the same as the name of the member). It also takes the strictness used when creating the `MocklisClass` so that it can react
correctly in the cases where the configuration is missing or incomplete. This is exactly what can be found in the ``IMockInfo`` interface that is on every
call on every I-membertype-Step interface. Steps can take advantage of this information if they want to; indeed the ``Missing`` step picks up
the information from this parameter to provide the best possible exception message for the user.

Name clashes
------------

There are cases where the generated code for mocks would clash with either each other or with identifiers already declared in base classes. In these cases
Mocklis will add a numerical suffix to the introduced identifier. To take a very simple example, ``IEnumerable<T>`` derives from ``IEnumerable``, and both
have a `GetEnumerator` method. The generatod code looks like the following, and unfortunately you have to know which method you want to add a step to and
use the corresponding name.

.. sourcecode:: csharp

    [MocklisClass]
    class TestClass : IEnumerable<int>
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        public TestClass()
        {
            GetEnumerator = new FuncMethodMock<IEnumerator<int>>(this, "TestClass", "IEnumerable", "GetEnumerator", "GetEnumerator", Strictness.Lenient);
            GetEnumerator0 = new FuncMethodMock<System.Collections.IEnumerator>(this, "TestClass", "IEnumerable", "GetEnumerator", "GetEnumerator0", Strictness.Lenient);
        }

        public FuncMethodMock<IEnumerator<int>> GetEnumerator { get; }

        IEnumerator<int> IEnumerable<int>.GetEnumerator() => GetEnumerator.Call();

        public FuncMethodMock<System.Collections.IEnumerator> GetEnumerator0 { get; }

        System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => GetEnumerator0.Call();
    }

Name clashes can also appear in the type parameter names, and in the types used to create a `ValueTuple`. When creating a `ValueTuple` we could clash with
the default names `Item1`, `Item2` and so forth, so Mocklis will rename these as well when needed.

Type parameter substitutions
----------------------------

The type parameter names declared by an interface, and the type parameter names used when referencing that interface do not need to be the same.
Mocklis does substitutions where necessary, and sorts out situations where there would be a name clash.

Here's a simple example - the interface calls the key TKey, but the class uses T, therefore all instances of TKey in the interface have been
replaced with T in the class, and the same goes for TValue and U.

.. sourcecode:: csharp

    public interface ITestClass<in TKey, out TValue>
    {
        TValue GetValue(TKey key);
    }

    [MocklisClass]
    class TestClass<T, U> : ITestClass<T, U>
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        public TestClass()
        {
            GetValue = new FuncMethodMock<T, U>(this, "TestClass", "ITestClass", "GetValue", "GetValue", Strictness.Lenient);
        }

        public FuncMethodMock<T, U> GetValue { get; }

        U ITestClass<T, U>.GetValue(T key) => GetValue.Call(key);
    }






Experimental Stuff
==================

Mocklis has a project & associated NuGet package for experimental things: ``Mocklis.Experimental``. It is meant for things that are
in a bit of flux and may either graduate to the main ``Mocklis`` package, or be found wanting and deleted. Think of it as an incubation
space for new functionality. It will not follow the versioning of the other Mocklis NuGet packages, but will stay perpetually pre-release.

At the moment it only contains the ``Gate`` step.

Gate steps
----------

The idea behind the ``Gate`` step is that it will complete a ``Task`` (as in Task Parallel Library), when the step is called. The ``Task`` can then be used to
drive other things happening in the step, effectively forcing a strict ordering of events in the face of many threads running.

The syntax is still very experimental - it currently only exists for 'Method' mocks, and might well be killed off altogether...

.. sourcecode:: csharp

    public async Task SuccessfulPing()
    {
        // Arrange
        var mockConnection = new MockConnection();
        mockConnection.Send
            .Gate(out var sendGate)
            .Return(Task.CompletedTask);
        mockConnection.Receive
            .Stored<MessageEventArgs>(out var messageReceive);
        var pingService = new PingService(mockConnection);

        // Act
        var ping = pingService.Ping();
        await sendGate;
        messageReceive.Raise(mockConnection, new MessageEventArgs(new Message("PingResponse")));
        var pingResult = await ping;

        // Assert
        Assert.True(pingResult);
    }
