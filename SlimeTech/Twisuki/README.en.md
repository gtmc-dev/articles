---
translates: ./README.zh.md
translated-from-revision: cac19431dc289bba701dcf7a21178f480c2e91fc
chapter-title: Twisuki
intro-title: Preface
---

Welcome to this Slime Tech tutorial, meow~ This is Suyang Twisuki.

Inject Twisuki, meow~ Inject Twisuki, thank you, meow~

[Profile](https://twisuki.github.io) [Bilibili](https://bilibili.com/space/317707977) [Github](https://github.com/Twisuki)

All right, enough fangirling. Back to the text. I had heard about this Slime Tech tutorial for quite some time, and at Molforte's invitation I joined the writing group and became responsible for writing this tutorial.

As a Slime Tech researcher (I do not dare call myself a developer; real developers all have quite a few original machines they can show. I can only count as a scholar), I first encountered Minecraft in 2013 and started learning redstone roughly two years later. But I have only been in contact with the modern Slime Tech community for the last three or four years. Although I do not have any particularly impressive machines of my own, I think that long-term, if unsystematic, study can still accumulate a great deal of knowledge and produce some valuable experience. I will exhaust my knowledge reserves and organize them into this small tutorial, hoping it can be of some help to you.

I think learning Slime Tech has roughly the following characteristics:

1. Slime Tech is not an especially arcane field. The gigantic machines you see are, at their core, only combinations of a limited set of components.

   You may marvel at how spectacular and grand an enormous machine looks, but in practice the larger a machine is, the more it proves that each small part of it is simpler, because it has not been compressed to an extreme degree. Once you understand the core and the operating logic, design becomes mostly a matter of wiring.

   Its timing requirements are not as precise as in some other fields. By this I mean that microtiming is basically not needed, and even when it appears, it is modularized; you only need to deal with that module and do not have to interfere with its internal logic. But Slime Tech is strict in that there is almost no fault tolerance: adding or removing a single block can break the machine, and manually changing one small part may also break the entire thing.

   There are very few moving components. Although there are many mechanics, compared with other fields there generally are not many especially magical alternative applications. In other words, you are unlikely to marvel at Slime Tech design at the component level, which greatly lowers the difficulty of understanding the machine.

2. Slime Tech cannot be learned metaphysically. If you do not practice, you will never know where the machine will break.

   Simulating it a thousand times in your head is not as useful as backing it up and running it once. When you think you have calculated the timing correctly, you will definitely have missed some connecting rod, some piston that may create a BUD, or some module stuck together so it cannot push or pull. Slime Tech design must be hands-on.

   If you only study theory without doing anything, you will end up like me: able to produce an elegant small module, but unable to make a complete machine yourself, and unable to build one even if you ignore elegant wiring...

   Testing also requires backups. If you are worried that it will break, it definitely will; if you back it up before testing, it will still break, but at least you will suffer less.

3. As long as you study hard and practice more, you too can design your own machines like the experts.

   Do not believe the statement above. If you learn the knowledge well, you can only learn how to tear machines apart; if you practice well, you can only know how far you are from reassembling what you have understood...

   That is a bit discouraging, but it is true. Do not set your goals too high, because this tutorial only teaches basic literacy and introductory knowledge. We have to admit that the experts are geniuses. If you are the same, you probably do not need this tutorial at all. I cannot design machines myself either, so this is as far as I can teach...

Studying this assumes that you can read Chinese, know how to move, know how to open the inventory, and know how to interact with blocks. If commands or utility mods are involved, you need the ability to read English or use translation software. Most importantly, you must have the will to learn, to read through the tutorial, and to practice hands-on. I will cheer you on.

This tutorial is limited to Java Edition (offline is fine, but NetEase Edition is not included), and specifics should be analyzed version by version. In addition, all mods in this tutorial are used in English, including their names and functions, though the tutorial will introduce them.

If there are omissions, please contact me through the channels noted at the beginning.

Happy learning.

Twisuki Suyang

2025-01-20

---

After finishing Chapter 1, I rethought the purpose of writing this tutorial. Perhaps before preparing Chapter 2, I need to strengthen my own knowledge reserves first. My knowledge comes from practice, and practice is good, but it is also true that it is disconnected. I have fallen into "building behind closed doors," and what is more, the "vehicle" has not even been built yet...

This tutorial will try to follow these principles:

1. Standardization

   In this tutorial, whether for Minecraft itself or for various mods, terms are taken from the English-language interface wherever possible. I will try to use standardized and common terminology to keep the tutorial consistent.

   At the same time, because the Slime Tech field lacks beginner-oriented standard terminology, I will try to give accurate definitions and explanations. But this tutorial has limited influence and cannot become a unified standard for the entire Slime Tech community, so there will be many difficulties during writing.

2. Lightweight setup

   In a technical-Minecraft environment where players often use enormous modpacks, configuring mods yourself or using only a small number of mods is not very common. But for beginners, directly downloading someone else's modpack, or asking others and receiving a large pile of mods, creates a lot of pressure when understanding is insufficient: compatibility problems, keybind conflicts, unknown mod functions, and unclear dependencies and dependents all become hard to solve independently.

   I have always advised beginners not to use ready-made modpacks, nor to ask others for a large list of mods. I mainly write this tutorial using a 1.21.1 client with only 25 mods, and I am very familiar with the function of each one. During the tutorial, I will try to use the functions of existing mods rather than recommend installing new ones.

   My recommended mod list is: `Carpet mod`, the `masa suite` without extensions (`Malilib`, `Tweakeroo`, `Litematica`, `Minihud`, `ItemScroller`), `Client Command`, `pistorder`, and `IMEBlocker`. Please explore other mods on your own.

3. Overall structure

   At the beginning of writing, this tutorial was remade several times, and it has now reached a relatively complete structure.

   The tutorial includes a `Preface`, a `General Outline (Table of Contents)`, the `Main Text`, and an `Afterword`. The main text is temporarily divided into 6 chapters and 14 sections. Each chapter is split into an introductory part and the main content, and the main content contains headings, section headings, and so on, forming a relatively complete structure.

As a beginner-oriented tutorial, this tutorial starts only from the basics of Slime Tech and does not explain other kinds of knowledge. Beginners will need to consult the `Minecraft Wiki` for those.

This updated preface will serve as an editorial standard to follow. I also hope readers can roughly understand the organization and structure of this tutorial and learn more effectively.

Happy learning.

Twisuki Suyang

2025-01-26
