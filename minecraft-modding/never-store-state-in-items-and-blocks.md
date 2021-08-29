# Never store state in `Item`s and `Block`s
> All following code snippets use **Yarn/Quilt Mappings** (version `1.17.1+build.25`), and are up to date as of Minecraft version **`1.17.1`**.

> This article is **up to date** as of August 29th, 2021.

...unless you have a good reason to do so. (Spoiler alert: you probably don't.)

## The TL;DR
`Item`s and `Block`s in Minecraft utilize the [flyweight pattern](https://gameprogrammingpatterns.com/flyweight.html), which  is a way of reducing duplicate data and behavior that are *the same* across different instances.

<details>
<summary>Brief explanation of the flyweight pattern in Minecraft, for those who don't want to follow the link above.</summary>

Seriously though, that is a *good read*. Give it a look when you finish this. I cannot express how useful it is for developers.

In less abstract terms, a — let's say coal — item in a slot in your inventory is *not* an individual `Item` instance; rather, every single coal item that existed, exist and will exist *share* **one** instance.

Rather, the 'items' that occupy slots are actually `ItemStack`s. So when you have nine pieces of coal spread across nine slots, there's nine `ItemStack`s, but only one `Item` instance — not nine.
</details>

If you want *per-stack* behavior (which is frankly, most of the time), you should store the custom data in the `ItemStack` rather than the shared `Item` in NBT:

```java
// assuming all other code call `setOn` and `isOn`
// to get the state

// bad; once one flashlight is turned on, all others turn on as well
// that can't be good, right?
public class FlashlightItem extends Item {
    // if the flashlight is on
    public boolean isOn;
    
    // snip

    public void setOn(boolean on) {
        this.isOn = on;
    }
    
    public boolean isOn() {
        return this.isOn;
    }
}

// good
public class MyItem extends Item {
    // since each stack of flashlight is different,
    // we need access to the ItemStack itself
    public static void setOn(ItemStack stack, boolean on) {
        // stores the information on the NBT data of the stack
        stack.getOrCreateTag().putBoolean("on", on);
    }

    public static boolean isOn(ItemStack stack) {
        return stack.getOrCreateTag().getBoolean("on");
    }
}
```

For `Block`s this problem is even harder to solve, since there are *at least two* strategies to store data — `BlockState`s and `BlockEntity`s.

(Just FYI, this is bad *in most cases*:)
```java
public class MyBlock extends Block {
    public Direction facing;
}
```

Generally, for blocks with properties with *fixed*, *few* alterations – such as facing, power state, growth state, etc – you should use `BlockState`s:
```java
public class MyBlock extends Block {
    public static final DirectionProperty FACING = Properties.FACING;

    @Override
    public void appendProperties(StateManager.Builder builder) {
        // adds facing to the block as a state
        builder.add(FACING);
    }
}
```

For more complex data, like storing fluid data in fluid tanks, items like in inventory blocks (e.g. hoppers, chests, shulker boxes and furnaces) and more should be stored in `BlockEntity`s. I won't talk about them in detail here because it's another beast on its own.

## The Long Explanation
~~TBH, the TL;DR is atypically long already~~
