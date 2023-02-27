---
layout: sip
permalink: /sips/:title.html
stage: pre-sip
status: submitted
title: SIP-52 - Binary APIs
---

**By: Author Nicolas Stucki**

## History

| Date          | Version            |
|---------------|--------------------|
| Feb 27 2022   | Initial Draft      |

## Summary

This proposal introduces the `@binaryAPI` and  `@binaryAPIAccessor` annotations on term definitions. The purpose of binary APIs is to have publicly accessible definitions in generated bytecode for definitions that are private or protected.


## Motivation

### Provide a sound way to refer to private members in inline definitions

Currently, the compiler automatically generates accessors for references to private members in inline definitions. This scheme interacts poorly with binary compatibility. It causes the following three unsoundness in the system:
* Changing any definition from private to public is a binary incompatible change
* Changing the implementation of an inline definition can be a binary incompatible change
* Removing final from a class is a binary incompatible change

You can find more details in https://github.com/lampepfl/dotty/issues/16983

### Avoid duplication of inline accessors

Ideally, private definitions should have a maximum of one inline accessor, which is not the case now.
When an inline method accesses a private/protected definition that is defined outside of its class, we generate an inline in the class of the inline method. This implies that accessors might be duplicated if a private/protected definition is accessed from different classes.

### Removing deprecated APIs

There is no precise mechanism to remove a deprecated method from a library without causing binary incompatibilities. We should have a straightforward way to indicate that a method is no longer publicly available but still available in the generated code for binary compatibility.

```diff
- @deprecated(...) def myOldAPI: T = ...
+ private[C] def myOldAPI: T = ...
```


## Proposed solution

### High-level overview

This proposal introduces 2 the `@binaryAPI` and `@binaryAPIAccessor` annotations, and changes adds a migration path to inline methods.

#### `@binaryAPI` annotation

A binary API is a definition that is annotated with `@binaryAPI` or overrides a definition annotated with `@binaryAPI`.
This annotation can be placed on `def`, `val`, `lazy val`, `var`, `object`, and `given` definitions.
A binary API will be publicly available in the bytecode.

This annotation cannot be used on `private`/`private[this]` definitions.

Removing this annotation from a non-public definition is a binary incompatible change.

Example:

~~~ scala
class C {
  @binaryAPI private[C] def packagePrivateAPI: Int = ...
  @binaryAPI protected def protectedAPI: Int = ...
  @binaryAPI def publicAPI: Int = ... // warn: `@binaryAPI` has no effect on public definitions
}
~~~
will generate the following bytecode signatures
~~~ java
public class C {
  public C();
  public int packagePrivateAPI();
  public int protectedAPI();
  public int publicAPI();
}
~~~

#### `@binaryAPIAccessor` annotation

A binary API with accessor is a definition that is annotated with `@binaryAPIAccessor`.
This annotation can be placed on `def`, `val`, `lazy val`, `var`, `object`, and `given` definitions.
The annotated definition will get a public accessor.

This can be used to access `private`/`private[this]` definitions within inline definitions.

Example:
~~~ scala
class C {
  @binaryAPIAccessor private def privateAPI: Int = ...
  @binaryAPIAccessor def publicAPI: Int = ...
}
~~~
will generate the following bytecode signatures
~~~ java
public class C {
  public C();
  private int privateAPI();
  public int publicAPI();
  public final int C$$inline$privateAPI();
  public final int C$$inline$publicAPI();
}
~~~

Note that the change from `private[this]` to package private, protected or public is a binary compatible change.
Removing this annotation is a binary incompatible change.

#### Binary API and inlining

If there is a reference to a binary API in an inline method we can use the definition without needing an inline accessor.

Example 3:
~~~ scala
class C {
  @binaryAPI protected def a: Int = ...
  protected def b: Int = ...
  inline def foo: Int = a + b
}
~~~
before inlining the compiler will generate the accessors for inlined definitions
~~~ scala
class C {
  @binaryAPI protected def a: Int = ...
  protected def b: Int = ...
  final def C$inline$b: Int = ...
  inline def foo: Int = a + C$inline$b
}
~~~

Note that if the inlined member is `a` would be private, we would generate the accessor `C$inline$a`, which happens to be binary compatible with the automatically generated one.
This is only a tiny mitigation of binary compatibility issues compared with all the different ways accessors can be generated.

### Specification

We must add `binaryAPI` and `binaryAPIAccessor` to the standard library.

```scala
package scala.annotation

final class binaryAPI extends scala.annotation.StaticAnnotation
final class binaryAPIAccessor extends scala.annotation.StaticAnnotation
```

#### `@binaryAPI` annotation

* Only valid on `def`, `val`, `lazy val`, `var`, `object`, and `given`.
* TASTy will contain references to non-public definitions that are out of scope but `@binaryAPI`. TASTy already allows those references.
* Annotated definition will be public in the generated bytecode. Definitions should be made public as early as possible in the compiler phases, as this can remove the need to create other accessors. It should be done after we check the accessibility of references.


#### `@binaryAPIAccessor` annotation

* Only valid on `def`, `val`, `lazy val`, `var`, `object`, and `given`.
* An public accessor will be generated for the annotated definition. This accessor will be named `<fullClassName>$$inline$<definitionName>`.

#### Inline

* Inlining will not require the generation of an inline accessor for binary APIs.
* Inlining will not require the generation of a new inline accessor, it will use the binary API accessors.
* The user will be warned if a new inline accessor is automatically generated.
  The message will suggest `@binaryAPI` or `@binaryAPIAccessor` and how to fix potential incompatibilities.
  In a future version, these will become an error.

### Compatibility

The introduction of the `@binaryAPI` and `@binaryAPIAccessor` do not introduce any binary incompatibility.

Using references to `@binaryAPI` and `@binaryAPIAccessor` in inline code can cause binary incompatibilities. These incompatibilities are equivalent to the ones that can occur due to the unsoundness we want to fix. When migrating to binary APIs, the compiler will show the implementation of accessors that the users need to add to keep binary compatibility with pre-binaryAPI code.

A definition can be both `@binaryAPI` and `@binaryAPIAccessor`. This would be used to indicate that the definition used to be private, but now we want to publish it as public. The definition would become public, and the accessor would be generated for binary compatibility.

### Other concerns

* Tools that analyze inlined TASTy code might need to know about `@binaryAPI`. For example TASTy MiMa.

### Open questions

#### Question 1
Should `@binaryAPIAccessor` accessors be named `<fullClassName>$$<definitionName>`? This encoding would match the names of `trait` accessor generated for private definition. We could use a single accessor instead of two. This would introduce an extra binary incompatibility with pre-binaryAPI code.

#### Question 2
```scala
class A:
  @binaryAPIAccessor protected def protectedDef: Int = ...
class B extends A:
  override protected def protectedDef: Int = ...
  inline def inlinedDef: Int =
    // Should this use the accessor of generated for `A.protectedDef`? Or should we warn that `protectedDef` should be a `@binaryAPI`
    protectedDef
```

## Alternatives

Having alternatives is not a strict requirement for a proposal, but having at least one with carefully exposed pros and cons gives much more weight to the proposal as a whole. -->

### Only add `@binaryAPI`
This would simplify the system and the user interaction with this feature. The drawback is that we could not access `private[this]` definitions in inline code. Users would need to use `private[C]` instead, which could cause name clashes.

### Only add `@binaryAPIAccessor`
This would simplify the system and the user interaction with this feature. The drawback is that we would add code size and runtime overhead to all uses of this feature. It would not solve the [Removing deprecated APIs](#removing-deprecated-apis) motivation.

## Related work

* Proof of concept: https://github.com/lampepfl/dotty/pull/16992
* Initial discussions: https://github.com/lampepfl/dotty/issues/16983

<!-- ## FAQ -->