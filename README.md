# Local Context

    Stability: 1 - Experimental

Local contexts provide a mechanism similar to thread-local storage in
threaded programming. Contexts are created on namespaces and can be
be nested.

Every namespace is created with a default context. The currently active
context on a namespace is available via `namespace.active`.

```javascript
var context = require('context');

var writer = context.createNamespace('writer');
writer.set('value', 0);

function requestHandler() {
  var outer = writer.createContext();
  // writer.get('value') returns 0
  writer.set('value', 1);
  // outer.get('value') returns 1
  // writer.get('value') returns 1

  process.nextTick(function () {
    outer.run(function () {
      var inner = writer.createContext();
      // writer.get('value') returns 1
      writer.set('value', 2);
      // outer.get('value') returns 1
      // inner.get('value') returns 2
      // writer.get('value') returns 2
    });
  });

  setTimeout(function () {
    // runs with the default context, because nested contexts have ended
    console.log(writer.get('value')); // prints 0
  }, 1000);
}
```

## context.createNamespace(name)

* return: {Namespace}

Returns a new namespace for context-sensitive values.

## context.getNamespace(name)

* return: {Namespace}

Look up an existing namespace.

## process.namespaces

* return: array of {Namespace} objects

List of available namespaces.

## Class: Namespace

The Namespace groups local contexts, and encapsulates their nesting
behavior, as well as containing the associated keys and values set on
that namespace.

### namespace.active

Returns the currently active context on a namespace.

### namespace.createContext()

Create a new local context.

### namespace.set(key, value)

Set a value on the current local context. Shorthand for
`namespace.active.set(key, value)`.

### namespace.get(key)

Look up a value on the current local context. Recursively searches from
the innermost to outermost nested local context for a value associated
with a given key.

## Class: Context

A Context encapsulates the current nesting level and its associated
values. Contexts are derived from [EventEmitter][], and have a life
cycle (which, for now, consists of the `end` event when they're about to
go out of scope). Contexts are created exclusively via Namespaces.

### context.run(callback)

Run a function within the specified local context. Shortcut for
`context.bind(callback)()`.

### context.bind(callback)

Bind a function to the specified local context. Works analagously to
`Function.bind()` or `domain.bind()`.

### context.set(key, value)

Set a value on the specified local context.

### context.get(key)

Look up a value on *only* the specified local context -- doesn't fall
through to containing scopes.

## Rationale

The domains mechanism is a useful tool for adding context to errors
raised in asynchronous call chains (or, if you like living dangerously /
tempting the wrath of @isaacs, to recover from errors without restarting
your services). It also almost serves the purposes of developers
who want to annotate async call chains with metadata or extra state
(examples: logging, tracing / profiling, generic instrumentation),
but due to the needs of error-handling, it doesn't quite generalize
enough to be truly useful in this regard. There are modules that allow
developers to do similar things when they have full control over their
stacks ([CrabDude/trycatch](https://github.com/CrabDude/trycatch) and
[Gozala/reducers](https://github.com/Gozala/reducers), among many
others), but none of these modules are a good fit for developers writing
tooling meant to be dropped transparently into user code.

See also [joyent/node#3733](https://github.com/joyent/issues/3733).

Here is a sketch at what the user-visible API might look like. My
original attempt at this used a slightly modified version of the domains
API with some special-purpose logic for dealing with nested contexts,
but allowing multiple distinct namespaces is actually simpler and trades
memory for execution time. It also makes it possible to special-case
behavior for specific namespaces (i.e. my hope would be that domains
would just become a specialized namespace, and `_tickDomainCallback`
and `_nextDomainTick` would be all that would be required to deal with
namespaces), although that isn't included here.

Here's an example of how the API might be used:

```javascript
var context = require('context');

// multiple contexts in use
var tracer = context.createNamespace('tracer');

function Trace(harvester) {
  this.harvester = harvester;
}

Trace.prototype.runHandler = function (callback) {
  var trace = tracer.createContext();

  trace.on('end', function () {
    var transaction = trace.get('transaction');
    this.harvester.emit('finished', transaction);
  };

  trace.run(callback);
};

Trace.prototype.annotateState = function (name, value) {
  var active = tracer.active;
  active.set(name, value);
};
```