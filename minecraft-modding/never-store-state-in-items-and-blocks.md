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

So picture this scenario. Imagine you have a `DirtBlock`<sup id="footnote-1-top">[1](#footnote-1)</sup> class that represents, well, a dirt block. And for every dirt block in the earth, on the ground or as a component of your dirt shack, a corresponding `DirtBlock` exists, logically so.

But no, this is simply infeasible for a game like Minecraft – for starters, you might know that blocks can be placed anywhere from -64 to 320 on the vertical (Y) axis, and negative to positive 30 *million* on the north-south (Z) and east-west (X) axes. Doing some quick maths would show that there's 3.456×10<sup>17</sup> blocks in a completely full world, and since each of those aforementioned `Block` instance takes a bit of memory... Yeah, it's not looking good for your system.

On a less extreme scale, let's just say you have 400×384×400 blocks loaded at all times<sup id="footnote-2-top">[2](#footnote-2)</sup>, which equates to 61440000 blocks. 61.44 *million*. Just imagine how much memory it would take to store all of these in memory.

And it's kind of a naive approach anyway - we already know that most dirt blocks are the *same kind of* dirt block as most dirt blocks, why not push that concept further, and say that in fact, *all* dirt blocks are the same thing, but accessed in different places? Similarly for items, all flint items are just flint, all equally good for crafting flint and steel.

This is known as the *flyweight pattern*, and is the strategy Minecraft uses to store block data more efficiently. You can learn more about it [here](https://gameprogrammingpatterns.com/flyweight.html) in an ***excellent*** article by Game Programming Patterns. Seriously, go read it now. I will wait till your return.

With the knowledge of the flyweight pattern, we can immediately see why simply putting a field into `Item`s and `Block`s like, I dunno, *every other Java class*, doesn't work. They *are* weird, but weird for very good reasons.

### That sounds great in theory and all, but *HOW* am I supposed to add data to them?

Okay so I lied a little in the "everything is the same" narrative – obviously when you think about it, an open door can't be the same thing as a close door, can it? Or is a brand-new diamond pickaxe the same to one that has nearly no durability left?

That's why Minecraft actually *stores* individual states, for individual blocks and item stacks! (Sorry for lying to you earlier. It's hard for me as well.) Instead, those shared instances share the things *in common*, like registry names, material, sound when you step on it, how many of those you can stack together, etc.

For item stacks (as in the "items" you can put in a slot), the data-storing object is simply the `ItemStack` itself – it can store a variety of data, in the form of NBT tags.
```java
public void someMethod(ItemStack stack) {
    // you can also use `getNbt`, which returns `null`
    // when NBT data doesn't currently exist on the stack
    // `getOrCreateNbt` creates one if there isn't one present,
    // so it always returns a valid NBT element to play with.
    var nbt = stack.getOrCreateNbt();

    // you can store *all kinds of data* here.
    // booleans, shorts, ints, bytes, strings, ...
    // check out `NbtHelper` for more types you can
    // convert to and from NBT.
    nbt.putInt("tAttUQoLTUaE", 42);

    // just `get` the same type with the same key
    // when you want to use it again
    var theAnswer = nbt.getBoolean("tAttUQoLTUaE"); // should be `42`
    // and it does!
    System.out.println(theAnswer); // >> 42
}
```

For blocks, it gets a bit... *complicated*. See, Minecraft actually provides *two* systems of storing data in individual blocks: `BlockState`s and `BlockEntity`s.

### State your proper properties to use `BlockState`s
`BlockState`s work on *properties* that have a fixed amount of state they can be in.

So for example, a redstone lamp only has one property: whether it's *powered* or not. If it's powered, then it should light up! And if it doesn't... well you know the rest. Since the lamp can only be either on or off, it has only *two* states, like a boolean! Therefore, properties like *powered* are called `BooleanProperty`s.

```java
// DISCLAIMER: this is *not* how the real RedstoneLampBlock looks like.
// I didn't even look it up lol
public class RedstoneLampBlock extends Block {
    public static final BooleanProperty POWERED = Properties.POWERED;

    @Override
    public void appendProperties(StateManager.Builder builder) {
        builder.add(POWERED);
    } 
}
```

There are also all sorts of different kinds of `Property`s: `DirectionProperty`s for a block's facing (north, east, south, up, etc.); `IntProperty`s for properties that can be described as an integral state, like comparator outputs, farmland wetness and crop growth; and finally `EnumProperty`s for other, miscellaneous states, like whether a block is the upper or lower half of a double block, like doors and double plants.

*"But why then do `BlockEntity`s even exist?"*, you might ask. That's because of some limitations on `BlockState`s that are not sufficient in some cases:
- `BlockState`s can only take *exact* and *finite* states. What that means that you can't have more values than what the property allows, which can make for some annoying behavior, such as using an entire `int` to hold a property with only 16 possible variants. <sup id="footnote-3-top">[3](#footnote-3)</sup>
- You'll run out of states real fast. I mean it. Minecraft actually uses a number that maps from and to `BlockState`s, and if you have enough properties with enough states, *bad things may happen*. I'm too scared to test it, but you can go ahead.

Generally, I would recommend `BlockEntity`s for anything that needs to store data other than booleans, small ints and enums. And if you have `BlockState`s with a *lot* of properties, it may be time to rethink about things.

### Fun fact: `BlockEntity`s have nothing to do with `Entity`s
TBC! This is so long that I can't finish it in one session lol

## Footnote
<sup id="footnote-1">[1](#footnote-1-top)</sup>That doesn't exist in modern Minecraft code, by the way. Just a regular `Block` would suffice.

<sup id="footnote-2">[2](#footnote-2-top)</sup>An extremely crude approximation for all loaded blocks in all loaded chunks, assuming the default render distance of 12. The number of loaded chunks should be quite a bit lower than this (given that loaded chunks form a cylinder, not a cuboid), but it proves my point regardless.

<sup id="footnote-3">[3](#footnote-3-top)</sup>Oh and have you noticed that there is no `FloatProperty`? The reason is because properties are *too damn exact*. You might not think of how different 4.2 is to 4.200001, but *your* computer does. So if you want to accept a range of values from 4.15 to 4.25, just bear in mind that there are **209715*** different numbers in between.

If you wanna throw in a boolean property, boom, 419430 states. In short, ***don't do this***.

\*Calculated by subtracting the mantissa part of 4.25 (524288) by 4.15 (314573). Since they are of the same exponent, this should be correct.