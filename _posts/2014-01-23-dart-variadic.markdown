---
layout: post
title:  "Variadic functions in Dart"
date:   2014-01-23 21:00:00
categories: dart
---

There is a [feature request to support functions
which can take a variable number of arguments in Dart][dart-issue-16253]
(the same request was mentioned earlier in
[a comment to another feature request][dart-issue-10811-c11]).
But we can implement such functions
without modifying the language or the compiler!

{% highlight dart %}
typedef dynamic ApplyType(List positionalArguments);

class Variadic implements Function {
  final ApplyType _apply;

  Variadic(this._apply) {}

  @override
  dynamic noSuchMethod(Invocation invocation) {
    if (invocation.memberName == #call) {
      if (invocation.isMethod)
        return _apply(invocation.positionalArguments);
      if (invocation.isGetter)
        return this;
    }
    return super.noSuchMethod(invocation);
  }
}

num sumList(List<num> xs) =>
  xs.fold(0, (num acc, num x) => acc + x);

final Function sumVariadic = new Variadic(sumList);

void main() {
  print(sumList([100, 200, 300]));
  print(sumVariadic(100, 200, 300));
}
{% endhighlight %}

Notes:

* The trick is that according to the language specification,
  `sumVariadic(…)` is a shorthand for `sumVariadic.call(…)`
  ([Section 12.15.4, “Function Expression Invocation”][function-expression-invocation]).
  Although we cannot define `call` as a method of `sumVariadic`
  (because then `call` itself must be a variadic function!),
  we can define the [`noSuchMethod`][noSuchMethod] method
  to specify what this method invocation should actually do.

* The declaration of `Variadic` produces a static warning
  “Concrete classes that implement `Function` must implement the method `call()`”.
  This is as described in the language specification
  ([Section 15.5, “Function Types”][function-types]),
  but I am not sure if this warning is desirable.

* With this approach, the static type of `sumVariadic(100, 200, 300)`
  is `dynamic`, not `num`.

* Needless to say, if you want to use variable-length argument lists
  seriously, you probably want the proper support of the language
  instead of this hack.
  But it is fun to try.

* The code above supports only positional arguments.
  Support for named arguments is left as an exercise.

[dart-issue-16253]: http://code.google.com/p/dart/issues/detail?id=16253
[dart-issue-10811-c11]: http://code.google.com/p/dart/issues/detail?id=10811#c11
[function-expression-invocation]: https://www.dartlang.org/docs/spec/latest/dart-language-specification.html#h.5l8tud6ne77w
[noSuchMethod]: https://api.dartlang.org/docs/channels/stable/latest/dart_core/Object.html#noSuchMethod
[function-types]: https://www.dartlang.org/docs/spec/latest/dart-language-specification.html#h.hj977zpcf6uf
