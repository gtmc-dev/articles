---
translates: ./[进阶]关于4gt树场的一切.zh.md
translated-from-revision: 20ad46927f7ac8db7b9c875504c9a899209214af
banner:
  src: /img/banner.png
  alt: Graduate Texts in Minecraft
title: Everything About 4gt Tree Farms
---

# :advanced Everything About 4gt Tree Farms

_This chapter requires maximum integration of previous knowledge. For parts mentioned before, we will indicate where they were first introduced. ~~hope you enjoy overbraining~~_

<!--由于本章没图完全讲不了，这里充斥着星河的劣质制图，最好换掉-->

Before we begin, we must mention: although detection-based 4gt tree farms do exist, they are quite complex, and **we will briefly discuss them at the end**. All other tree farms here **run on a 4gt clock**.

> Also, since a piston action takes 3gt, you could run the tree farm on a 3gt clock, but obviously this would require using an autoclicker for bonemealing. Traditionally, any tree farm using an autoclicker is not recognized. However, 12gt and 8gt mega spruce have now become exceptions.

## 4gt Birch

No matter what, we still need to start with the simplest: 4gt birch.

Since the 4gt main trunk and base are relatively independent ~~and the base can almost follow a formula~~, we will first discuss the trunk—the core architecture design.

### Core Architecture and Timing Design

Here we discuss two typical and relatively simple architectures:

![4gt白桦架构1](./img/4gt%20birch%20layout1.png)

_This is a very ancient architecture. In a sense, the original designer seems untraceable, but it is also one of the excellent examples for analyzing 4gt design._

First, analyze the timing (it is recommended to try designing it yourself first, then look at the timing we provide):

- First cycle: First, piston 1 at 0t pushes the processing log to the center and pushes the log downward; then piston 2 at 0t retracts the log.

- Second cycle: First, piston 2 at 0t pushes the processing log to the center and pushes the log to the right; then piston 1 at 0t retracts the log, **resetting**.

Obviously, each cycle only contains "one piston action" (_all pistons only perform one synchronized action_), requiring only 3gt to complete, so this can clearly serve as a 4gt tree farm architecture.

> To keep the architecture clear, we removed the leaf processing. You can try designing one yourself.

![4gt白桦架构2](./img/4gt%20birch%20layout2.png)

_This architecture is not as ancient. It was made by Xinghe, and the timing is slightly more complex._

Timing: (all at `0gt BE`) Piston 1 extends at 0t to push out the upper log; then piston 3 retracts at 0t the lower log, piston 4 extends at 0t to push the log downward; then piston 2 retracts at 0t the log horizontally, **resetting**.

This way we also complete the task of processing logs within "one piston action".

> I did not introduce retraction-based architecture design here: its complexity is completely unnecessary for birch. Later in the 4gt multi-species section, we will solve many, many, many problems related to it.

### 4gt Base and Sapling Circulation System

As mentioned in [4.2.1](./为什么你的树场慢——高速树场入门.md), generally we use bottom suction or side suction to handle the tree roots. For 4gt, **three-shot + side suction** is a more convenient choice.

Since the side suction piston completes the retraction action at 2gt TE, we only need to make the horizontally extending piston extend immediately **and retract before 4gt BE ends**, and we can easily design a 4gt side suction root processing.

![4gt根部侧吸](./img/4gt_side.png)

> In fact, bottom suction can also be used. Traditionally, we would use two 3gt dirt-returning bottom suction modules on the x-axis and z-axis respectively; later, hampter published a bottom suction architecture that achieves overall 4gt reset through slime block push-pull, as shown below.

The side suction architecture leaves us a very wonderful AFK position: standing below the side suction, crouching (sneaking), aiming at the upper side corner of one of the dispensers on the left or right, we can start planting trees.

The next issue is sapling circulation and bone meal supply. We generally **do bone meal supply first** ~~because there is a formula~~.

By using droppers to throw bone meal upward like this, pulling it horizontally to the periphery of the tree farm, and connecting it to an unpacker, you get a very standard bone meal supply module. Note that **since clock-based 4gt tree farms only have one tree-growing window every 4gt, we do not use alternating bonemealing, but use synchronized bonemealing**.

For sapling circulation, since 4gt brings a large number of items that need to be processed, we generally use **3-4 droppers** to throw saplings to the player. Since the common wiring method for 4gt is dustless (_especially observer 0t, which we will discuss later_), and there is not much space inside for sapling splashing, **we often need to collect saplings that fly outside the tree farm core**. The conventional approach is to build a wall around the core, then **fill it with hoppers**. Sometimes we also need to collect saplings stuck above the core. Generally, we will **flush them down with water** or **collect them with hopper minecarts**.

After these, you only need to do the wiring, and you can get a 4gt birch tree farm. However, obviously, if using redstone dust, this tree farm will become very laggy. Therefore, we need to understand the dustless wiring method for 4gt tree farms.

## Dustless 4gt Wiring—Dustless 0t Generators

For the overall approach to dustless wiring, it can be roughly said:

- First, we must understand that not everything must run at different depths. For things that "must run at different depths," we actually have **different operational spaces** for **rising edges** and **falling edges**.

- Second, we must understand that **4gt tree farms operate on clocks**. We do not need to strictly design the wiring for starting the clock according to the running timing. **As long as the correct timing can be maintained after everything starts**, the rest is just a matter of turning on/off and resetting. These two things are especially important for dustless wiring. They can help us greatly **save BED devices**, thereby reducing lag.

After understanding these two things, you will understand that **tree farms designed based on dustless 4gt wiring must be modular**. The larger the tree farm, the more this is true.

> For wiring based on redstone dust, the previously mentioned "stringing them one by one in depth order" method is still the simplest, most straightforward, and useful. This is different from the dustless wiring situation.

However, since 4gt multi-species is driven by a clock, **connecting the various parts together** is actually very simple. What we really need to understand is **vertically stackable dustless 0t generators**. With them, we can truly make our dustless 4gt tree farm run according to the timing we want.

### Observer-Related 0t Generators

This is an observer-based, 8gt cycle "single-edge" 0t generator (_i.e., it only produces 0t when pushing down (generally called rising edge here[^1]). For the principle, please refer to the timing theory section of GTMC_). We directly connect it to an 8gt clock here, and all subsequent analysis is conducted under 8gt clock drive.

[^1]: <u>The rising edge concept here is different from what was mentioned before. Here it is clarified: the original meaning of rising edge is the "edge" where the input signal goes from nothing to something (0 to 1) (conversely, falling edge is from something to nothing (1 to 0))</u>

Anyway, since 4gt tree farms run on a clock, we can give **the piston activated by the clock to push down a certain depth**. Theoretically, after pushing the observer upward, if the observer can provide power and the time before removing the observer is enough for the target piston to start pushing, the upward push 0t is completed.

The next question is, obviously the observer will not activate after being pushed up, so what to do? Since the observer can only produce 0t on one edge, we let the observer **be activated during the first downward push**. This way it will only be activated after being pushed up, **and produce 0t when pushed down**.

![4gt0tgen](./img/4gt_0tgen.gif)

> In fact, we can attach a row of blocks to the observer face and move them together with the observer, but that thing is very laggy. I do not want any of you to abuse it. It is only superior to the above design when you need to fill the surrounding space to facilitate sapling collection.

Since the 0t signal produced by this 0t generator has a **rising edge at TT**, we **can only control its falling edge signal**. The method is very simple: just chain a bunch of pistons that update each other.

> Actually, we can do some more optimized processing combined with the wiring of the tree farm itself. Anyway, just give the upward pushing piston a deeper NC update.

> In fact, the rising edge depth is not uncontrollable. We only need to push in a powered block at 0t during the BE phase, and we control the rising edge depth of the power. But this method has a serious problem: it is too large. Many times, if the architecture design does not happen to allow inserting this structure in the middle, we will use redstone dust redirection.

Another method is to directly use sticky pistons (or slime blocks) to push-pull the observer. You only need to make the NC update to the sticky piston have a certain depth to make this 0t work. It is generally only used for rising edge 0t (double-edge is possible but generally unnecessary, you can try it yourself).

### Redstone Dust Redirection-Related 0t Generators

As shown

![redirect](./img/Redstone_Redirect_0tgen.png)
![redirect](./img/Redstone_Redirect_0tgen1.png)

Since the piston next to the piston that pushes down the redirection pillar will emit an NC update when pushing down, **simultaneously updating the note block and the piston that pushes down the note block**, as everyone knows, **note blocks do not add depth**, so the note block will **update all target pistons to make them start extending** before the downward push begins, thus completing 0t.

Redstone dust redirection is **the most convenient 0t method for controlling the rising edge signal depth of 0t** in our dustless 4gt wiring.

> In fact, not everywhere you must control depth. Controlling the activation order of pistons in BE through different activation orders of observers in TT is also possible.

> The biggest problem with redstone dust redirection 0t generators is that you need to trap minecarts, otherwise a large number of saplings will accumulate in the middle, so please use it carefully (but actually it is still a bit more convenient than the observer 0t that pushes and pulls powered blocks).

### Wall Power-Based 0t Generators

As shown

![Wall_0t](./img/Wall_0t.png)

After activating the observer through wall power, use a piston to remove the powered block. A very simple design.

> Actually, it seems not used very much due to volume issues.

At this point, you only need to do the wiring assembly yourself, and you can complete this dustless 4gt birch. [Here we recommend Scorpion's design as a reference](https://www.bilibili.com/video/BV1iN4y1X7o7)

> Actually, if you look carefully at Scorpion's wiring, you will find: we do not necessarily have to use 0t. The 2gt signal given by the observer is also usable. This is also a quite important technique for reducing lag in 4gt tree farms. Folen's 4gt multi-species still suffers from lag compared to things made with Kay and Land's laggier architecture because it uses 0t everywhere.

## 4gt Multi-Species

Since the content of 4gt multi-species is quite extensive, we will go through it bit by bit.

### Branch Trees—Basic Timing for Acacia/Azalea/Cherry Blossom

In [4.3.2](./为什么你的树场慢——高速树场入门.md) we mentioned that we only need to do the diagonally retracting trunk processing designed in TT again on the other side, and we can easily get a 4gt architecture. Here, due to the growth detection requirements of multi-species, **we cannot use the birch architecture**. This pseudo-double-recursion architecture first advocated by Shixiong becomes our only choice.

![4gt全树种架构1](./img/4gtUniLayout.png)

_Here we temporarily use Shixiong's original architecture as an example, but temporarily ignore some additional parts used to process leaves._

If you still remember the content of [3.1.2](./尝试设计一台全树种树场.md), you should be able to see that this architecture can handle branch trees. The question is the timing design.

Currently ~~except for plexi's alien architecture~~ all 4gt multi-species use the branch tree processing timing from Fanhua Qianmu. Here is the timing he designed:

Piston 1 extends at 0t to push out piston 2, piston 2 retracts at 0t the logs on the side of the trunk; after that, piston 3 extends at 0t, piston 4 extends at 0t (if there is a log in front of 4 before 3 extends, then 4 will not extend).

This successfully **first transfers part of the branches out of the processing position**, then performs centering and conventional processing. As for the remaining piston at the bottom, just give it a 0t ~~it won't break even if it can't extend~~.

> Actually, if you remember, you will find: this is the timing TT originally designed for 6gt jungle. But the earliest 4gt jungle tree farm did not use this timing. We will mention the reason soon.

You should be able to imagine that this timing can even handle the situation where the core is completely filled with logs along the x and z axes.

> To increase branch processing capacity, we can actually replace the poor piston at the bottom with double recursion (you can think about other architecture designs yourself. The standard answer is plexi's design).

> Of course, we have more extreme processing: insert a double recursion every other block in the entire row of logs used to activate the side pseudo-double-recursion, and make the entire side pseudo-double-recursion synchronize at an 8gt cycle, only letting the pseudo-double-recursion that directly extracts the trunk run alternately (also plexi's design).

Layer1:

![plexi的逆天4gt全树种架构](./img/4gtUnilayout+.png)

Layer2:

![plexi的逆天4gt全树种架构](./img/4gtUniLayout+1.png)

### [Special Case of Plexi Architecture] Special Handling for Azalea and Cherry Blossom

Actually, if you use Shixiong's original architecture, you theoretically do not need to worry about this. But if you replace the sticky piston pushing logs at the bottom with double recursion, things become much more complicated.

Suppose due to the special growth mechanism of azalea and cherry blossom, we get three branches between the upper suction wall secondary activation logs and the lower double recursion. At this time, if we do not add delay to the rising edge 0t of the double recursion, **the situation will occur where the trunk has not been sucked away and the double recursion cannot push**. After 3gt, the double recursion is pushed out, and you will lose efficiency. For azalea, this will be even worse. **Azalea's growth detection is 3x3**, so after the secondary piston of the double recursion is thrown out, **azalea will not stop growing trees, but will generate logs between the primary and secondary**, then send your secondary piston away.

After plexi inserts double recursion in the middle of the suction wall activation logs, we need to process that side as well. But obviously, **there is no place for you to put rising edge control there**. What to do?

> In fact, the main reason there is no way to control the rising edge is that the log diagonally above the dpe secondary is powered by the observer 1gt before extending. If we do not want it to be powered at this time, we need to make it start retracting after 4gt, which will conflict with the double recursion on the opposite side. Either give up this activation, or it will become the situation of sending away the piston mentioned above. So if we control the rising edge of the primary extension, we cannot use the same secondary falling edge signal as the suction walls on both sides that have no rising edge. Obviously there is no place here to put another different activation. At the same time, if we control the rising edge of the primary (in fact, as long as the signal provided starts later than the observer, it will be finished), after the tree grows, it will prevent leaves and NC update the secondary piston, making it extend early. Your primary cannot push, which is not an ideal situation in any case.

Plexi came up with a good solution: **if it cannot push, just pull the piston away directly**, thereby removing the push at 3gt, thus avoiding throwing out the regular piston. Theoretically there should be quite a few implementation methods. You can try more yourself, ~~although Xinghe and plexi, qontrol, lintex argued for a long time and only the wiring architecture plexi had done before could work~~.

### Jungle Wood

Welcome to the actual biggest boss in 4gt tree farms. Due to jungle's 1/40 sapling drop rate, 4gt tree farms actually only run half of each cycle. Jungle sapling circulation has become the biggest obstacle for 4gt multi-species ~~I'm sure you still remember Luoxi's extreme 100.5% recovery rate back then~~.

We mentioned above that in 4gt multi-species, we abandon TT's original timing for jungle, and **synchronize the pseudo-double-recursion used to extract the trunk on one side with the pseudo-double-recursion on the opposite side**, so that more leaves can be processed in the same cycle, thereby obtaining more saplings.

Can we process a bit more leaves? Let's complete Shixiong's original architecture first.

![4gtUniOriginal](./img/shixiong_4gtUTF_Layer1.png)

![4gtUniOriginal](./img/shixiong_4gtUTF_layer2.png)

It seems plexi increased branch processing while also processing more positions with higher probability of growing leaves. Let's complete it and see.

![4gtplexi](./img/Plexi_Complete1.png)

![4gtplexi](./img/Plexi_Complete2.png)

Obviously, plexi's architecture processes quite a bit more leaves, which greatly alleviates the pressure on the sapling circulation system ~~but this does not mean you can casually pull hopper chains, you still need to do some design~~.

> Considering that jungle sapling drop rate is small, this will cause a large variance in the number of saplings dropped by each tree, which is very unstable. We need some method to cache excess saplings and re-throw them to the player when saplings are insufficient, called active sapling circulation. However, due to the large leaf processing capacity of modern 4gt tree farms, we no longer need its assistance to get a stable and sufficient jungle sapling supply.

In [4.2.2](./为什么你的树场慢——高速树场入门.md) we mentioned that high-speed tree farms often have a large number of drops to process. Obviously, the leaf processing capacity added for jungle is useless for other tree species; on the contrary, to reduce the pressure on hopper chains, **we need to reduce leaf processing capacity**. Therefore, generally we will make all other tree species run under the branch tree timing.

> In fact, you can also imitate perfect timing tree farms and shut down part of the structure, but since modern 4gt architecture has strong integrity, ~~shutting down is almost the same as not shutting down~~, so it's just made lighter.

### Log Output—Suction-to-Push<!--这一坨麻烦BF回头做个动图-->

If you look carefully at the current tree farm, you will find a very serious problem: **every step of the block stream is pulled by an entire row of sticky pistons**. Such a block stream obviously cannot be processed. So here we need to fill a hole: log output for pure retraction-based pseudo-double-recursion architecture.

> As for why not in #4, it is because Xinghe believes that for modern high-speed tree farms, except for 4gt-related and 8gt mega spruce, this part can be avoided through architecture design. If you say that for certain architectures this output method can increase branch processing capacity, then you should know that there is something called PUTF+ in the world, which uses 12gt timing and achieves cherry blossom efficiency exceeding 6gt multi-species. So this processing method is still only necessary for 4gt-related.

The first method, we can notice that the top and bottom of the log stream are empty. We can **pull out logs from above, then push them out horizontally**, thus completing the suction-to-push for the top and bottom two logs. By operating on the log stream in this cycle, we can get a stepped block stream. **This block stream is especially suitable for making wither processing**, because we only need to push the top and bottom two groups vertically together and then horizontally insert them into the wither cage.

![凋零流](./img/4gt_witherstream.png)

The second method, we can add a double recursion module like this at the end of the block stream, thereby **moving the block stream back two blocks at once**, and pushing the block stream out along the way.

![标准流](./img/4gt_StandardStream.png)

Actually, if you think carefully, you will find that the essence of needing these is because **we have no place to put the pistons for pushing out**, so we find a place to put down the pistons for pushing out, then find a way to move the logs over. A common approach is to use honey-slime. **They can stick logs to their sides**, thus providing extra space to place a row of modules similar to side suction, thereby completing suction-to-push.

![蜜绿流](./img/4gt_HSStream.png)

Obviously, honey-slime streams can reduce 0t and lower lag, while shortening the length of the block stream. It is our top priority now.

### Wiring—Integrated/Independent/Modular?

Although we said before that 4gt is driven by a clock, so as long as the timing is correct when running, **we still need to consider the stability and performance overhead of the design**.

For stability considerations, using the same clock and connecting the timing lines that run with the tree farm after the clock is definitely more reliable. For example, the honey-slime block stream mentioned above, **if the timing of the honey-slime wall and other block stream pistons is wrong, those pistons may be stuck by the honey-slime wall and destroy the tree farm structure**. Therefore, for example, in plexi's design, the timing of some block stream pistons is derived from the clock that activates the honey-slime wall. This is **integration**.

However, if all parts are forcibly connected together, this will obviously cause a lot of lag. Therefore, **we also need to connect parts beyond a certain range to another clock**, thereby reducing the performance overhead caused by signal transmission. This is **independence**.

> In fact, we have very, very concise 4gt clock designs, which means that throwing away vertical signal transmission and directly adding clocks can sometimes reduce lag. You can see this design in plexi's honey-slime block stream output section.

Pistons in an area connecting to the same clock will lead to the **modularization** tendency we mentioned before. We can design the wiring of each part separately, and finally assemble them like building blocks. Taking the two 4gt multi-species architecture diagrams shown above as examples, modularization is especially obvious in the upper pseudo-double-recursion wall, **because it does not need to connect with the remaining parts, and as long as the wiring is behind it, it will not cause any additional space occupation and affect assembly**.

At this point, you only need to simply design the timing wiring and connect everything together, and you are done. Obviously, reading the textbook for the first time alone cannot fully understand the timing and wiring methods of 4gt multi-species. I recommend again: **go copy someone else's wiring**. Here I especially recommend plexi's 4gt multi-species. As long as you are a bit serious about dismantling the wiring, based on our previous knowledge, you can definitely fully understand how it works. The only regret is that the release page is on [Tree Hungers Discord](https://discord.gg/8bUbuuS). You need to use a VPN, and you cannot open the release page link at all before joining this server, so you can only search for it yourself (.

Finally, we only have one somewhat niche thing left to discuss: detection-based 4gt tree farms.

## Detection-Based 4gt Tree Farms

Actually, speaking of it, the biggest problem with detection-based 4gt tree farms is: **the only dustless method we can use that can provide zero-delay rising edge 0t** is **redstone dust redirection**. If you try to dismantle the wiring of a few 4gt multi-species or build one yourself, you will find: **it really takes up so much space**.

So now the biggest problem is how to fit all this redstone dust redirection in. But about this I can only say: **architecture is more important than anything. If the architecture is not excellent enough, you have no possibility of fitting it in** (of course you can also hang part of it on the clock ~~but I think such a "detection-based tree farm" has no practical significance~~).

> If you are not afraid of death, you can of course try a redstone dust-based detection-based 4gt multi-species. ~~Xinghe's computer cannot run it anyway~~.

Another problem is that we need some method to make our detection unit **able to run a 4gt cycle**. The simplest and most straightforward solution is of course **to make the output signal structure reset within 4gt**. Another less elegant method is **to make another set, each set resets within 8gt**.

For this kind of detection-based dustless tree farm with extremely compact timing, you can see Qontrol's [dustless detection-based 4gt birch](https://www.bilibili.com/video/BV1tm4y147wg) and [dustless detection-based 6gt multi-species PUTF#](https://www.bilibili.com/video/BV1GH4y1h7ru/) (there is no way, there is no completed dustless detection-based 4gt multi-species yet, can only use 6gt to make up the number).

> However, if you still remember, we said before that 4gt tree farms can also use bottom suction. Obviously, since bottom suction moves dirt, if the sapling has not grown, it will be destroyed, so we cannot connect bottom suction to the clock. At this time, you still need to use some ideas from dustless detection-based 4gt tree farms. But another question is, do you really need to add bottom suction to a 4gt tree farm? ~~Actually you still need to, we will mention this problem again in the dual-core section later~~
