# Overview

Note: the bulk of this document is translated almost verbatim from [the Moose
documentation](https://metacpan.org/dist/Moose/view/lib/Moose/Manual/MethodModifiers.pod).

---

Corinna provides a feature called "method modifiers". You can also think
of these as "hooks" or "advice".

It's probably easiest to understand this feature with a few examples:

```perl
role Some::Role {
     method foo :before () { print "about to call foo\n"; }
     method foo :after  () { print "just called foo\n"; }

     method foo :around () {
         print "  I'm around foo\n";

         $self->$ORIG(@_);

         print "  I'm still around foo\n";
     }
}

class Example :does(Some::Role) {
     method foo () {
         print "    foo\n";
     }
}
```

Now if we call `Example->new->foo` I'll get the following output:

```
about to call foo
  I'm around foo
    foo
  I'm still around foo
just called foo
```

You probably could have figured that out from the names `:before`,
`:after`, and `:around`.

Also, as you can see, the `:before` modifiers come before `:around` modifiers, and
`:after` modifiers come last.

When there are multiple modifiers of the same type, the `:before` and
`:around` modifiers run from the last added to the first, and `:after`
modifiers run from first added to last:

```
before 2
 before 1
  around 2
   around 1
    primary
   around 1
  around 2
 after 1
after 2
```

# Why use them?

Method modifiers have many uses. They are often used in roles to alter the
behavior of methods in the classes that consume the role. See [this RFC
page](https://github.com/Ovid/Cor/blob/master/rfc/roles.md) for more
information about roles.

Since modifiers are mostly useful in roles, some of the examples below
are a bit artificial. They're intended to give you an idea of how
modifiers work, but may not be the most natural usage.

# Before, after, and around modifiers

Method modifiers can be used to add behavior to methods without modifying the
definition of those methods.

## Before and after Modifiers

Method modifiers can be used to add behavior to a method that Corinna
generates for you, such as an field accessor:

```perl
field $size :reader :writer;

method set_size :before ($size) {
    Carp::cluck('Someone is setting size');
}
```

Another use for the `:before` modifier would be to do some sort of
prechecking on a method call. For example:

```perl
method set_size :before ($size) {
    die 'Cannot set size while the person is growing'
        if $self->is_growing;
}
```

This lets us implement logical checks that don't make sense as type
constraints. In particular, they're useful for defining logical rules
about an object's state changes.

Similarly, an `:after` modifier could be used for logging an action that
was taken.

Note that the return values of both `:before` and `:after` modifiers are
ignored.

## Around modifiers

An `:around` modifier is more powerful than either a `:before` or
`:after` modifier. It can modify the arguments being passed to the
original method, and you can even decide to simply not call the
original method at all. You can also modify the return value with an
`:around` modifier.

An `:around` modifier receives the original method injected into it via the
`$ORIG` variable.

```perl
method set_size :around ($size) {
    return $self->$ORIG()
        unless @_;

    $size = $size / 2
        if $self->likes_small_things();

    return $self->$ORIG($size);
}
```

**Important**: Note that while the `$ORIG` variable is injected directly into
the `around` method, this behavior and name is provisional and may be changed.

## Execution order of method modifiers and inheritance

When both a superclass and an inheriting class have the same method modifiers,
the method modifiers of the inheriting class are wrapped around the method
modifiers of the superclass, as the following example illustrates:

Here is the parent class:

```perl
class Superclass {
     method rant () { printf "        RANTING!\n" }
     method rant :before () { printf "    In %s before\n", __PACKAGE__ }
     method rant :after  () { printf "    In %s after\n",  __PACKAGE__ }
     method rant :around () {
         printf "      In %s around before calling original\n", __PACKAGE__;
         $self->$ORIG;
         printf "      In %s around after calling original\n", __PACKAGE__;
     }
}
```

And the child class:

```perl
class Subclass :isa(Superclass) {
     method rant :before () { printf "In %s before\n", __PACKAGE__ }
     method rant :after  () { printf "In %s after\n",  __PACKAGE__ }
     method rant :around () {
         printf "  In %s around before calling original\n", __PACKAGE__;
         $self->$ORIG;
         printf "  In %s around after calling original\n", __PACKAGE__;
     }
}
```

And here's the output when we call the wrapped method (`Child->rant`):

```
$ perl -MSubclass -e 'Subclass->new->rant'

In Subclass before
  In Subclass around before calling original
    In Superclass before
      In Superclass around before calling original
        RANTING!
      In Superclass around after calling original
    In Superclass after
  In Subclass around after calling original
In Subclass after
```

# Exceptions and stack traces

An exception thrown in a `:before` modifier will prevent the method it
modifies from being called at all. An exception in an `:around` modifier may
prevent the modified method from being called if it's thrown before
`$self->$ORIG` is called, but not after. An exception in an `:after` modifier
obviously cannot prevent the method it wraps from being called.

From the caller's perspective, an exception in a method modifier will look
like the method it called threw an exception. However, method modifiers are
just standard Perl subroutines. This means that they end up on the stack in
stack traces as an additional frame.

# Caveats

Be extremely careful if you use method modifiers to alter the output. If
multiple modifiers are used and one adds $10 to a total and another one adds
20% VAT (tax), the final result will depend on the order the modifiers have been
applied. Because this order is not guaranteed, you cannot be sure of what the
result will be. Instead, write a `final_total` method (or something similar)
which guarantees the order of application:

```perl
method final_total () {
    my $total = ... calculate total
    $total = $self->some_surcharge($total);
    $total = $self->add_vat($total);
    return $total
}
```

A method modifier implicitly adds the method to the list of required methods.

Modifiers do _not_ get applied to methods until class/role composition is
finished. Otherwise, the modifiers could be applied to the wrong method.
