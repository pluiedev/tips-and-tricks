# 🦆 interfaces

## The TL;DR
**Duck interfaces** are things you can use to *implement new methods* on existing classes when used in conjunction with mixins.

For example:
```java
public class Target {
    // look at it! dull, boring, vacuous...
    // let's add some new behavior!
}

// some people call it `TargetExtension`, some `TargetHook`,
// it doesn't matter; all that matters is to stick to one convention
public interface TargetDuck {
    // prefix your custom methods with the mod ID;
    // that way a collision with other mods will become extremely unlikely
    // you can also use _s instead of $s, should you prefer
    void modid$excitingNewBehavior(int arg);
}

// how to implement it:
@Mixin(Target.class)
public abstract class TargetMixin
    implements TargetDuck // very important
{
    @Override
    public void modid$excitingNewBehavior(int arg) {
        if (arg > 4)
            System.out.println("greater than four! *gasps*");
        else
            System.out.println("lesser than four! *boos*");
    }
}

// how to use it:
public class ConsumerClass {
    public void someFunc(Target target) {
        // cast; this always* succeeds
        var duck = (TargetDuck) target;

        // * -ish. for `final` classes like `ItemStack` you need to do:
        // `var duck = (TargetDuck)(Object)target;`
        
        // More info in the long explanation.

        duck.modid$excitingNewBehavior(3); // >> lesser than four! *boos*
        duck.modid$excitingNewBehavior(69); // >> greater than four! *gasps*
    }
}
```

## The Long Explanation
Mixins are *really* cool. Not only can they *modify* behavior, they can also *define* new behavior — simply declare new methods on the mixin, and it should work, right?

Well technically, yes. But practically, no. Here's why:

When declaring new methods on mixins, they get *directly included* into the final class, like so:
```java
public class Target {
    // an all encompassing void
}

@Mixin(Target.class)
public abstract class TargetMixin {
    public void newMethod() {
        System.out.println("new method! yay!");
    }
}

// results in something like this when dumped:
public class Target {
    @MixinMerged(
        mixin = "TargetMixin",
        priority = 1000,
        sessionId = "deadbeef-6969-6969-0420-bad1ddeadbad"
    )
    public void modid$newMethod() {
        System.out.println("new method! yay!");
    }
}
```

However, this is all done *in runtime*. Without advanced (and often dangerous) APIs and mechanisms like reflection, it's impossible to call those new methods in regular code.

That is, until we add a **duck interface** that contains our custom methods.

### 🦆s go quack quack
This is how a **duck interface** looks like:

```java
public interface TargetDuck {
    // it's a recommended practice to prefix your custom methods
    // with your modid and a dollar sign, to avoid collisions with
    // other similarly named methods added by different mods.

    // underscores are also fine if you don't like dollar signs.
    void modid$newMethod();
}
```

In Java, you can cast (nearly) all objects to another interface, and the compiler and linter will happily accept that, despite your code might result in a `ClassCastException`. The reason is that there might be a subclass that *does* implement the interface that the program could only check *in runtime*.
```java
public class Target {}

public class TargetChild extends Target implements OtherInterface {}

public class SomewhereElse {
    public void someFunc(Target target) {
        // `target` may be a TargetChild, or some other class that
        // implements OtherInterface. The compiler simply doesn't know.
        // Hence, this is perfectly okay in compile time.
        var other = (OtherInterface) target;
    }
}
```

(This also explains why `final` classes cannot be casted directly into the duck, but we'll touch on them later)

**This can be used in our advantage.**
By casting a target object to our duck, we can get ahold of our sweet, sweet methods without the compiler complaining, even though the duck interface and the method implementation is clearly not in the declaration of the `Target` class.

```java
@Mixin(Target.class)
public abstract class TargetMixin implements TargetDuck {
    @Override
    public void modid$newMethod() {
        System.out.println("new method! yay!");
    }
}

// results in something like this when dumped:
public class Target implements TargetDuck {
    @MixinMerged(
        mixin = "TargetMixin",
        priority = 1000,
        sessionId = "deadbeef-6969-6969-0420-bad1ddeadbad"
    )
    public void modid$newMethod() {
        System.out.println("new method! yay!");
    }
}

// how to use in other code:
public class SomewhereElse {
    public void someFunc(Target target) {
        var other = (TargetDuck) target;
        // compiles and works just fine!
        other.modid$newMethod(); // >> "new method! yay!"
    }
}
```

### What about `final` classes?

`final` classes are a bit trickier. Since the compiler knows that there are no possible children of those classes, it deduces that there is no way such a cast could proceed, so it stops you with a nasty error message, something along the lines of "I'm smarter than you, and that clearly doesn't make sense."

```java
public final class Tricky {}

public interface TrickyDuck {
    void modid$newMethod();
}

@Mixin(Tricky.class)
public abstract class TrickyMixin implements TrickyDuck {
    @Override
    public void modid$newMethod() {
        System.out.println("new method! yay!");
    }
}

// when using it in other code, though...
public class SomewhereElse {
    public void someFunc(Tricky tricky) {
        // does not compile!
        var other = (TrickyDuck) tricky;
        other.modid$newMethod();
    }
}
```

...but Mixin adds the interface to the class anyway, so you can't really rely on the primitive™ wisdom of the compiler now. Instead, you must coax it into compiling with a nasty hack:

`var other = (TrickyDuck) (Object) tricky;`

This is fine with the compiler (for some reason), although advanced IDEs like IntelliJ IDEA might complain that there's simply no way for that to work, so they want to simplify that to a no-op or something. Don't listen to them. They are idiots. (At least in this context.)

Simply suppress the warning:
```java
// method 1; you can also use this to suppress entire classes or files
@SuppressWarnings("ConstantConditions") 
public void someFunc(Tricky tricky) {
    // method 2: 👇 use this to suppress one problematic line
    // this is *specific* for IDEs on the IntelliJ platform (i.e. IDEA, Android Studio, etc)

    //noinspection ConstantConditions
    var other = (TrickyDuck) (Object) tricky;
    other.modid$newMethod();
}
```

### We're still going to talk about how FAPI does it, innit?
...yeah right.
> The following names are in **Yarn/Quilt Mappings**, and the following information is accurate as of Fabric API version `0.37.0+1.17`.
> 
> Users of other mappings may need to manually translate said names to the corresponding name in their desired mappings.


One prominent example of a `final` class requiring such brute-force casting in Minecraft modding is `ItemStack`, which Fabric API needed to store some additional data and logic. (see `net.fabricmc.fabric.impl.tool.attribute.ItemStackContext`)
