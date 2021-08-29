# Expanding Arrays...
> All following code snippets use **Yarn/Quilt Mappings** (version `1.17.1+build.25`), and are up to date as of Minecraft version **`1.17.1`**.

> This article is **up to date** as of Mixin version `0.9.4` (last updated August 29th, 2021).

[...and other immutable collections](#what-about-other-similarly-immutable-collections).

## The TL;DR
```java
public class Target {
    public final int[] things = { 1, 2, 3, 69, 114514 };
}

// this is a 'duck interface'
// it provides a way for code to call the `modid$expandThings` function safely.
// read more about them here 👇
// https://github.com/LeoCTH/tips-and-tricks/blob/main/minecraft-modding/mixins/duck-interfaces.md
public interface TargetDuck {
    void modid$expandThings();
}

@Mixin(Target.class)
public abstract class TargetMixin implements TargetDuck {
    @Shadow @Mutable @Final
    private int[] things;

    @Override
    public void modid$expandThings() {
        // first, we create a blank array with just one more space for the new element
        var newArr = new int[this.things.length + 1];
        
        // we copy over the existing elements to the new array
        System.arraycopy(this.things, 0, newArr, 0, this.things.length);
        
        // this is where the Mixin magic lies
        // just replace the old array with the new array 🤯
        this.things = newArr;
    }
}

// how to use it:
public class ConsumerClass {
    public void someFunc(Target target) {
        // before
        printThingsInfo(target.things); // >> len=5,content=[1, 2, 3, 69, 114514]

        var duck = (TargetDuck) target;
        duck.modid$expandThings();

        // after
        target.things[5] = 42; // totally fine!
        printThingsInfo(target.things); // >> len=6,content=[1, 2, 3, 69, 114514, 42]
    }

    // prints the length and contents of the array
    private void printThingsInfo(int[] things) {
        System.out.printf("len=%d,content=%s", things.length, Arrays.toString(things));
    }
}
```

## The Long Explanation
Sometimes you need to expand some array that Mojang has mercilessly hardcoded, ~~in the true horror fashion that Mojang is known for~~.

Take a look at this:

```java
package net.minecraft.item;

/* imports */

public abstract class ItemGroup {
	public static final ItemGroup[] GROUPS = new ItemGroup[12];

    public static final ItemGroup BUILDING_BLOCKS = (new ItemGroup(0, "buildingBlocks") {
        @Override
        public ItemStack createIcon() {
            return new ItemStack(Blocks.BRICKS);
        }
	}).setName("building_blocks");

    // snip

    public ItemGroup(int i, String string) {
		this.index = i;
		this.id = string;
		this.translationKey = new TranslatableText("itemGroup." + string);
		this.icon = ItemStack.EMPTY;
		GROUPS[i] = this;
	}
}
```

The `GROUPS` array is hardcoded to only support 12 entries, which corresponds to the 12 item groups in vanilla.

So imagine you're the designer of the new, state-of-the-art modding API. The users of this API expects support for custom item groups and creative tabs, just like other modding APIs like Forge, Fabric and Quilt. What would you do? After all, arrays have a fixed size; there is no possible way to expand them... riiiiiiiiight????

### The Switcheroo
As it turns out, Mixin provides an easy way of doing just that.

A mixin class can access private fields of its target class with the `@Shadow` annotation. Consider this example:

```java
// the target
public class Foo {
    private final int bar;

    public Foo(int bar) {
        this.bar = bar;
    }

    public void doOtherStuff() {
        // other stuff
    }
}

@Mixin(Foo.class)
public abstract class FooMixin {
    @Shadow @Final
    private int bar;

    @Inject(method = "doOtherStuff", at = @At("HEAD"))
    private void doOtherStuff(CallbackInfo ci) {
        // demonstrate our shady prowess
        System.out.println(bar);
    }
}
```

<details>
<summary> 
Sidenote: About <code>@Final</code> and the mysterious disappearance of the <code>final</code> modifier
</summary>

`@Final` is a hint for the Mixin processor to keep an eye out on any accidental writes to the field within the mixin, like `final`.
    
The reason *why* we should't use `final` here is that the Java compiler insists that it needs to be instantiated in the constructor, since `final` fields must have one value and stay that way throughout the lifetime of the object.

But alas, the Java compiler doesn't know any better — in our case, since the field is already initialized, there's no point in assigning a value to it again. Therefore the Mixin team introduced `@Final` to spare some of our (and the Java compiler's) brain cells. God bless them.
</details>

### Let's `final`ly break `final`
Now being able to read them is obviously advantageous, but being able to *write* to them would be even better.

Introducing... the `@Shadow @Mutable @Final` trio.

This set of annotations lets the Mixin processor to allow any writes to the field decorated with them, basically bypassing the entire `final` part of the signature. And since we can reassign final fields anyway, why not just create an entirely new array that it's a bit longer, and reassign the field from the old array to the new one?

This is exactly how the Fabric API expands the item group array:

> The following code uses Yarn/Quilt Mappings.

> The following information is accurate as of Fabric API version `0.37.0+1.17`.
```java
// in net.fabricmc.fabric.mixin.item.group.MixinItemGroup

@Shadow
@Final
@Mutable
public static ItemGroup[] GROUPS;

@Override
public void fabric_expandArray() {
    ItemGroup[] tempGroups = GROUPS;
    GROUPS = new ItemGroup[GROUPS.length + 1]; // expand and reassign!
    
    for (int i = 0; i < tempGroups.length; i++) {
        GROUPS[i] = tempGroups[i];
    }
}
```

Now, I would do things a *bit* differently from how the Fabric API does — my version should be a tiny bit faster by using a native array copy operation builtin to the language:
```java
@Override
public void fabric_expandArray() {
    // also I prefer creating a new array rather than storing the old array as a
    // temporary variable. I don't know which is superior but I think this is mainly
    // a stylistic choice and doesn't matter
    var newGroups = new ItemGroup[GROUPS.length + 1];

    // using `System.arraycopy` to achieve peak array cloning efficiency
    System.arraycopy(GROUPS, 0, newGroups, 0, GROUPS.length);

    // reassign
    GROUPS = newGroups;
}
```

### What about other similarly immutable collections?
So Mojang... has a thing with immutable collections in the more recent versions. As of the date of writing (August 24th, 2021), there are **762** usages of `ImmutableList` *alone* present in Minecraft code<sup id="footnote-1-top">[1](#footnote-1)</sup>, so I'd say there's quite a strong incentive to replace them with more mutable variants should it hamper some mods' needs.

In fact, this is not just an annoyance for modders — there are workarounds for immutability in the Minecraft code itself, but those are not particularly pretty:


> The following code has been manually edited — unwanted decompiler artifacts have been removed and comments have been added to aid comprehension.
```java
package net.minecraft.entity;

/* imports */

public abstract class Entity implements Nameable, EntityLike, CommandOutput {
    // ...

    // using an immutable list for something that's inherently dynamic like
    // passengers on an entity? surely that won't cause problems, right?
    private ImmutableList<Entity> passengerList = ImmutableList.of();
    
    // snip

    protected void addPassenger(Entity entity) {
		if (entity.getVehicle() != this) {
			throw new IllegalStateException("Use x.startRiding(y), not y.addPassenger(x)");
		} else {
			if (this.passengerList.isEmpty()) {
                // 🤨
				this.passengerList = ImmutableList.of(entity);
			} else {
                // copy once
				List<Entity> list = Lists.newArrayList(this.passengerList);
                
                // if this is on a server, the passenger is a player and the
                // primary passenger is not a player
				if (!this.world.isClient && entity instanceof PlayerEntity && !(this.getPrimaryPassenger() instanceof PlayerEntity)) {
					list.add(0, entity); // add the player as the primary™ passenger
				} else {
					list.add(entity);
				}

                // copy twice... hang on.
                // WHY DUPLICATE THE LIST TWICE JUST TO ADD A NEW PASSANGER?!
                // :mojank:
				this.passengerList = ImmutableList.copyOf(list);
			}
		}
	}
}
```
Yeah. They duplicate the passenger list *twice* to add a single passenger.
Naturally, you *can* do something like that, but since Mojang rarely uses immutable collections directly as the field's signature, just please, put a *mutable* collection in its place, especially when you need to modify it later. Like this:

```java
public class Target {
    public final List<String> greetings = ImmutableList.of("hi", "wassup", "yo");
}

public interface TargetDuck {
    void modid$addGreeting(String greeting);
}

@Mixin(Target.class)
public abstract class TargetMixin implements TargetDuck {
    @Shadow @Mutable @Final
    private List<String> greetings;

    @Override
    public void modid$addGreeting(String greeting) {
        if (greetings instanceof ImmutableList) {
            // unmodified; create the mutable replacement

            // also NB: use the constructor directly instead of `Lists.newArrayList`,
            // since that's totally obsolete since Java 7 and somehow Mojang is still using it.
            System.out.println("making the list mutable...");
            var mutable = new ArrayList<>(greetings);
            mutable.add(greeting);
            greetings = mutable; // when the list is no longer immutable 🦀
        } else {
            // mutable; just add to it directly
            System.out.println("adding directly");
            greetings.add(greeting);
        }
    }
}

// how to use it:
public class ConsumerClass {
    public void someFunc(Target target) {
        // before
        System.out.println(target.greetings); // >> ["hi", "wassup", "yo"]

        var duck = (TargetDuck) target;
        duck.modid$addGreeting("owo"); // >> making the list mutable...

        // after
        System.out.println(target.greetings); // >> ["hi", "wassup", "yo", "owo"]

        // add again
        duck.modid$addGreeting("hoi"); // >> adding directly

        System.out.println(target.greetings); // >> ["hi", "wassup", "yo", "owo", "hoi"]
    }
}
```


## Footnotes
<sup id="footnote-1">[1](#footnote-1-top)</sup>Counted using IntelliJ IDEA's builtin Find Usages function, with the scope set to anything under the `net.minecraft` or `com.mojang` packages.