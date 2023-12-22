# What This Is

This is a fix for Project Zomboid's zombie disappearance issues in Multiplayer.

`NetworkZombiePacker.class` fixes disappearances serverside.

`ZombieCountOptimiser.class` fixes disappearances clientside.

Both fixes can be used on any client/server at the same time, though the former will only affect servers and the latter will only affect clients.

This only affects Multiplayer as the issues that are fixed only happen in MP.

# How To Install

Drop both .class files in your zomboid game/server install.

The folder that contains the files to replace is `java\zombie\popman` for servers and `zombie\popman` for clients.

# Code changes

### Culling fix:

| File                      | Line  | Comment                                                              |
| ------------------------- | ----- | -------------------------------------------------------------------- |
| ZombieCountOptimiser.java | 5-7   | removed useless includes                                             |
| ZombieCountOptimiser.java | 10-12 | removed useless variables                                            |
| ZombieCountOptimiser.java | 16    | emptied main offending function (updates count of zombies to delete) |
| ZombieCountOptimiser.java | 20-29 | emptied secondary function for cleanliness                           |
| NetworkZombiePacker.java  | 8     | removed useless include                                              |
| NetworkZombiePacker.java  | 18    | removed useless include                                              |
| NetworkZombiePacker.java  | 61-67 | removed part of loop that deletes zombies upon client request        |

### Stales fix:

| File                      | Line  | Comment                                               |
| ------------------------- | ----- | ----------------------------------------------------- |
| NetworkZombiePacker.java  | 251   | added function call to update square lists of zombies |

# How This Works

Zombie disappearances have 2 separate causes:
- A limit put in place by the devs to prevent performance issues, dubbed "zombie culling"
- A bug caused by an oversight in the serverside memorization of zombie position, dubbed "stale zombies"

### Culling explanation:

Project Zomboid is a game plagued with performance issues. Framerate is often very low in singleplayer. Multiplayer does not escape from this problem and it is made even worse: servers have to worry about multiplicative performance issues as players pile up in numbers, and the effects are not only felt as FPS drops but also *network lag*.

For this reason, Zomboid devs have opted to implement a purposeful limit of 500 zombies around any player. This is a hardcoded limit and there is no way to change it from inside - or outside - the game without hacking the code. If more than 500 zombies are present around a player, they will slowly be *permanently* removed until only 500 are left.

The main issue we players have with this hidden limitation is that for many, Project Zomboid's entire point is to face overwhelming adversity, fight against hopeless odds and be slowly driven to the edge. For many, this means the world needs to be filled with the constant threat of masses of zombies.

While 500 zombies may seem like a lot when starting out, as a player becomes familiar with the game's mechanics and quirks, it becomes easy to deal with. Not only that, but zombie culling is coded in such a manner that several other issues arise, the first of which is that *a player running around a crowded town will cull 99% of its population*.

It would appear zombie culling deletes *older zombies first*. This means while running in a crowded area, the new zombies that appear at the edge of the screen will be the last to be deleted. Running straight through a crowded city will constantly cull every zombie in sight until the very last 500 at the end of your run.

Because the edge of the "render area" where new zombies appear is much farther than the edge of the area visible to the player, with a higher population setting and once zombie culling has taken effect, the player is often left with no zombies in sight - the remaining 500 ones being mostly out of sight at the edge of the render area.

Players are left with no option but to lower population settings in multiplayer until zombie culling does not take effect anymore. Unfortunately, even on the default, x1 population settings, culling will still take effect in some denser areas of the world, such as the mall, or when events trigger exoduses of zombies, such as car or house alarms.

When asking developers to provide us a way to explore the performance impact of higher population settings, we the players were met with distrust and refusal. The only hope was for the change to maybe be bundled in an upcoming update. Because the release schedule of this game is so slow (once every 2 years!), we took it upon ourselves to fix the issue any way we could.

Upon fixing it, we discovered that the performance impact of higher population settings were, in fact, similar to singleplayer in terms of FPS, and perfectly viable in terms of network lag. It would appear the 500 zombie limit is not needed to enjoy a lag-free multiplayer experience, even with dozens of players on a high population server.

&nbsp;

The java fix exists both as a clientside version, and as a serverside version. Because the clients do the zombie counting themselves and request deletion from the server, there are two different places where culling can be stopped.

The clients can be stopped in their counting so that they never request anything from the server. And the server can be stopped in its deleting the zombies, although the clients will still count zombies up eternally (and calculate whether they are out of sight of players), and this will come at an as of yet unmeasured performance cost for clients.

Both fixes are available here as a solution both for server owners who want to fix the issue for everyone, and players who want to fix the issue for themselves should a server be unpatched, and also make sure they don't suffer from the extra useless load of constantly counting zombies up.

Do note that if playing on an unpatched server as a patched client, all other non-patched clients will still send culling requests to the server and if such an unpatched client (player) were to come close to the player, some zombies would get culled anyway, albeit at a slower rate.

&nbsp;

### Stales explanation:

On an unpatched server, kiting zombies over a sufficiently long distance (for about one and a half minutes, usually) causes the zombies to stop in their tracks and disappear. This is unrelated to zombie culling and was in fact only widely recognized as a separate issue once culling was out of the way.

The disappearances are accompanied by clientside logs stating "Removing stale zombie 5000", indicating that it's been 5 seconds since the zombie was acknowledged by the server, therefore the client removed it as a last-resort type measure. But why does the server stop acknowledging the zombie?

It turns out that it's not directly *kiting* the zombies that causes them to disappear, it is rather *moving away from their point of origin*. In fact, the zombies disappear *exactly when* the "chunk" (area of 10x10 squares) containing their point of origin unloads.

Additionally, because zombies move around by themselves (notably from noise events), kiting zombies is not the only way to receive "Removing stale zombie 5000" messages. In fact, even on a default server, logs are often spammed by these messages, indicating that a lot of zombies are being removed improperly by the server.

Fortunately, the removals are not permanent as the zombies are technically _unloaded_ as if they were very far from the player. Also, if a second player stands near the origin chunk of a zombie, that zombie will not "go stale" (disappear because of this issue) since the chunk will not be unloaded when it is kited away.

This bug only affects multiplayer because it is only present in *serverside* code. The reason clients "time out" the zombies (the 5000 message) is because the clients do not remove them when the origin chunk unloads, only the server does.

The reason the server deletes a zombie when its origin chunk unloads is because the server *never properly updates the zombie's position on the world squares*. As such, only the originating square ever keeps the zombie in its "list of things that are standing on me right now". No other squares that the zombie passes through update their lists.

When the servers unloads a chunk (because no player is close enough, to save on memory), it will therefore remove all the zombies *originating from this chunk*, believing that they are still there.

&nbsp;

The fix involves adding instructions to serverside code to properly update squares with zombies passing over them. Thanksfully, as the groundswork has already been laid out, this is done by just adding the right function call in the right place (only one line added).

Because the source of the issue is serverside code, this can only be fixed for *servers*, unlike zombie culling which can be fixed for both.

As a final note, the fix allows servers to properly update position *for zombies that are simulated by players*. The zombies that are simulated by the server will still not properly update the squares they walk over. This, however, is a rare occurence and should not negatively affect the players' experience.

# FAQ

### Do I need to re-replace the files often?

Not for now. The files are reverted back to the original ones whenever you verify file integrity with steam, or whenever The Indie Stone releases an update. Thanksfully they release updates very rarely and they will probably fix these issues on their side in the next big update.

When the devs resume releasing updates and if they don't fix it on their side, you will have to re-replace the files regularly.

### Why are there two files? Which one do I use?

Use both. Replace both. It is just simpler this way.

One is used for servers (`NetworkZombiePacker`), the other for clients (`ZombieCountOptimiser`). However, it doesn't hurt to have both and makes for simpler instructions for everyone.

The reason there are two files is simply because not all the code is located in the same file. To fix every issue, we have to replace both.

### I don't have a dedicated server, I run my server from the game, where do I drop the files?

In the normal game install. If you run a server from the in-game interface, it will use the files in your game install. Reminder: `zombie\popman`

### How do I find my game files?

1. Open your Steam Library

1. Right-click Project Zomboid

1. Manage -> Browse local files.

### My server is hosted on this website, how do I access its files?

Typically by using a ftp app like FileZilla and inputting the ftp address provided by the website.

### There is no zombie folder?

Make sure you are looking at the root of the game installation (you should be inside a folder called something like "ProjectZomboid"). For servers, the zombie folder is located inside the *java* folder so go there first.

### There is no java folder?

I was made aware that the server hosting platform "G-Portal" previously *hid* the java folder from its users for some reason. It might or might not have been fixed by now, if not, on your ftp app input `java\zombie\popman` as file path as *java* might be hidden but its sub-folders are not.

If this doesn't fix your issue or you're on another server hosting platform, contact them first.

### What is this "client" you keep talking about?

In networking lingo, there are "servers" which act as a communication hub, receive all the requests and send back all the data (in video games these are the things you connect to), and "clients" which act as the individual machines connecting to the server and requesting/communicating info to it (in video games, that's you, the player).

When we say "client" we just mean "the instance of the game which is running as an individual player connecting to a server". If you're a player connecting to a server, that's you.

### Why should I trust your files?

The provided .class files are simply compiled .java files. You can see the original .java files with the code laid out bare, as well as the modified .java files from which the .class files are compiled.

You can check the .java files for their code. Our modifications remove bits of code and add *one* line of code. You can compile the .java files themselves, or decompile our .class files.

If you don't know your way around java code/a java compiler/decompiler, you'll have to trust our word that these files are safe.

### Why isn't this a mod?

While it *is* available as a "mod" on the steam workshop, it's true that unlike most other mods, this requires manual installation and can't be simply downloaded and activated in game.

The reason is that the kahlua API of Zomboid does not provide the necessary tools to fix the issue. The offending java code is not exposed to kahlua and therefore, the only way is overwriting it directly.

There is sadly no better way to do this, for now. You'll have to replace your files until a better solution is found/the devs fix the issue on their side.

### How can I decompile Project Zomboid?

For decompiling individual class files, the easiest way is using an online java decompiler such as http://www.javadecompilers.com/

For decompiling the entire game, look at the beautiful-java tool https://github.com/quarantin/beautiful-java (you will have to install git bash to run the bash commands on windows)

### How can I compile the java files myself?

You need to have java 17.0.8 installed somewhere as PZ needs an older java version to compile. We provided the right javac.exe file for you, which should work right off the bat. Or, you can get it from https://www.oracle.com/java/technologies/javase/jdk17-archive-downloads.html (TODO: verify it doesn't need the entire folder, if so add folder)

Once you've installed java 17.0.8 and downloaded the java files to compile, locate your javac.exe and run the following command:

`javac -cp (PZ install) -d (destination) (source)`

Replace the fields with your own file paths, ex:

`javac -cp "C:\Program Files (x86)\Steam\steamapps\common\ProjectZomboid" -d "%UserProfile%\Desktop" "%UserProfile%\Desktop\NetworkZombiePacker.java"`

Any file generated with a $ in its name such as `NetworkZombiePacker$DeletedZombie.class` is useless and can be discarded.

### Why are there 2 steam workshop mods for this?

The mod originally existed as just a zombie culling clientside fix, which was a simple file linked in a steam forum thread, created by Sooner535 and I. Envisioning the potential interest for it, a user called Nippytime took the initiative to bundle the file in a workshop mod he dubbed "Zombie Fix".

The requirements for this fix to be manually installed do not go hand in hand with workshop mods, however, and a lot of confusion arised from the mod. Unfortunately, it is much simpler for people to find the fix through the steam workshop than a google search or a reddit thread.

Along with this initial problem, when releasing the second serverside fix months later which also addressed the stales issue, Nippytime was unavailable to update the mod for a time. He then expressed the wish for a new mod to be created separate from his.

This github repository was created as a better alternative to both the workshop mod (which requires digging through obscure folders) and mega.nz links (which looks like an untrustworthy website), and a way for everyone to clearly see the java code laid out and have a chance at compiling it themselves, for better transparency.

# Additional Links

See https://steamcommunity.com/sharedfiles/filedetails/?id=3115293671 for the steam mod of this fix. Note that the fix still requires manual installation - steam will not allow replacing game files automatically.

See https://steamcommunity.com/sharedfiles/filedetails/?id=2896255721 for an older version which has been taken over by another modder. This version only has an older, clientside-only fix (meaning only ZombieCountOptimiser.class).

See https://theindiestone.com/forums/index.php?/topic/64889-stale-zombies/ for the stales bug report and investigation on official Project Zomboid forums.

See https://steamcommunity.com/app/108600/discussions/6/3364775232101137232/ for the culling bug report and investigation on the Project Zomboid steam forums (since locked by a TIS employee).

See https://youtu.be/8Bhjr9pMjGo for an explanatory video of the issues and the fix
