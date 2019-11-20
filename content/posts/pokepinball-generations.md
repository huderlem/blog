---
author: "Marcus Huderle"
date: 2019-11-20
linktitle: Post-Mortem - Pokémon Pinball Generations
title:  Post-Mortem - Pokémon Pinball Generations
weight: 10
tags: ["twitchplayspokemon", "pokemon", "game boy", "reverse engineering", "data analysis"]
---

## Introduction

GitHub Link: https://github.com/huderlem/pokepinball-generations

Two years ago, I collaborated with some folks at [Twitch Plays Pokémon](https://www.twitch.tv/twitchplayspokemon) to bring an enhanced Pokémon Pinball experience to the stream. There were many pieces that had to come together to make this possible, and the end result was quite fun and rewarding.

To give some background context, *Pokémon Pinball* is a Game Boy Color game in which you catch and evolve Pokémon made by developers Jupiter and HAL Laboratory. There are two different stages--the Red and a Blue stages. This game was made in the early days of Pokémon when there were only 151 Pokémon in existence. The Twitch Plays Pokémon stream always has a randomly-simulated game of *Pokémon Pinball* running in the bottom corner of the screen.  Stream viewers can place bets on each game and win prizes depending on the final score of the simulated game. Additionally, a random stream viewer is awarded a badge anytime a Pokémon is caught in the simulated game. Since *Pokémon Pinball* only contains the original 151 Pokémon, viewers rack up lots of badges for the same species of Pokémon over and over. The idea was that the viewer's experience could be more fun if the number of Pokémon were expanded.

In the end, we added two new stages to the game--the Gold and Silver stages. These featured the 100 new Pokémon found in the 2nd-generation games, *Pokémon Gold and Silver*. Let's walk through everything that needed to happen to bring that idea to life.

## Disassembling and Reverse Engineering the Game

If you want to modify a game, you obviously need to first understand how it works! There are several techniques available to reverse engineer and make modifications to a Game Boy game. It all starts with first obtaining a dump of the ROM--the game data on the cartridge. At that point, you have a binary blob that is made up of two things: code and data. Using an [emulator with debugging capabilities](http://bgb.bircd.org/), you can inspect different parts of ROM and RAM until you eventually understand the entire game engine. For this project, a full disassembly was created for *Pokémon Pinball*, which means the actual source code and assets for the game were completely reverse engineered. The function names and labels for things do not match the original source code, but the resulting game ROM is identical when compiled. This means you can make modifications at the source code level and compile the code to obtain a new game ROM. That disassembly project can be found here: https://github.com/pret/pokepinball

I'm purposely skimping on details here, since that is a whole can of worms. Fortunately, I had already made significant progress on the [pokepinball disassembly](https://github.com/pret/pokepinball) before this collaboration with Twitch Plays Pokémon began.

## Making Pokémon Pinball Generations

Now that the game was fully modifiable via the disassembly source code, it was time to plan out features for the game. The following functionality was desired:

1. Add all 100 new Pokémon from *Pokémon Gold and Silver*
2. Add new Gold and Silver stages
3. Update high scores and stage select screens to support Gold and Silver stages

By far, the most challenging part about this process was creating the new art assets for the Pokémon and stages. We were not commissioning artists, so some of us had to roll up our sleeves and learn some pixel-pushing skills. I'll briefly touch on some of the interesting points for each of the pieces above.

### Add 100 new Pokémon

Internally, the game engine treats Pokémon species as a single byte, which is an 8-bit value. For example, [Squirtle's species value](https://github.com/pret/pokepinball/blob/master/constants/pokemon_constants.asm#L9) is 7. Since one byte can only hold values from 0-255, this means that the game engine can only support 255 total Pokémon. Luckily, there are only 251 Pokémon in *Pokémon Gold and Silver*. However, there are now >= 890 Pokémon in existence, and we want to be able to add more in the future. Therefore, the game engine needs to support multi-byte species values. I updated the entirety of the game's code to treat species as 16-bit values, as well as updating the data two be represented with two bytes. Luckily *Pokémon Pinball* is not a very large game, so doing this was manageable.

The process of adding a new Pokémon required one or two new art assets:

1. Portrait which is displayed in the main "billboard" area during gameplay.
2. An optional animated sprite sheet used when catching a Pokémon during gameplay.

{{<figure src="/blog/posts/pokepinball-generations/hoothoot.png" title="Portrait and Catch Animation">}}

Matching the original artistic style of the game was a challenge when creating these images. The portrait images also have unique palette restrictions on them, which are defined as follows:

1. Define two individual four-color palettes
2. Assign one palette to each of the 8x8-pixel squares that make up the portrait image.

Let's look at Bulbasaur's portrait to see the palette restrictions in detail.  You can see that its eye and mouth tiles are carefully arranged in the grid such that they don't overlap with any dark green colors.

{{<figure src="/blog/posts/pokepinball-generations/palette-restrictions.png" title="Portrait Palette Restrictions">}}

To help artists adhere to these restrictions, I created an in-browser palette validator tool. This allowed people to easily check if their image was valid before handing it off to be inserted into the game. Unfortunately, the algorithm was flawed, so false negatives were possible. Overall, this tool was a huge benefit to the project, since it reduced a lot of potential confusion from people who aren't intimately familiar with the game engine.

{{<figure src="/blog/posts/pokepinball-generations/validator.png" title="Palette Restriction Validator">}}

### Add Gold and Silver Stages

Initially, we wanted to create entirely new stage layouts for Gold and Silver. However, that would have required creating a stage editor, and we didn't have the patience to do that. Instead, we opted to clone the Red and Blue stages and give them a different color so they appeared Gold and Silver. When implementing the Gold and Silver stages, I wanted to make them completely separate from Red and Blue in the engine. This was to allow future modifications to the Gold and Silver stages without affecting Red and Blue. Therefore, I opted to duplicate all of the stage data and stage logic and partitioned them into independent Gold and Silver stages. In other words, they are completely independent stages from Red and Blue as far as the game engine is concerned. Their data just happens to be identical to Red and Blue (collision, graphics, etc.).

Now that we had playable Gold and Silver stages, we had to add in the unique data to make them feel like different stages from Red and Blue. As mentioned above, the colors were changed, but the bigger changes were the map locations, catchable Pokémon, egg-hatching, and a special GS Ball upgrade.

During gameplay, the player is always located in a certain map location. For example, the player can start in Pallet Town and move between locations using the Map Move bonus. The current map location determines which Pokémon can be caught. Since the Gold and Silver stages are based on *Pokémon Gold and Silver*, they feature Johto locations such as New Bark Town, Ruins of Alph, and Mt. Silver. Below is a depiction of all the maps available. The Gold and Silver map locations are on the right half, and the original Red and Blue locations are on the left half. These images had identical palette restrictions to the Pokémon portraits explained earlier. We did a decent job of replicating the original art style, except we could have opted for brighter colors in general. (See if you can guess which locations each of the pictures represent.)

{{<figure src="/blog/posts/pokepinball-generations/allmaps.png" title="Map Locations">}}

One special feature that was added to the Gold and Silver stages is the egg-hatching mode. When the player tries to evolve a fully-evolved Pokémon in the Red and Blue stages, it starts a "training" mode which is identical to the evolution minigame. The end result is a simple points reward. In the Gold and Silver stages, this will trigger an egg-hatching mode instead. The minigame mechanics are identical to evolution, but the end result is a baby Pokémon or the first-stage of the full-evolved Pokémon. This is the only way to obtain baby Pokémon like Elekid and Igglybuff.

Another special feature is the GS Ball upgrade. This is a ball upgrade that occurs after the Master Ball during gameplay. The GS Ball is a special item that can be used to encounter Celebi if the player is also in the Ilex Forest map location.

{{<figure src="/blog/posts/pokepinball-generations/gsball.png" title="Silver Stage with GS Ball">}}

### Update High Scores and Stage Select Screens

Since the game keeps track of high scores for each stage, the Gold and Silver stages needed to be supported. This included updating RAM to save those high scores, as well as building new UI to support switching between the different stages when viewing high scores. I kept the visuals the same, but the colors reflect each of the stages.

{{<figure src="/blog/posts/pokepinball-generations/highscores.png" title="High Scores Screens">}}

I decided to completely redesign the stage select screens so that they feature the mascot Pokémon for each stage. In the original game, the screens show a zoomed-out view of the actual stage appearances.

{{<figure src="/blog/posts/pokepinball-generations/stageselect.png" title="Stage Select Screens">}}

## Determining Catchable Pokémon Locations

Now that we've covered the general development of Pokémon Pinball Generations, we can look at how we determined which Pokémon could be caught at which map locations. This was an important problem for Twitch Plays Pokémon because of the badge distribution to viewers. We wanted to make sure that certain Pokémon had the desired rarity. An in-depth approach was taken to solve this problem.

Luckily, Twitch Plays Pokémon keeps a log of every caught Pokémon during simulated gameplay. Therefore, we had a good idea of what kind of rarity distributions we wanted to preserve when introducing the Gold and Silver stages. The other factors we wanted to understand were distributions about in-game events. Some examples were:

1. What is the distribution of ball upgrade types when catching a Pokémon?
2. What is the distribution of map location groups when catching a Pokémon?
3. What is the average time remaining when successfully catching/evolving a Pokémon?

To answer these questions, we simulated the game thousands of times and injected custom logging into the game to track gameplay events we cared about. [BizHawk](https://github.com/TASVideos/BizHawk) is an emulator that has Lua scripting support. To emit logged events out of the game, we defined a RAM address as the "log trigger". The Lua script would write to a JSON file every time that RAM address was written to. By writing certain values to related RAM addresses, the Lua script would know which type of event was meant to be logged and any associated metadata with the event. With all the logging events in place, it was just a matter of gathering an acceptable amount data to analyze. I left about 8 instances of BizHawk running at 600% speed overnight, which resulted in roughly 3,000 pinball games. After combining the resulting JSON files, I wrote a Python script that produced descriptive charts to use when determining game balance questions. Are there concerns about statistical significance with only 3,000 games? Sure, but these numbers were good enough for our purposes. Some example charts are provided below.

{{<figure src="/blog/posts/pokepinball-generations/report_evolution_stages_blue.png" title="Evolution Stages">}}
{{<figure src="/blog/posts/pokepinball-generations/report_rare_mons_blue.png" title="Rare vs. Common Pokémon">}}
{{<figure src="/blog/posts/pokepinball-generations/report_time_left_catchem_evolution_blue.png" title="Remaining Minigame Time">}}

## Conclusion and Future Work

As mentioned earlier, this game is currently running on the Twitch Plays Pokémon stream, so the project was an overall success and fun time. Hats off to those involved in the project. It's certainly a memory I'll never forget and something I'll love talking about with others for a long time. The project has been left in an open-ended state, though. There is opportunity to add in new stages and Pokémon for all of the future main series Pokémon games. There is also a huge opporutnity to build a stage editor to enable building custom collisions, features, and graphics for the new boards.

Thanks for reading.
