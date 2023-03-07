# ECMAScript Decorators for Class Method and Constructor Parameters

This proposal adds support for decorators on the parameters of class constructors and class methods.

## Status

**Stage:** 0  \
**Champion:** Ron Buckton (@rbuckton)  \
**Last Presented:** (none)

_For more information see the [TC39 proposal process](https://tc39.es/process-document/)._

## Authors

- Ron Buckton (@rbuckton)

# Overview and Motivations

Decorators are a metaprogramming capability for ECMAScript which allow you to annotate a declaration with a function
reference that will be invoked with arguments pertaining to the defining characteristics of that declaration, as well as
to potentially replace or augment the declaration. The current Stage 3 Decorators proposal allows for the use of
Decorators on `class` declarations and expressions, as well as their methods, getters, setters, and fields.

Parameter decorators extend this metaprogramming capability to target the parameters of class constructors and class
methods, and are intended to support a number of use cases including:

- Constructor parameter-based *Dependency Injection* (DI).
- Object-relational mapping (ORM) entity construction that couples with private fields.
- Method parameter marshaling for Foreign Function Interfaces (FFI).
- Metadata for Parameters (when paired with https://github.com/tc39/proposal-decorator-metadata).
- Routing: bind HTTP request headers/querystring parameters/post body fields/etc. to parameters.
- Argument validation: disallow `null`/`undefined`, range validation, regexp string validation, etc.

Many of these scenarios are drawn from existing uses of legacy decorators in TypeScript, as well as similar capabilities
in languages like Java and C#. For example, VS Code makes heavy use of constructor parameter decorators for dependency
injection.

Parameter decorators allow you to easily annotate a method or constructor parameter so as to ascribe specific metadata
or to alter behavior:

```js
class UserManager {
  createUser(@NotEmpty username, @NotEmpty password, @ValidateEmail emailAddress, @Minimum(0) age) { ... }
}
```

While its feasible to do the same with regular method decorators, you are only really able to associate such a
decoration via a parameter's ordinal position. It becomes much harder to maintain them over time as parameters are
added, removed, or reordered, as well as far more difficult to read and review when you must relate a decorator and 
a parameter visually based solely on ordinal position:

```diff js
  class UserManager {
    @param(3, ValidateEmail)
    @param(2, Minimum(0))
    @param(1, NotEmpty)
    @param(0, NotEmpty)
--  createUser(username, password, age, emailAddress) { ... }
++  createUser(username, password, emailAddress, age) { ... }
++  // oops, forgot to fix @param decorator order...
  }
```

Method decorators also don't provide useful context for a parameter, such as its name, that could otherwise be leveraged
by something like a `@FromForm` parameter in an HTTP router:

```js
// without parameter decorators:
class BookApi {
  @Route("/book/:isbn/review", { method: "post", form: true })
  @param(0, FromUri({ name: "isbn" })) // need to repeat declaration name here and in parameter list...
  @param(1, FromForm({ name: "subject" }))
  @param(2, FromForm({ name: "description" }))
  @param(3, FromForm({ name: "score" }))
  postReview(isbn, subject, description, score) { ... }
}

// with parameter decorators:
class BookApi {
  // no need for repetition if the names match...
  @Route("/book/:isbn/review", { method: "post", form: true })
  postReview(@FromUri isbn, @FromForm subject, @FromForm description, @FromForm score) { ... }
}
```

Parameter decorators might also provide a useful avenue for data validation and transformation by providing a mechanism
to observe and potentially replace an incoming argument, similar to how a field decorator allows you to observe and 
potentially replace an initializer:

```js
// argument validation (observe argument without mutating it):

function NotEmpty(target, context) {
  if (context.kind !== "parameter") throw new TypeError();
  return function (arg) {
    if (typeof arg !== "string" || arg.length !== 0) {
      throw new TypeEror(`Argument '${context.name}' expects a non-empty string`);
    }
    return arg
  }
}

// FFI marshaling (marshal input argument from foreign type to native type):

/** Convert FFI pointer for length-prefixed BSTR to a `string` */
function BStr(target, context) {
  return function (arg) {
    if (arg instanceof ffi.Pointer) {
      return arg.asBStr();
    }
    return arg;
  }
}

/** Convert FFI pointer for a 2-byte null-terminated unicode character string to a `string` */
function LPWStr(target, context) {
  return function (arg) {
    if (arg instanceof ffi.Pointer) {
      return arg.asLPWStr();
    }
    return arg;
  }
}
```

# Prior Art

- TypeScript: Legacy (experimental) Decorators
- C#: Parameter attributes
- Java: Parameter annotations

# Syntax

Parameter decorators use the same syntax as class and method decorators, except they may be placed preceding a parameter
in a class `constructor` or class method declaration:

```js
// constructor parameter decorators
class CustomizationService {
  constructor(
    @inject("StorageService") storageService,
    @inject("UserProfileService") userProfileService
  ) {
    ...
  }
}

// method parameter decorators:
class BookApi {
  @Route("/book/:isbn", { method: "get" })
  getBook(@FromUri isbn) { ... }

  @Route("/book/:isbn/review", { method: "post", form: true })
  postReview(@FromUri isbn, @FromForm subject, @FromForm description, @FromForm score) { ... }
}

// setter parameters:
class User {
  ...
  get username() { return this.#username; }
  set username(@NotEmpty value) { this.#username = value; }
}
```

Parameter decorators on object literal methods and setters, or on function declarations and expressions, are out of
scope for this proposal. However, we intend for the design of this proposal to allow for future expansion into that
space once function decorators have been adopted within the language.

# Grammar

The following is a rough outline of the proposed grammar and is not intended to be interpreted as the final syntax.
Should this proposal be adopted, it is possible the syntax could change based on feedback.

```diff grammarkdown
  FunctionRestParameter[Yield, Await] :
--    BindingRestElement[?Yield, ?Await]
++    DecoratorList[?Yield, ?Await]? BindingRestElement[?Yield, ?Await]

  FormalParameter[Yield, Await] :
--    BindingElement[?Yield, ?Await]
++    DecoratorList[?Yield, ?Await]? BindingElement[?Yield, ?Await]
```

# Semantics

## Early Errors

Along with the above proposed grammar, it would be an early error if _DecoratorList_ were in the _FormalParameters_ of a
_FunctionDeclaration_, _FunctionExpression_, _GeneratorDeclaration_, _GeneratorExpression_, _AsyncFunctionDeclaration_,
_AsyncFunctionExpression_, _AsyncGeneratorDeclaration_, _AsyncGeneratorExpression_, _ArrowFunction_, or
_AsyncArrowFunction_. It would also be an early error if _DecoratorList_ were in the _FormalParameters_ of a
_MethodDefinition_ if that _MethodDefinition_ is immediately contained within an _ObjectLiteralExpression_.

## Decorator Expression Evaluation

Parameter decorators would be evaluated in document order, as with any other decorators. Parameter decorators _do not_
have access to the local scope within the method body, as they are evaluated statically, at the same time as any
decorators that might be on their containing method or class.

For example, given the source

```js
@A
@B
class C {
  @C
  @D
  method(@E @F param1, @G @H param2) { }
}
```

the decorator expressions would be _evaluated_ in the following order: `A`, `B`, `C`, `D`, `E`, `F`, `G`, `H`

## Decorator Application Order

Parameter decorators will be applied prior to application of any decorators on their containing method. The parameter
decorators for a given parameter are applied independently of those on a subsequent parameter. Within the decorators
of a single parameter, those decorators are applied in reverse order, in keeping with decorator evaluation elsewhere
within the language.

For example, given the source

```js
@A
@B
class C {
  @C
  @D
  method(@E @F param1, @G @H param2) { }
}
```

decorators would be _applied_ in the following order:

- `F`, `E` of `param1`
- `H`, `G` of `param2`
- `D`, `C` of `method`
- `B`, `A` of `class C`

## Anatomy of a Parameter Decorator

A parameter decorator is generally expected to have two parameters: `target` and `context`. Much like a field decorator,
however, the `target` argument will always be `undefined` as a parameter is not itself a reified object in JavaScript.

The `context` for a parameter decorator would contain useful information about the parameter:

```ts
type ParameterDecoratorContext = {
  kind: "parameter";
  name: string | undefined;
  index: number;
  rest: boolean;
  function: {
    kind: "class" | "method" | "setter";
    name: string | symbol | undefined;
  };
  metadata: object;
  addInitializer(initializer: () => void): void;
}
```

- `kind` &mdash; Indicates the kind of element being decorated.
- `name` &mdash; A string if the parameter is named, or `undefined` if the parameter is a binding pattern.
- `index` &mdash; The ordinal position of the parameter in the parameter list, which is necessary for constructor
  parameter injection for scenarios like dependency injection.
- `rest` &mdash; Indicates whether the parameter is a `...` rest element.
- `function` &mdash; Contains limited information about the function to which the parameter belongs, which is necessary
  to distinguish between members when assigning to the `metadata` property, and in the future to distinguish between
  class method parameters and function parameters, as that may also impact how metadata is assigned.
- `metadata` &mdash; In keeping with the Decorator Metadata proposal, you would be able to attach metadata to a class.
- `addInitializer` &mdash; This would allow you to attach an extra static or instance initalizer, much like you could
  for a decorator on the containing method.

A parameter decorator may either return `undefined`, or a function value. If a function is returned, it will be later
invoked when that parameter is bound during its containing function's invocation. This behaves much like a function
returned from a field decorator:

```js
class C {
  method(@A param1) {
    ...
  }
}
```

is roughly equivalent to

```js
var _param1_init
class C {
  static {
    _param1_init = A(undefined, { kind: "parameter", name: "param1", index: 0, /*...*/ });
  }
  method(param1) {
    if (_param1_init !== undefined) param1 = _param1_init.call(this, param1);
    ...
  }
}
```

# Examples

TBD

# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.
* [ ] [Prose][Prose] outlining the problem or need and the general shape of a solution.
* [ ] Illustrative [examples][Examples] of usage.
* [ ] High-level [API][API].

### Stage 2 Entrance Criteria

* [ ] [Initial specification text][Specification].
* [ ] [Transpiler support][Transpiler] (_Optional_).

### Stage 3 Entrance Criteria

* [ ] [Complete specification text][Specification].
* [ ] Designated reviewers have signed off on the current spec text:
  * [ ] [Reviewer #1][Stage3Reviewer1] has [signed off][Stage3Reviewer1SignOff]
  * [ ] [Reviewer #2][Stage3Reviewer2] has [signed off][Stage3Reviewer2SignOff]
* [ ] The [ECMAScript editor][Stage3Editor] has [signed off][Stage3EditorSignOff] on the current spec text.

### Stage 4 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have  been written for mainline usage scenarios and [merged][Test262PullRequest].
* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].


<!-- # References -->

<!-- Links to other specifications, etc. -->

<!-- * [Title](url) -->

<!-- # Prior Discussion -->

<!-- Links to prior discussion topics on https://esdiscuss.org -->

<!-- * [Subject](https://esdiscuss.org) -->

<!-- The following are shared links used throughout the README: -->

[Champion]: #status
[Prose]: #motivations
[Examples]: #examples
[API]: #api
[Specification]: #todo
[Transpiler]: #todo
[Stage3Reviewer1]: #todo
[Stage3Reviewer1SignOff]: #todo
[Stage3Reviewer2]: #todo
[Stage3Reviewer2SignOff]: #todo
[Stage3Editor]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo
