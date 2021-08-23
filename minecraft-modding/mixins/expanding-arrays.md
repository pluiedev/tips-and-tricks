# Expanding Arrays
> The following code uses Yarn/Quilt Mappings.

## The TL;DR
```java
public class Target {
    private final int[] things;

    // initialization omitted
}

// this is a 'duck interface'
// use `((TargetDuck) target).modid$addThing(thing);` to access the new method
// read more about them in <link pending>
public interface TargetDuck {
    // NB: don't downplay potential naming clashes!
    void modid$expandThings();
}

@Mixin(Target.class)
public class TargetMixin implements TargetDuck {
    @Shadow @Mutable @Final
    private int[] things;

    @Override
    public void modid$expandThings() {
        // first, we create a blank array with just one more space for the new element
        var newArr = new int[this.things.length + 1];
        
        // we copy over the existing elements to the new array
        System.arraycopy(this.things, 0, newArr, 0, this.things.length)
        
        // this is where the Mixin magic lies
        // just replace the old array with the new array
        this.things = newArr;
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
public class FooMixin {
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

