# Mod Loader

Mod Loader is a plugin for Grand Theft Auto III, Vice City and San Andreas that adds an easy and user-friendly way to install and uninstall modifications into the game, as if the game had official modding support. No changes are **ever** made to the original game files, everything is injected on the fly, at runtime!

The usage is as simple as inserting the mod files into the *modloader/* directory. Uninstalling is as easy as that too, delete the mod files and you are done. Hot swapping mods while the game is running? By using Mod Loader you can!

Still not sure? Check out [this](https://www.youtube.com/watch?v=TvRpQa8dJ7E) nice video from Ivey. For more, check out our [GTAForums](http://gtaforums.com/topic/669520-mod-loader/) thread and our [GTAGarage](http://www.gtagarage.com/mods/show.php?id=25377) page.

### Building and Installing

Requirements:

+ [Premake 5](https://premake.github.io/download/) *(install with `winget install -e --id Premake.Premake.5.Beta`, or download from the official website)*
+ [Visual Studio](https://www.visualstudio.com/downloads) 2017 or greater.
+ [Windows XP Platform Toolset](https://learn.microsoft.com/en-us/cpp/build/configuring-programs-for-windows-xp) is required; add it from Visual Studio’s Individual Components Installer if needed.

Run the following command in the root of this directory to generate the project files:

    premake5 vs2022

You can install the generated binaries into your game directory by running:

    premake5 install "C:/Program Files (x86)/Rockstar Games/GTA San Andreas"

Or, you might want the files to be automatically installed every time you build the solution:
 
    premake5 vs2022 "--idir=C:/Program Files (x86)/Rockstar Games/GTA San Andreas"
