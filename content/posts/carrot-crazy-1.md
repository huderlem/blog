---
author: "Marcus Huderle"
date: 2019-12-29
linktitle: Reverse Engineering Carrot Crazy - Part 1 - Passwords
title: Reverse Engineering Carrot Crazy - Part 1 - Passwords
weight: 10
tags: ["game boy", "reverse engineering", "carrot crazy", "passwords"]
---

## Introduction

This is part 1 of a blog series where I reverse engineer a Game Boy Color game titled [*Looney Tunes: Carrot Crazy*](https://en.wikipedia.org/wiki/Bugs_Bunny_%26_Lola_Bunny:_Operation_Carrot_Patch). It's a rather obscure game, but I quite enjoyed it when I was a kid. For each part in this series, I will set a goal and document the steps I took to achieve that goal. An example of a goal might be to make Bugs Bunny jump twice as high. I use the trusty BGB Game Boy emulator for a lot of my work, since it has a robust debugger and memory viewer. I'll be building up a disassembly of the game as the series goes on.

*Looney Tunes: Carrot Crazy* is a simple platformer. The player controls Bugs Bunny and Lola Bunny to traverse the 2D levels, while collecting carrots and other things. The first two stages in each level are similar, but the third is always a side-scroller where the boss character chases the player.

GitHub Link: https://github.com/huderlem/carrotcrazy

- [Part 1 - Passwords]({{< ref "carrot-crazy-1.md" >}})
- [Part 2 - Graphics Compression]({{< ref "carrot-crazy-2.md" >}})
- [Part 3 - Level Maps]({{< ref "carrot-crazy-3.md" >}})
- [Part 4 - Entities & Metasprites]({{< ref "carrot-crazy-4.md" >}})

## Goal

> Find all of the level-skip passwords, and create our own custom password.

The *Carrot Crazy* cartridge does not use battery RAM, so it doesn't save any data when the Game Boy is powered off. However, the player is given passwords at certain points in the game, which let the player start at different levels. Let's unearth what all of the passwords are without playing the game!

{{<figure src="/blog/posts/carrot-crazy-1/password-entry-screen.png" title="Password Entry Screen">}}

## Method

In order to find the list of all the passwords in the game, we will need to find the routine that compares the entered password to some list of valid passwords. First, we will locate the RAM location for the player's password input. It's very likely that this will be 3 adjacent bytes. Once we've found the RAM address for the password input, we can set a memory-access breakpoint in BGB to pause execution when that RAM address is read. I expect this to trigger when the password is submitted by the player, since some code needs to compare the password input with some set of recognized passwords.

## Find RAM Location of Password Input

At this point, I know nothing about the game's engine or memory layout. It's a complete enigma to me, but I do know a lot about how Game Boy games operate. RAM is located at address range `$c000`-`$dfff` (when in doubt, consult the [Game Boy pandocs](http://bgb.bircd.org/pandocs.htm#memorymap)). Somewhere in that address range, I expect to find those three password bytes. One strategy I can use to find them is to open the BGB debugger and look at the memory viewer's RAM addresses.

{{<figure src="/blog/posts/carrot-crazy-1/bgb-debugger.png" title="BGB Debugger">}}

Wow, that's a lot of data! How do we know what to look for? BGB's memory viewer updates in realtime, so I should be able to modify my password input by playing the game and see the a corresponding byte change in the memory viewer. Surprisingly, this is often a very effective approach to locating data in RAM. Game Boy RAM is not that large, so I'll just scroll down until I spot a byte that is changing according to the password input I'm changing while playing the game. Luckily, not much is happening in-game when on the password-entry screen, so most of RAM is static.

As I scroll down, I'm noticing that the entire RAM bank 0 (`$c000`-`$cfff`) is static and contains a lot of ASCII text used in the game. I'll tuck that knowledge away for later. Then, I notice a chunk of RAM starting at `$db00` which is rapidly changing. Using the same eye trick, I conclude that it facilitates the sound engine. This is because I can see some of the bytes changing sync with the current background music.

Finally, I see a chunk of RAM at address `$df00` that appears to be changing at the same frequency as the blinking password character's head in-game. Based on the [principle of spatial locality](https://en.wikipedia.org/wiki/Locality_of_reference), I'm pretty confident that the three password bytes are nearby. Sure enough, I spotted the three bytes located at `$dee8`, `$dee9`, and `$deea`. From fiddling around with the password inputs, the values are the following:

```plaintext
0 = Marvin the Martian
1 = Daffy Duck
2 = Yosemite Sam
3 = Tasmanian Devil
4 = Elmer Fudd
```

Now that we know where the password input is stored in RAM, we should be able to find the routine in ROM that is responsible for comparing password input to a known set of passwords.

## Find Password Comparison Routine

The password input bytes must be used by at least two pieces of logic in the game's code.

1. Checking if the entered password is valid.
2. Displaying the blinking character's head to represent the currently-entered password.

As mentioned earlier, I will add a memory-access breakpoint on the password input RAM location. I'll add an on-read breakpoint to `$dee8`, since that's the first password character.

{{<figure src="/blog/posts/carrot-crazy-1/bgb-memory-access-breakpoint.png" title="Defining the Memory Access Breakpoint">}}

Now that the breakpoint is defined, I'm observing that the RAM address is being read twice during every frame. This is a bit annoying, since that makes working with the breakpoint more difficult. The two routines that are accessing the byte seem to be responsible for sanitizing the password byte and loading the appropriate character head graphics. Neither of these is what I'm searching for, since I'm looking for the code that executes *after* the password is submitted. Luckily, BGB has a joypad built into the debugger, so I can tell BGB which keys to press in the upcoming frame. Sure enough, when the password is submitted, a new routine at ROM address `0:0736` triggers the breakpoint.

In the BGB debugger, we see the code for the routine surrounding the breakpoint: (I've replaced some raw addresses with human-friendly labels.)

```plaintext
    ld a, 5
    ld [MBC5RomBank], a
    ld hl, $6e8c
    ld a, [hli]
    ld d, a
.checkPassword
    ld e, 3
    ld bc, wPasswordCharacters ; $dee8
.loop
    ld a, [bc] ; <--This is where my breakpoint triggered.
    inc c
    cp [hl]
    jr nz, .noMatch
    inc hl
    dec e
    jr nz, .loop
    ld a, [hli]
    cp a, $ff
    ...
.noMatch
    inc e
    inc e
    ld c, e
    ld b, 0
    add hl, bc
    dec d
    jr nz, .checkPassword
```

This is a pretty straightforward routine that is iterating over a table of passwords and comparing the player-entered password against them. First, this routine loads the table of passwords. The table is located in ROM bank 5, address `6e8c`, otherwise written as `5:6e8c`. We can see that register `d` holds the total number of passwords in the table. It is the first byte in the password table located at `5:6e8c`. Based on the code after `.noMatch`, we can also see that each entry in the password table is 5 bytes long. I suppose that's three bytes for the password characters, and the other two are some metadata describing which level the password is for.

## Documenting the Password Table

We're almost there! Let's open the cartridge ROM file in a hex editor and navigate to ROM address `5:6e8c` to view the contents of the password table. (`5:6e8c` is address `$16e8c` in the ROM file, since it's in ROM bank 5.)

{{<figure src="/blog/posts/carrot-crazy-1/hex-editor-password-table.png" title="Password Table">}}

It looks like this in its raw form:

```plaintext
Passwords: ; $16e8c
    db 6 ; number of passwords
Password0:
    db $0, $1, $4 ; characters
    db $7, $0     ; stage metadata
Password1:
    db $1, $4, $3
    db $15, $0
Password2:
    db $3, $2, $0
    db $15, $ff
Password3:
    db $2, $4, $1
    db $30, $0
Password4:
    db $0, $2, $3
    db $30, $ff
Password5:
    db $3, $4, $1
    db $ff, $ff
```

Fantastic--based on this password table, the game has six unique passwords. The only issue is that we don't know what exactly the stage metadata represents. From gameplay experience, I happen to know that there are different passwords for levels on easy and hard modes. So, I can deduce that the second byte of the stage metadata is probably `$0` for easy mode and `$ff` for hard mode. That means the first byte is very likely the level id. Let's test it out by changing the values and seeing what happens during gameplay.

Using a hex editor, I've modified the first password to the following:

```plaintext
Password0:
    db $0, $0, $0 ; All Marvin the Martian
    db $2, $0     ; stage metadata
```

This takes me to the language selection screen. This is a bit unexpected, since I would expect it to take me to level 2, perhaps a Yosemite Sam level. After experimenting with that first byte, it seems that the game is operating on a giant screen state machine where different "screens" have global ids. Below is a list of some of the screen ids:

```plaintext
0: Infogrames Copyright Screen
1: Warner Bros. Copyright Screen
2: Language Selection Screen
3: Title Screen
4: Options Screen
5: Bugs & Lola Intro Scene
6: Studio Hallway (Hub World)
7: Treasure Island Scene 1 Intro
8: Treasure Island Scene 1 - Area 1
```

Sure enough, the first password in the table has value `$7` as its screen id, which corresponds to the Treasure Island Scene 1 screen in my list above. If we test the password `$0, $1, $4` (Marvin the Martian, Daffy Duck, Elmer Fudd), the player should be immediately taken to the Treasure Island Scene 1 Introduction.

{{<figure src="/blog/posts/carrot-crazy-1/treasure-island-password.gif" title="Treasure Island Password">}}

Voila! The only thing remaining is what the behavior of the last password is. It has values `$ff, $ff` for the stage metadata, and `$ff` does not seem to correspond with a screen id. From external documentation, I happen to know that the developers left in a debug password which lets the player skip the current stage by pressing START + SELECT during gameplay. That is indeed what this final password is.

In summary, we've found that all of the passwords recognized by the game are the following:

- Marvin the Martian, Daffy Duck, Elmer Fudd
  - Treasure Island Scene 1
  - Easy Mode
- Daffy Duck, Elmer Fudd, Tasmanian Devil
  - Crazy Town Scene 1
  - Easy Mode
- Tasmanian Devil, Yosemite Sam, Marvin the Martian
  - Crazy Town Scene 1
  - Hard Mode
- Yosemite Sam, Elmer Fudd, Daffy Duck
  - The Space Station Scene 1
  - Easy Mode
- Marvin the Martian, Yosemite Sam, Tasmanian Devil
  - The Space Station Scene 1
  - Hard Mode
- Tasmanian Devil, Elmer Fudd, Daffy Duck
  - Ability to skip levels by pressing START + SELECT during gameplay

## Creating Our Own Password

Now that we've fully reverse-engineered the password system, we should be able to create our own password without destroying any of the existing passwords. I want a password that takes me to the last level of the game--Elmer Fudd's Forest.

First, we'll need to move the password table to some other location in the ROM, since I want to append to it. I can't append to it in its current ROM location because I don't want to overwrite the existing data that occurs directly after it. In a hex editor, I can see that there is plenty of free space at the end of ROM bank 5, so we'll move the existing password table to that location.

{{<figure src="/blog/posts/carrot-crazy-1/hex-editor-free-space.png" title="Relocating the Password Table">}}

First, we have to change the first byte to `7`, since there are now seven passwords in the table. Then, I append my 5-byte password structure to the end of the table.

```plaintext
Password6:
    db $4, $4, $4 ; All Elmer Fudd
    db $3d, $00   ; Elmer Fudd's Forest Scene 1, Easy Mode
```

Now that we have our new password table in the ROM, we have to update the routine that loads the password table, since it's still using the old address. We moved the password table to address `$17dd6`, so we will modify the beginning of the routine to the following:

```plaintext
ld a, 5 ; The table is still located in ROM bank 5
ld [MBC5RomBank], a
ld hl, $7dd6 ; New password table address
```

That's it! Let's test to see if our password works.

{{<figure src="/blog/posts/carrot-crazy-1/elmer-fudd-password.gif" title="Custom Elmer Fudd Password">}}

## Conclusion

That wraps up part 1 of this series. We were able to reverse engineer the password system and even inject our own custom password into the game. Hopefully this provided some good insight into basic Game Boy game reverse engineering. In future installments, we'll be going more in-depth as we tackle things like custom levels and game entities.

Thanks for reading.
