
FRLua
=====

FRLua is a library inspired by Bacon.js to provide Functional Reactive programming capabilities in Lua.
It is targeted at luajit 2.1 and lua >=5.1 <5.4.
It is currently implemented in pure lua.
This is version 0.1.2 of the library.  This package uses semver.

Most of the API is very similar to that of Bacon.js.

##Observables
There are two main types of object provided by this library: EventStreams and Properties.
Both EventStreams and Properties are Observables.
Properties have a concept of a current value; EventStreams do not.

All provided Observables will only perform a calculation when something is using the value they produce.
This is implemented by having them automatically unsubscribe from their data sources when their last subscriber unsubscribes.

##Events
There are four types of events that are propagated by the provided observables: Initial, Next, Error, and End.

They are simply identified by the strings "Initial", "Next", "Error", and "End"

When a new subscriber connects to a Property, it will recieve an Initial event with the current value of the Property.

Next events contain a new value of a property or an Event in an EventStream. Next events must always be dispatched to all subscribers of an Observable.

Error events are dispatched when an error occurs in an EventStream or Property. They may have anything that may be used as an error in lua, which is to say anything, but usually a string.

End events are dispatched when an Observable ends. It should not emit any more events after the End Event. Preferably an Observable should become eligible for garbage collection soon after the End event.

##API

###Methods

`observable:map(func)` creates a new Observable that produces values that are the result of applying func to the values produced by the source observable.
func can be anything allowed by the function construction rules. 

`observable:onValue(func)` Invokes the provided function with every value produced by the observable. The function can be anything allowed by the function construction rules. It returns a function that removes the handler when called.

`observable:each(func)` synonym for onValue

`observable:onError(func)` Invokes the provided function with every error produced by the observable. The function can be anything allowed by the function construction rules. It returns a function that removes the handler when called.

`observable:onEnd(func)` Invokes the provided function at the end of the observable. The function can be anything allowed by the function construction rules. It returns a function that removes the handler when called.

`observable:mapError(func)` Converts Error events into next events by passing the associated value to the provided function. The function construction rules apply.

`observable:Not()` creates an observable that is equivalent to `observable:map(function(val) return not val end)`

`prop:combine(prop2, func)` Combines the properties into a single property with the provided function. The function can be anything allowed by the function construction rules.

`prop:And(prop2)` combines the properties with and.

`prop:Or(prop2)` combines the properties with or.

`prop:Xor(prop2)` combines the properties with xor.

`prop:eq(prop2)` combines the properties with `==`.

`prop:ne(prop2)` combines the properties with `~=`.

`prop:gt(prop2)` combines the properties with `>`.

`prop:lt(prop2)` combines the properties with `<`.

`prop:ge(prop2)` combines the properties with `>=`.

`prop:le(prop2)` combines the properties with `<=`.

`eventstream:filter([func])` Filter the EventStream by the given predicate function. The function can be anything allowed by the function construction rules. If no function is provided the identity predicate is used, which falls back to lua's built in behavior. Everything except false and nil is regarded as true.

`eventstream:flatMap(func)`  Map the events into other event streams, then merge the resulting event streams into a single event stream. The func can be anything allowed by the function construction rules.

`eventstream:skipDuplicates([eql])` Removes events that are equal to the previous event according to the provided equality predicate. If no predicate is provided the default `==` operator is used.

`eventstream:zip(evenstream2, func)` Pair the events and invoke func with the values. Func can be anything allowed by the function construction rules. This method buffers the events from the streams until it has a corresponding event in the other stream, so if the streams emit at different rates, the buffer can get very large.

`bus:push(val)` pushes the given value into the bus as a Next event.

`bus:error(err)` pushes the given error into the bus.

`bus:End([val])` pushes an End event into the bus with optional data.

`bus:event(event, data)` pushes an event into the bus.

`bus:plug(stream)` plugs a stream into the bus. Every event except the End event from the stream will be emitted by the bus. It returns a function that unplugs the stream.

###Functions

####Creation

`FR.fromBinder(func)` Creates an EventStream from a binder function. The binder function accepts a single argument: a function sink. The binder is expected to register an event handler in such a way that the sink function gets called with the event and the value. It should return an unsubscribe function to be called when the events aren't needed.

`FR.once(value)` Creates an EventStream that emits the single value once then ends.

`FR.Repeat(func)` Creates an EventStream that Sequentially emits all of the events from the EventStreams generated by the function. The function is called with the iteration index.

`FR.never()` creates an EventStream that ends immediately.

`FR.Bus()` Create a Bus. A bus is an EventStream with additional methods to push events into it and to plug EventStreams into it.

`FR.fromCallback(func)` Invokes the provided function with a callback and creates an EventStream that emits the values passed to the callback.

`FR.constant(value)` Creates a Property that holds the constant value.

####Manipulation

`FR.combineAsArray(props)` Accepts an arraylike table of Properties and returns a Property that has the value of an arraylike table of the values of the sources.

`FR.combineTemplate(template)` Accepts a template table and creates a property that generates tables that are based off of the template but with every property in the template replaced by its value.
The template can contain nested tables which are also traversed and substituted. The traversal is performed with `pairs`, so it will obey any metamethods that `pairs` does. The template is not expected to change once it has been passed to this function.

####Miscellaneous

`FR.isEventStream(obj)` Returns true iff the object is an EventStream.

`FR.isProperty(obj)` Returns true iff the object is a Property.

`FR.isObservable(obj)` Returns true iff the object is an Observable.

####Function Construction Rules

Several functions have a signature of "Anything allowed by the function construction rules". The function construction rules are as follows:

A function is specified as a function constructor followed by an optional series of arguments for it to be curried with.

If the function constructor is a function, then the result function is either exactly that function or a function which calls the provided function with curried arguments.

If the function constructor is a string, then the string is interpreted to specify a function based on the first character of the string.

If the first character of the string is ":" then the remainder of the string is interpreted as a method name. The method can be curried with subsequent arguments, but the self argument is always taken from the live arguments rather than the curried arguments.

If the first character of the string is ";" then the remainder of the string is interpreted as a method name. The method can be curried with subsequent arguments, and the self argument will be taken from the curried arguments if available.

If the first character of the string is "." then the remainder of the string is used as a field name. Any subsequent arguments are ignored. The generated function will index its argument object for the given field.

If the first character of the string is "(" then the remainder of the string is used as a field name containing a function. Subsequent arguments are curried. The object provided as the first argument to the generated function is indexed for the given key and the result is called with the given arguments.

If the first character of the string is "'" then the remainder of the string is used as a constant. If the rest of the string is empty, then if there is another argument it is used as a constant. otherwise, the empty string is passed as a constant.

If the string is "[" then the generated function performs indexing. If an additional argument is provided, it is used as the index.

If the string is "]" then the generated function performs reversed indexing. The index is the first argument and the object is the second. If an additional argument is provided, it is used as the object.

Any other type of value is wrapped as a constant.
