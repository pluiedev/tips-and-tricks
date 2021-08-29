# Mixins Should Always Be `abstract`
> This article is **up to date** as of Mixin version `0.9.4` (last updated August 29th, 2021).

## The TL;DR
**Mixins should always be `abstract`;** they should be `abstract class`es for most mixins or `interface`s for invoker/accessor mixins.

```java
// bad
@Mixin(Target.class)
public class TargetMixin {
    //...
}

// good
@Mixin(Target.class)
public abstract class TargetMixin {
    //...
}
```

## The Long Explanation
So why *should* we make our mixins `abstract`? Well, there are a couple of reasons.

### Uninstantiable from code
You really aren't supposed to create objects of mixin classes; since mixins are "applied" on top of the original class, and is never accessible in its original form during runtime, any attempt at creating one simply results in an error.

If we have this set up, for example:
```java
public class Target {}

@Mixin(Target.class)
public class TargetMixin {
    // bare methods just get injected into the original class, BTW
    public void thing() {
        System.out.println("Hello world!");
    }
}
```

When the mixin gets processed during startup, the mixin processor would transform the `Target` class into something like this...

```java
public class Target {
    @MixinMerged(
        mixin = "TargetMixin",
        priority = 1000,
        sessionId = "deadbeef-6969-6969-0420-bad1ddeadbad"
    )
    public void thing() {
        System.out.println("Hello world!");
    }
}

// the `TargetMixin` class has gone *poof*!
```

...and the mixin class itself is nowhere to be found.
So if we have some code that does this:

```java
public class SomewhereElse {
    public void someFunc() {
        var mixinInstance = new TargetMixin(); // this is silly
        mixinInstance.thing();
    }
}
```

Mixin will happily tell you that no, you can't do that.
```
Caused by: org.spongepowered.asm.mixin.transformer.throwables.IllegalClassLoadError: Illegal classload request for TargetMixin. Mixin is defined in modid.mixins.json and cannot be referenced directly
	at org.spongepowered.asm.mixin.transformer.MixinProcessor.applyMixins(MixinProcessor.java:330)
	at org.spongepowered.asm.mixin.transformer.MixinTransformer.transformClass(MixinTransformer.java:208)
	at org.spongepowered.asm.mixin.transformer.MixinTransformer.transformClassBytes(MixinTransformer.java:178)
	at org.spongepowered.asm.mixin.transformer.FabricMixinTransformerProxy.transformClassBytes(FabricMixinTransformerProxy.java:23)
	at net.fabricmc.loader.launch.knot.KnotClassDelegate.getPostMixinClassByteArray(KnotClassDelegate.java:162)
	at net.fabricmc.loader.launch.knot.KnotClassLoader.loadClass(KnotClassLoader.java:154)
	at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:519)
```

Having all mixins `abstract` helps to alleviate this, since `abstract` classes can never be instantiated. Safety FTW!

### Boilerplate reduced
Now to demonstrate it, I'll cite a somewhat common target to mix into in Minecraft, so that naturally means...

> All following code snippets use **Yarn/Quilt Mappings** (version `1.17.1+build.25`), and are up to date as of Minecraft version **`1.17.1`**.

So say you want to mix into `LivingEntity` to override some damage calculation in `damage`. More specifically, you want to override checking for invulnerability — if the entity is of a certain type and the damage is of a certain kind, then it's not going to be invulnerable no matter what.

So you quickly drafted something out:
```java
@Mixin(LivingEntity.class)
public class LivingEntityMixin {
    @Redirect(
        method = "damage",
        at = @At(
            value = "INVOKE",
            target = "Lnet/minecraft/entity/Entity;isInvulnerableTo(Lnet/minecraft/entity/damage/DamageSource;)Z",
            ordinal = 0
        )
    )
    private boolean isInvulnerableTo(DamageSource dmgSrc) {
        if (this.type == MyEntities.MY_TYPE && dmgSrc instanceof ProjectileDamageSource)
            return false;
        else
            return this.isInvulnerableTo(dmgSrc);
    }
}
```

But alas! Both `this.type` and `this.isInvulnerableTo` are defined in `Entity`, not `LivingEntity`, so you can't `@Shadow` them! So what should you do?

The officially recommended way is to simply inherit from `Entity`; since the original class already inherits from `Entity`, there's no possible danger there and you can get its fields and methods freely. So here it goes:

```java
@Mixin(LivingEntity.class)
public class LivingEntityMixin extends Entity {
    @Redirect(
    // snip
```

But when you try to compile it, you'll get this nasty error:

```
error: LivingEntityMixin is not abstract and does not override abstract method createSpawnPacket() in Entity
public class LivingEntityMixin extends Entity {
       ^
```

So then you realized you have to override the abstract methods of `Entity` to make it work. Logical, since we *are* inheriting from it and we have a concrete class.

```java
@Mixin(LivingEntity.class)
public class LivingEntityMixin extends Entity {
    @Override
    protected void initDataTracker() {

    }

    @Override
    protected void readCustomDataFromNbt(NbtCompound nbtCompound) {

    }

    @Override
    protected void writeCustomDataToNbt(NbtCompound nbtCompound) {

    }

    @Override
    public Packet<?> createSpawnPacket() {
        return null;
    }

    @Redirect(
        method = "damage",
    // snip
}
```

Ok, so *now* it should compile, right?
```
error: constructor Entity in class Entity cannot be applied to given types;
public class LivingEntityMixin extends Entity {
       ^
  required: EntityType<?>,World
  found:    no arguments
  reason: actual and formal argument lists differ in length
```

Ahh... we need to explicitly declare a constructor. Damn.
So the entire thing looks like this.

```java
@Mixin(LivingEntity.class)
public class LivingEntityMixin extends Entity {
    public LivingEntityMixin(EntityType<?> entityType, World world) {
        super(entityType, world);
    }

    @Override
    protected void initDataTracker() {

    }

    @Override
    protected void readCustomDataFromNbt(NbtCompound nbtCompound) {

    }

    @Override
    protected void writeCustomDataToNbt(NbtCompound nbtCompound) {

    }

    @Override
    public Packet<?> createSpawnPacket() {
        return null;
    }

    @Redirect(
        method = "damage",
        at = @At(
            value = "INVOKE",
            target = "Lnet/minecraft/entity/Entity;isInvulnerableTo(Lnet/minecraft/entity/damage/DamageSource;)Z",
            ordinal = 0
        )
    )
    private boolean isInvulnerableTo(DamageSource dmgSrc) {
        if (this.type == MyEntities.MY_TYPE && dmgSrc instanceof ProjectileDamageSource)
            return false;
        else
            return this.isInvulnerableTo(dmgSrc);
    }
}
```

***Yikes.*** What if there's a way that doesn't involve any of this boilerplate?

Oh yeah, `abstract` classes needn't to override abstract methods from the superclass.

```java
@Mixin(LivingEntity.class)
public abstract class LivingEntityMixin extends Entity {
    // we still need this because Java still demands us to fulfill whatever
    // the superclass's constructor wants
    // however, this doesn't get called or merged by Mixin so it's just there
    // to please the compiler. again, it's not that smart for our use case.
    public LivingEntityMixin(EntityType<?> entityType, World world) {
        super(entityType, world);
    }

    @Redirect(
        method = "damage",
        at = @At(
            value = "INVOKE",
            target = "Lnet/minecraft/entity/Entity;isInvulnerableTo(Lnet/minecraft/entity/damage/DamageSource;)Z",
            ordinal = 0
        )
    )
    private boolean isInvulnerableTo(DamageSource dmgSrc) {
        if (this.type == MyEntities.MY_TYPE && dmgSrc instanceof ProjectileDamageSource)
            return false;
        else
            return this.isInvulnerableTo(dmgSrc);
    }
}
```
80% of the boilerplate gone in an instant. Sweet!

This also applies to `@Shadow` methods in your code. If you can recall, `@Shadow`ing a method looks like this:
```java
public class Target {
    public int randomNumber() {
        return 4;
    }
}

@Mixin(Target.class)
public class TargetMixin {
    // basically the same function definition,
    // albeit with a `private` visibility and `@Shadow`
    @Shadow private int randomNumber() {
        // not important; mixin ignores it anyway
        return 69420;
    }
}
```

But this gets extremely tedious after a while — you have to match the return type for every single function, even if the method just consists a return and a default value.

Some solve this with an exception that gets thrown whenever the content of the `@Shadow` method is actually executed (i.e. never):
```java
@Shadow private int randomNumber() {
    // if something does manage to execute this,
    // it's irrecoverable anyway
    throw new AssertionError();
}
```

But that's just one more boilerplate than ideal.

With `abstract` mixins, we can just mark the method as `abstract`:
```java
@Mixin(Target.class)
public abstract class TargetMixin {
    // look mom, no error!
    @Shadow private abstract int randomNumber();
}
```

And boom. Done. Since `@Shadow` methods are nothing but a stand-in for the real method, this is perfectly safe, and no more worrying about implementation details.

<details>
<summary>
Footnote: The unfortunate truth about <code>static</code> <code>@Shadow</code> methods
</summary>

With `static` `@Shadow` methods things get a bit more complicated.

Since there is no such thing as an `abstract static` method, you *have* to use a boilerplate for those. Sorry!
```java
@Shadow private static int randomNumber() {
    throw new AssertionError();
}
```
</details>