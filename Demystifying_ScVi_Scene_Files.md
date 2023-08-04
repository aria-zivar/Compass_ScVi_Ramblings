## **Demystifying Scene Files**

This is just a short write-up on what you can expect when looking into a scene file for Scarlet and Violet, and how Compass was able to make changes and create new scene files.

This isn't a guide, but more a broad view of breakdown of the process and what scene files are, or what the deal with their particular construction is.

I'll spoil the secret right away, though: **It's flatbuffers. It's all just flatbuffers.**

_Flatbuffers all the way down._

---

## Overview

For the context of this write up, scene files refer to the Trinity Scene files (.trscn, .trsot, etc.) found in `/world/scene` and `/world/obj_template`. 

These can be opened using the flatbuffer schema, [`trinity_Scene.fbs`](https://raw.githubusercontent.com/pkZukan/PokeDocs/main/SV/Flatbuffers/scene/trinity_Scene.fbs), and flatc. Pretty straight-forward so far.

### What are scene files for?

They contain data values for basically every object, character, node, template, or otherwise 'thing' in the entire game. 

This includes stuff such as its X, Y, and Z coordinates on the map, its scale and rotation values, which animations or model components to load and use, the distance it can be rendered or stages of rendering, and other visual information like this. It also contains data which associates these templates or objects with specific script functions, the values used for these and other related scripts, references to other values used for various gameplay components, and so on.

These "construct" the world, in essence.

A lot of these objects are instantiated from templates that are stored in other parts and referenced back through their property sheets, and many others are just placeholders for the game's Lua to modify and turn into what we experience.

If you're familiar with the `/world/data` content of ScVi, you know that all of the .bin files are flatbuffers which can be converted into a JSON output, which is directly editable and can be converted back to .bin to create mods quite easily. → This is the foundation of the basic level of modding used by Compass, SV+, and most other non-graphical mods so far released.

However, scene files aren't _quite_ so easy as that.

#### Opening a Scene File

So let's take a look at one of these as an example of what to expect when working with the scene files:

In the directory `scene/parts/field/streaming_event/world_fixed_placement_symbol_`, which is where the in-world placeholder objects are stored for all of the 700+ symbol encounters, if you convert `world_fixed_placement_symbol_#.trscn` (where 0 is for Scarlet and 1 is for Violet), you'll get a .json → It is all flatbuffer content, after all.

When you open this .json, you'll find some basic information about the file in the header, and then the `scene_object_list` field, which contains a array of objects with `type_name` and `nested_type`, and `sub_objects` that themselves may recursively contain these same fields.

And the `nested_type` field is just integers. Hundreds, thousands of them sometimes, depending on the field content. This `world_fixed_placement_symbol` file is almost 600k lines long, after all, and it's almost all because of these `nested_type` fields.

Back to the spoiler part at the top: _It's flatbuffers~_

The `type_name` field tells you what kind of trinity object it is, for which there's probably already a flatbuffer schema ready-to-use for a lot of these in the [`PokeDocs`](https://github.com/pkZukan/PokeDocs/tree/main/SV/Flatbuffers/scene) repo.

All the `nested_type` is just the uint8 breakdown of a flatbuffer, one byte at a time, in decimal format. But on its own, that's pretty useless. In order to do anything with this, you need to write this `nested_type` field out directly as binary data, and then use flatc again with the appropriate schema to convert it into JSON.

This will allow you to view that field like any of the other flatbuffer files, as the results are stored in a .json file which you can open and edit like you're used to if you've done any editing of the `world\data` files.

But if you open that .json you just made, you'll find a new header and ... _oh no, more flatbuffer data?_

#### The Complexity of Scene Files

This is effectively how scene files are constructed: 

Top-level files contain massive indices of flatbuffer data stored as a uint8 array to represent the packed flatbuffer content, and within that flatbuffer content is often stored _more_ flatbuffers. This continues, with more and more flatbuffer content stored in nested fields stored in other nested fields.

Some are stored as different flatbuffer types, some are nested data instead of separate data with sometimes other types of nested types stored within, etc.

They will all have additional flatbuffer content, but it's all just raw data that needs converting no matter how deep you go.

**If you do not have the flatbuffer schema for a particular type then you will need to create your own to unpack the data correctly.**

Now, obviously, the developers don't work with these files in the same format as we are forced to. They have the benefits of IDEs and GUIs that can access the unpacked data, or present it to them in a far more (hopefully, right?) user-friendly fashion. They don't have to manually input numbers for values, because their object placement tool knows the X, Y, and Z coordinates, or a slider can be used to rotate the object or choose values when putting stuff in the world. 

Most of this is likely even just handled automatically according to class templates and the like, and probably the majority of the developers don't even know where their hell their data is actually being stored in the game or that it's put in a format like this.

Unfortunately, we don't get to have those tools to make our lives easy, and so we have to fumble around in the dark, breaking apart every one of these fields only to find more containers of data that have to be opened up the same way. _It's just flatbuffers all the way down for us._

So, you repeat this process of opening the scene file one layer at a time. That `world_fixed_placement_symbol_#.trscn` file starts out at about 683KB, then opens up to a 9MB .json, then opens up to around 27.6MB when all of the flatbuffer content is converted. 

This scene file has nearly 750 top-level objects, and most of these have several more contained within them, of which many also contain even more objects with even more layers of flatbuffers. 

That single scene files results in over **5,000** files when everything is opened and converted for human-readableness. 

And that's just for _one_ version of the game.

This is how every scene file works, though some will be shorter than this ... and some will be _much_ longer.

But, yes, _eventually_, you will reach the end of the nested flatbuffers and finally get a .json that has _actual_ data for you to modify!

Most of this data, however, is out of context, or stored in other formats that don't match up as easily as those found in `world\data`, for instance. Most of the time it'll just be an integer, or some floating-point value, with no identifying name or other context clue from the label. While some of them are pretty obvious, like any three sequential floating-point values in a container will likely be a Vector3 struct for X, Y, Z values. Most of of the fields will be _unk\_#_ as in “unknown”.

If you're familiar with digging through flatbuffers with [FlatCrawler](https://github.com/kwsch/FlatCrawler), and editing or creating flatbuffer schemas (there are a fair amount of unions, but it's all otherwise straight-forward), you can label these values in your schemas, but as most are generic placeholders and just table-type bits of data ... not sure there's much to be gained by labelling most of these values, unfortunately.

To figure out what this data is for, you need to contextualize them with the type of object and what data or references to the construction of the game it would belong in, see how these values differ in other related scene objects (or unrelated ones that use the same or similar flatbuffer construction), or just edit with new values to see what behaviors change (or what breaks!).

Once you edit the fields that you want to change or test changes to, you need to do all of this in _reverse_.

Convert back from .json to binary, take the uint8 values and insert into or overwrite the correct `nested_type` field, convert modified .json back to binary, etc., and remake the top-level file again.

Not gonna sugar coat it: _This will be quite tedious, obnoxious, and annoying. Automate this if you value your sanity._

### Some of the Things Compass Did with Scene File Editing

Compass modifies a number of these scene files, either by changing values (such as changing Penny's Pokémon in her dorm in the post-game to match those used in her battles with the player), or creating entirely new sections of the files to add to the world (such as for the inclusion of Ren, and shiny versions of Rio and Roo outside of Montenevera's Better Buff Bistro).

To quickly break down some of the other scene file modifications we made for Compass, to give an idea of what sort of things can be changed (and there's a lot more, for sure!):

*   Changing Arven's partner Pokémon in the Path of Legends battles so that the model and visuals match the values from the `trdata_array` required editing the scene files and the `trdata_array`
*   The corrected doubles battles positions for major event fights required creating new scene files for every one of these battlefield locations 
    *   This also required modifying the Lua, but that's a separate topic~
*   The new symbol encounters required creating new objects within that `world_fixed_placement_symbol` scene file (and adding new sections to the `fixed_symbol_table_array` data).
    *   This was fairly involved in particular as it includes pathing behavior and other pseudo-AI elements, but it's all just scene data
*   The new raid ally tiers also required modifying scene files 
    *   This also included creating new binary flatbuffer schemas and additional data in trdata\_array to go alongside.
*   Our [Improved Dynamic LODs](https://www.nexusmods.com/pokemonscarletandviolet/mods/26) mod modifies scene files to change the values which we found are used for the LOD changing script.

But it's not like anything we've done for Compass (or released as its own mod in the case of _Improved Dynamic LODs_) is necessarily "unknown" or "hidden knowledge" or anything and is more just a pain in the ass to bother doing. 

The tools you can use to get into and mess with these files have been around since roughly the beginning of ScVi, essentially, but it's just generally unnecessary for the vast majority of mods, as you don't need to even look at the scene files if you're just changing a Pokémon's stats or adding a new Pokémon to a trainer's team, for instance. 

Everything in SV+ is done through changing `world/data` files, and up until recently, the same was true of Compass, so you can make mods that change a significant amount of the game, and overhaul-level mods, without touching the scene files, Lua, or main binary.

In this regard, Compass does not have any special or secret tools. Aria's created some automation scripts as part of our build process that let us work with these files more quickly, which is useful for us but largely meaningless outside the context of Compass (which is why we don't distribute them)

That sort of automation approach is what I recommend anyone take if they want to get in real deep with this massive mess of data — just to conserve some of your sanity, heh~

For example:

*   The `world/data` files are a bit over **6,300** files when converted. 
*   The `world/scene` files break down into over **200,000** files for _one_ version of the game.

_It's just flatbuffers all the way down and that Raboot hole is pretty deep._

The biggest issue in dealing with scene files, though, is finding what you're looking for → After all, [there are 440k .json files to dig through](https://discord.com/channels/949476153251496026/1058203963373142147/1093014042328698961) when you open it all up. And with so much of it being out-of-context, it can be annoying to tell if the data you're looking at really is the one you need to edit to make the modification you're wanting.

It's also really annoying to work with, for sure, because some of these flatbuffers contain upwards of 10 layers of nested flatbuffers, and many of them have 2-20 more flatbuffers. Some of the "simple" directories, like for NPCs just placed throughout the world, have 900+ top-level container objects. So the less you do by hand, the better.

### Closing

There are some things which can be changed from the scene files which we haven't either completed figuring out, or it's on hold as we take a break from Compass modding directly, such as animation framerate fixes (seems like there's a fair amount of scripting components here and a lot of inconsistency that hasn't been worked out), or the expansion of or creation of new event encounters. Some things have been tested and confirmed possible but aren't currently in our active development list, such as new wild trainers in the world and in the Academy Ace Tournament (this requires a _significant_ amount of Lua, but there's scene file components which have been worked out as well).

I just felt like writing a brief tidbit on the topic, since it's an interesting part of potential Scarlet and Violet modding that not many seem to be actively working with. Once you find that it's all just flatbuffers and get the hang of navigating your way through them, you'll be able to dig around pretty readily and start ~breaking~modding things.

Maybe it gives some ideas. Maybe it's just interesting to see how the game is constructed. Maybe next game they'll use a better process and not require a million hash table and flatbuffer lookups everytime something happens or needs to load.
