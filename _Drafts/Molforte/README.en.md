---
translates: ./README.zh.md
translated-from-revision: cac19431dc289bba701dcf7a21178f480c2e91fc
chapter-title: Molforte
intro-title: Overview
---

## Goals

This chapter aims to give readers the ability to design Slime Tech machines and, at the very least, complete an introductory pass through Slime Tech so they can analyze simple Slime Tech machines.

At present, this text is based only on Molforte's own limited perspective. **It does not represent the views or design philosophy of the entire Slime Tech community**, so I hope readers will approach it critically.

## What Is Slime Tech?

**As for the explanation of Slime Tech**, the Wiki already has an answer:

> [Flying technology](https://zh.minecraft.wiki/w/Tutorial:%E9%A3%9E%E8%A1%8C%E5%99%A8 "Tutorial: Flying machines") based on [slime blocks](https://zh.minecraft.wiki/w/%E9%BB%8F%E6%B6%B2%E5%9D%97 "Slime Block"). Broadly, it refers to redstone technology centered on slime blocks; narrowly, it specifically refers to slime-block piston-worm technology.

On that basis, my explanation is:

> Slime Tech originally refers to a green, cute block (the slime block) or creature (the slime). By extension, it refers to redstone technology centered on slime blocks, including the slime-block portions of various machines and flying machines. In the narrow sense, it refers only to flying-machine technology. Note that we generally also include honey-block-related technology under Slime Tech.

Slime Tech and mobile machinery (mostly movable battleships, aircraft, and similar builds) are essentially the same kind of technology, but this chapter currently focuses only on survival-oriented flying-machine technology. Readers can transfer the ideas on their own, or I may look for a suitable mobile-machinery expert later.

Slime Tech is closely related to Mechanical Redstone. Slime Tech can be viewed as movable Mechanical Redstone technology, but the implementation approaches differ greatly.

## Prerequisites

To keep you from getting lost while reading this text, before formally reading it we assume that you are familiar with the basic usage of vanilla Minecraft redstone components, and that you already know how to use foundational technical Minecraft mods such as Tweakeroo, Carpet, and Litematica.

---

## Preface

This is a preface.

But it was written as an overview instead. Since it is just a demo, feel free to take it wherever it is useful.

I am Jimu.

This tutorial aims to explain how to gain the basic ability to use flying machines when designing flying wiring for technical Minecraft machines and when playing survival.

The tutorial assumes that you already understand the basic usage of components and have some foundation in microtiming.

---

**A flying machine is a special kind of Mechanical Redstone.**

A flying machine is a structure in vanilla Minecraft that can move itself several blocks as a whole.

That said, as long as it can move, it counts. ~~How is a two-way piston worm with an external signal source not a flying machine?~~

~~A TNT cannon might not be unable to count as a flying machine either.~~

I believe that **the essence of a flying machine is a movable way to route signals; it is not fundamentally different from fixed signal routing**.

Whether to use a flying machine, and to what extent, depends on the specific function you want to implement.

---

Technical Minecraft is a playstyle that requires circuits to be physically built. When the function is the same, the less manpower and material cost required, the better. If it is worth it, you can even trade some running speed for that. For example, using a world eater for clearing is obviously cheaper than an entire dark mass of TNT dupers. Because of this, flying machines still hold a significant place in today's environment of large technical projects.

For that reason, flying-machine design in technical Minecraft has slightly different priorities from other branches of Slime Tech.

Common uses of flying machines in technical Minecraft include large-span engineering machinery, such as three-way machines and world eaters; short- and medium-distance signal transmission and Mechanical Redstone applications; long-distance transport; disposable vehicles; and the extraction of non-renewable materials.

---

Simply put, the principle of a flying machine is **stepping on your own foot to rise into the sky**. After all, Minecraft blocks have neither gravity nor conservation of momentum.

In a flying machine, the structure that implements this principle can be called the **engine**. Parts outside it that exist for function or for added piston load and do not participate in the "stepping" can be called **extensions**.

Of course, you can also divide the structure according to the machine's actual purpose ~~and give each part an extremely impressive name~~. For example, in a tunnel-bore group, you might identify the drill head, connecting rods, timers, and so on. It does not really matter how you divide things, and overlap is fine too. As long as the terms clearly refer to something, nobody cares. However, to highlight the explanation of flying-machine principles in this tutorial, unless otherwise noted I will consistently use the engine-plus-extension model.

According to the number of directions in which the engine can fly, flying machines are classified as N-way flying machines. For example, a machine that goes and never returns is one-way; one that shuttles back and forth is two-way; adding a side pusher to a two-way machine makes it three-way. Under this classification, a world eater is also a three-way flying machine. On this point, technical-Minecraft flying machines differ somewhat from other branches of Slime Tech: for survival technical builds, a flying machine only needs to reach the destination and accomplish the goal, so reserved stations are acceptable.

Flying machines also have stations where they dock. A two-way flying machine is usually divided by its stopped positions into a departure station and a return station, though it can also have neither, or have more stations added. ~~You can also build it in place and let it complete its mission in spectacular explosions.~~

That is all.
