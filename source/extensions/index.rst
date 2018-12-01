=================
Extending Mocklis
=================




How Mocklis works
=================








Writing new steps
=================






Part of the contract for medial step is that if they aren't assigned any furthes steps to pass on calls to,
they should behave as if they were given a `Missing` step. The following would return the value 120 once, 
and from then on act as if the mock wasn't configured.

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.TotalLinesOfCode
        .ReturnOnce(120);

The exception for the second call would look something like:

    *Mocklis.Core.MockMissingException: No mock implementation found for getting value of Property 'ISample.TotalLinesOfCode'. Add one using 'TotalLinesOfCode' on the 'MockSample' class.*




Writing new validation
======================






Writing a new logging context
=============================


