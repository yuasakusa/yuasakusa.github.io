---
layout: post
title:  "Initializing the same variable twice in Dart"
date:   2014-01-18 00:10:00
categories: dart
---

I recently started learning
[programming language Dart](https://www.dartlang.org/),
and realized that according to the wording
of the [language specification][dart-spec],
it is possible to initialize some variables twice in Dart.
And I am not confusing initialization with assignment.

I am referring to the language spec 1.1,
although I do not think that the relevant part has changed since language spec 1.0.

This happens because an instance variable of a class
can have an initializer in its declaration,
and a constructor of that class can contain an initializer
for the same instance variable in its initializer list (after a colon).

Consider the following Dart code.

{% highlight dart %}
// Program 1

class C {
  int x = 10;    // first initializer
  C() : x = 20;  // second initializer
}

main() {
  print((new C()).x); // => 20
}
{% endhighlight %}

In the code, there are two initializers
which tell how instance variable `x` is initialized:

1. Instance variable declaration `int x = 10` contains initializer `= 10`.
2. Constructor `C()` contains initializer `x = 20` in its initializer list.
(Note that `x = 20` here is not an assignment expression.)

This code prints 20, which is the result of the second initializer.
But this is not because the existence of the second initializer
_prevents_ the initialization specified by the first initializer.
Rather, according to the language spec,
`x` is initialized to 10 first as stated in
the [section about the `new` expression][dart-spec-new],
and then the second initializer initializes `x` to 20 as stated in the
[section about the initializer list of a constructor][dart-spec-init-list].
Therefore, the output of the program is a result
of the second initializer having overwritten the result of the first initializer.

I cannot come up with a way to observe the fact that `x` is actually initialized twice,
and there may well be no such way.
However, it is easy to observe that the expression in the first initializer is evaluated:

{% highlight dart %}
// Program 2

class C {
  int x = 100 ~/ 0; // when evaluated, this will throw
                    // IntegerDivisionByZeroException
  C() : x = 20;
}

main() {
  print((new C()).x); // Unhandled exception: IntegerDivisionByZeroException
}
{% endhighlight %}

This does not look like an oversight in the spec,
because the spec explicitly states that it is a runtime error
if a _final_ instance variable is initialized by both the variable declaration
and an initializer in a constructor
(see also [issue 12539 of Dart][dart-issue-12539]).

Although it does not seem to cause any real harm,
I find it unintuitive that some variables are initialized twice.
I have no idea why the spec defines the behavior this way,
particularly given that the effect of the first initialization does not seem visible
except that evaluation of the expression in the first initializer may have a side effect
and that the first initialization causes the second initialization to be a runtime error
in the case of final variables.

I am not sure if it is worth,
but there are at least two ways to modify the language spec
so that no variables are initialized more than once:

* Forbid a constructor having an initializer for an instance variable
whose declaration also contains an initializer.
In my opinion, if this is implemented,
this should be implemented as a compile-time check rather than runtime check.
In this case, both Programs 1 and 2 above become compile error.
It should be a compile error also for final instance variables.
* Treat the initializer in the instance variable declaration as a “default initializer”.
If a constructor contains an instance variable initializer,
the initializer in the declaration of that instance variable should be ignored,
and its right-hand side should not be evaluated.
In this case, both Programs 1 and 2 will print 20,
and in particular, Program 2 will no longer result in an uncaught exception.
It will be also fine if a final instance variable has initializers
in both the variable declaration and the constructor,
in which case only the initializer in the constructor will be used.

[dart-spec]: https://www.dartlang.org/docs/spec/
[dart-spec-new]: https://www.dartlang.org/docs/spec/latest/dart-language-specification.html#h.twiod7rqtbah
[dart-spec-init-list]: https://www.dartlang.org/docs/spec/latest/dart-language-specification.html#h.7ybyo5btajop
[dart-issue-12539]: https://code.google.com/p/dart/issues/detail?id=12539
