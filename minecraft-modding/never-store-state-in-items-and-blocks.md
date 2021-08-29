# Never store state in `Item`s and `Block`s
> All following code snippets use **Yarn/Quilt Mappings** (version `1.17.1+build.25`), and are up to date as of Minecraft version **`1.17.1`**.

> This article is **up to date** as of August 29th, 2021.

...unless you have a good reason to do so. (Spoiler alert: you probably don't.)

## The TL;DR
`Item`s and `Block`s in Minecraft utilize the [flyweight pattern](https://gameprogrammingpatterns.com/flyweight.html), which  is a way of reducing duplicate data and behavior that are *the same* across different instances.

<details>
<summary>Brief explanation of the flyweight pattern, for those who don't want to follow the link above.</summary>

Seriously though, that is a *good read*. Give it a look when you finish this. I cannot express how useful it is for developers.

In less abstract terms, a — let's say coal — item in a slot in your inventory is *not* an individual `Item` instance; rather, every single coal item that existed, exist and will exist *share* **one** instance.

Rather, the 'items' that occupy slots are actually `ItemStack`s. So when you have nine pieces of coal spread across nine slots, there's nine `ItemStack`s, but only one `Item` instance — not nine.
</details>

If you want per-stack behavior, you should store the custom data in the `ItemStack` rather than the shared `Item` in NBT:

```java
// bad
public class MyItem extends Item {
    public boolean isOn;
    
    // snip

    @Override
    public TypedActionResult<ItemStack> use(World world, PlayerEntity playerEntity, Hand hand) {
        var stack = playerEntity.getStackInHand(hand);
        // flip; on becomes off, off becomes on
        this.isOn = !this.isOn;

        // success but without swinging the hand
        return TypeActionResult.consume(stack);
    }
}

// good
public class MyItem extends Item {
    public boolean isOn;
    
    // snip

    @Override
    public TypedActionResult<ItemStack> use(World world, PlayerEntity playerEntity, Hand hand) {
        var stack = playerEntity.getStackInHand(hand);
        // flip; on becomes off, off becomes on
        this.isOn = !this.isOn;

        // success but without swinging the hand
        return TypeActionResult.consume(stack);
    }
}
```

For `Block`s this problem is even harder to solve, since there are *at least two* strategies to store data — `BlockState`s and `BlockEntity`s.