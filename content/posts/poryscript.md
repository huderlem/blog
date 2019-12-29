---
author: "Marcus Huderle"
date: 2019-10-27
linktitle: Poryscript
title: Poryscript
weight: 10
tags: ["poryscript", "pokemon", "game boy advance"]
---


## Introduction

GitHub Link: https://github.com/huderlem/poryscript

As the Pokémon Gen 3 decompilation projects mature, some of my focus has shifted to improving the developer experience. One of the painpoints of working with the Gen 3 engine is comprehending existing event scripts and writing new scripts. Poryscript is a higher-level scripting language that makes writing scripts much simpler. In this blog post, I explain the motivation behind Poryscript and how it works.

I'll be referring specifically to [pokeemerald](https://github.com/pret/pokeemerald) for the rest of this post, but everything also applies to [pokeruby](https://github.com/pret/pokeruby) and [pokefirered](https://github.com/pret/pokefirered).


## Background

As mentioned above, the pokeemerald scripting engine can be painful to work with. Let's try to understand why.

The scripting engine uses a data-driven approach. Scripts are written in bytecode which is composed of commands, comparisons, and jumps. Those bytecode commands are read and interpreted sequentially by the game's scripting engine. (The command interpretation functions are located in [`src/scrcmd.c`](https://github.com/pret/pokeemerald/blob/master/src/scrcmd.c)). The scripting bytecode commands are defined as assembler macros--these are human-friendly names that translate directly into bytes when the code is assembled into the final game ROM. Here is an example of a simple script when the player meets their rival on Route 103:

```plaintext
Route103_EventScript_Rival::
    lockall
    checkplayergender
    compare VAR_RESULT, MALE
    goto_if_eq Route103_EventScript_RivalMay
    compare VAR_RESULT, FEMALE
    goto_if_eq Route103_EventScript_RivalBrendan
    end

Route103_EventScript_RivalMay::
    msgbox Route103_Text_MayRoute103Pokemon, MSGBOX_DEFAULT
    closemessage
    ...

Route103_EventScript_RivalBrendan::
    msgbox Route103_Text_BrendanRoute103Pokemon, MSGBOX_DEFAULT
    closemessage
    ...
```

If you squint your eyes, it's not terribly difficult to follow the logic of this script. It checks the player's gender and displays the opposite-gendered rival's message. However, there are several ergonomic problems that make developer lives difficult.

1. This is one script, but it looks like three separate scripts.
2. Branching logic is verbose and difficult to follow.
3. The actual rivals' text content is located elsewhere, maybe in a completely different file.

As a script grows longer and more complicated, this complexity multiplies until it's one giant hairball. It's not uncommon for a single script to branch > 10 times or even use loops. There is certainly opportunity to improve the experience when using this scripting system.

## Enter Poryscript

Above, we identified three problems with the bytecode scripting. Let's look at how we could address each of them.

> This is one script, but it looks like three separate scripts.

The reason that script looks like three separate scripts is because of branching logic. Every time a conditional branch (also knows as a jump) occurs, a new scripting block is defined. This is because the scripts are implemented as assembler macros, so a label needs to be generated for the branch destination. It would be nice if the destination branch could be automatically generated.

> Branching logic is verbose and difficult to follow.

Branching logic is composed of two parts. First, a comparison happens. This is usually a `compare` command, but there are other commands like `checkflag` which also trigger comparisons. Second, a conditional branch occurs. This is a command that checks the result of the previous comparison and decides whether or not it should jump to the destination. Combining these two operations would remove some verbosity and simplify things.

> The actual rivals' text content is located elsewhere, maybe in a completely different file.

The text is referred to with a label in the scripts above. In order to see what the text actually says, the developer needs to go manually find it. This text might live in a completely different file than the script itself. If the text were somehow directly-inlined in the script, it would remove that cognitive overhead.

Poryscript addresses all of those issues by introducing some high-level control-flow constructs and syntactic sugar. When designing Poryscript, I started by writing out what I felt was a reasonable way to write scripts. The syntax I settled on looks like this:

```plaintext
script Route103_EventScript_Rival {
    lockall
    checkplayergender
    if (var(VAR_RESULT) == MALE) {
        msgbox("Hello, I'm May.")
    } else {
        msgbox("Hello, I'm Brendan.")
    }
    closemessage
    ...
}
```

This makes it very clear what the entire boundaries of the script are. All branching logic is handled by familiar if-else statements. Text is directly inlined to the command, so that you don't have to switch between contexts to know exactly what the rival NPC says. At this time, Poryscript supports many different control-flow structures, such as `if-elif-else`, `while`, `do..while`, and `switch`.  The high-level language is automatically transpiled into the original bytecode scripting language. Let's dive into the challenges behind that.

## Transpiling Into Bytecode

Since the game engine doesn't understand Poryscript, the Poryscript transpiler converts Poryscript language into the bytecode commands that pokeemerald understands. The main challenge with that process is handling branching flow. Recall that the bytecode language needs a new scripting block defined for any branching flow. This means that Poryscript needs to correctly and efficiently create these blocks. The algorithm itself is not as complex as I expected it to be, however, there were many small details that needed to be fleshed out. We know that Poryscript needs to emit many blocks that jump to each other. To facilitate that, we can define each block as having the following structure:

1. List of non-branching commands
2. Condition check
3. Jump to destination Block if condition was met
4. Jump to return Block

By carefully constructing every Block with this format, we end up with an interconnected list of Blocks that represent the entire control flow of the script. Let's look at how that works for the if/else logic in the script above.

![Block Diagram](/blog/posts/poryscript/block-diagram.png)

The blocks form a directed and fully-connected graph. Since each block knows its destination Block(s), the Poryscript transpiler can render the equivalent bytecode script commands by including `goto` statements to jump to each Block. Without optimization, the resulting bytecode script is shown below. (Block numbers can be seen in the suffix of each label.)
```plaintext
Route103_EventScript_Rival::
    lockall
    checkplayergender
    goto Route103_EventScript_Rival_4

Route103_EventScript_Rival_1:
    closemessage
    releaseall
    return

Route103_EventScript_Rival_2:
    msgbox Route103_EventScript_Rival_Text_0
    goto Route103_EventScript_Rival_1

Route103_EventScript_Rival_3:
    msgbox Route103_EventScript_Rival_Text_1
    goto Route103_EventScript_Rival_1

Route103_EventScript_Rival_4:
    compare VAR_RESULT, MALE
    goto_if_eq Route103_EventScript_Rival_2
    goto Route103_EventScript_Rival_3

Route103_EventScript_Rival_Text_0:
    .string "Hello, I'm May.$"

Route103_EventScript_Rival_Text_1:
    .string "Hello, I'm Brendan.$"
```

The result can be optimized further by reordering Blocks to take advantage of "fall through" behavior, which reduces unnecessary `goto` commands. Let's say Block A jumps to Block B. If Block B is moved directly after Block A, then Block A doesn't need a `goto` command to navigate to Block B--it will "fall through" naturally, since it's the next bytecode command. The optimized version of the above script looks like this, and it only requires two `goto` commands instead of four:

```plaintext
Route103_EventScript_Rival::
    lockall
    checkplayergender
    compare VAR_RESULT, MALE
    goto_if_eq Route103_EventScript_Rival_2
    msgbox Route103_EventScript_Rival_Text_1
Route103_EventScript_Rival_1:
    closemessage
    releaseall
    return

Route103_EventScript_Rival_2:
    msgbox Route103_EventScript_Rival_Text_0
    goto Route103_EventScript_Rival_1

Route103_EventScript_Rival_Text_0:
    .string "Hello, I'm May.$"

Route103_EventScript_Rival_Text_1:
    .string "Hello, I'm Brendan.$"
```

## Conclusion

There are many more features of Poryscript not covered here. You can read about them in the official project [README](https://github.com/huderlem/poryscript/blob/master/README.md). Hopefully this post provided some useful insight into how Poryscript works, and why it's useful. I anticipate it will be used by most serious projects based on the Gen 3 decompilations, due to the significant ergonmic benefits it provides. There is also potential for it to be extended to support other scripting engines, like [pokecrystal](https://github.com/pret/pokecrystal).

Feedback and issues should be reported via the Issues tab on [GitHub](https://github.com/huderlem/poryscript/issues).

Thanks for reading.
