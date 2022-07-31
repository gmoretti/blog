---
layout: post_with_comments
title:  "A bit on online multiplayer games in Godot"
description: Creating a simple mutliplayer fighter game example
date:   2022-07-30 21:00:00 +0200
---
<img src="https://upload.wikimedia.org/wikipedia/commons/5/5a/Godot_logo.svg" alt="Godot Engine" width="350"/>

I wanted to try something related to programming but as different as possible as what I do as a software developer in a day to day basis, so I got back to Godot engine. I had downloaded it and tried it out for the first time during the first months of the pandemic, and did a couple of tests with some 3d models from Blender and I loved it. So this time I wanted to learn a bit how multiplayer would work in a simple 2d game. Just a couple of players and some guns.  

The players should be able to see each other synchronized, be able to shoot and take damage and die. Simple enough. Also some of the tutorials I was following suggested a lobby and waiting room, so I did, because why not.  

Now, a quick YouTube search took me to one [excellent series on multiplayer Godot](https://www.youtube.com/watch?v=lnFN6YabFKg&list=PLZ-54sd-DMAKU8Neo5KsVmq8KtoDkfi4s). This basically has all you need to start and also an explanation  on the network concepts. 
I skipped things related to authentication since I'm obviously not there yet, but all the rest was perfectly applicable to my 2d fighter platformer.  

It was easy enough to get tutorials on basic movement in 2D to allow my player to move around, and another one on how to create tilemaps for your world. I use some free game art from https://opengameart.org/ and move on with the multiplayer tutorials. Now, the following were my main pain points while understanding the code itself.

# Interpolation

For me, the most difficult concept to grasp in the multiplayer environment was implementing interpolation and how to have players play in the past, so they have enough information to interpolate movement between the not so recent past and the recent past. This concept is much better explain in the Game Development Center's series episode 12.  

Here there's another document that can help in understanding the justification behind this solution: [entity-interpolation](https://www.gabrielgambetta.com/entity-interpolation.html)

# Clock Synchronization

Another issue is the fact that we need to account for the latency generated in the server-client communication, when not working you may encounter issues like this one:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Multiplayer is difficult 🥹🥹🥹 <a href="https://twitter.com/hashtag/gamedev?src=hash&amp;ref_src=twsrc%5Etfw">#gamedev</a> <a href="https://twitter.com/hashtag/GodotEngine?src=hash&amp;ref_src=twsrc%5Etfw">#GodotEngine</a> <a href="https://t.co/zfqINjIWvc">pic.twitter.com/zfqINjIWvc</a></p>&mdash; Giuseppe Moretti (@gmoretti) <a href="https://twitter.com/gmoretti/status/1548757827660939264?ref_src=twsrc%5Etfw">July 17, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

Here clock in the bottom right instance is behind so action coming from the server does not get "reproduced" in the correct time.  

Once time is compensated, you can use that same clock to account for damage and/or deaths. You send a timestamp alongside this information when informing clients. And the clients wait for their clocks to match that timestamp to execute the actions.

This is the code in the client. We don't react to the damage until the local clock is equal to the damage time.
``` python
# keys (damage variable) in damage_dict are timestamps
func ReceiveDamage():
	for damage in damage_dict.keys():
		if damage <= Server.client_clock:
            #Reacting to the damage
			current_hp = damage_dict[damage]["Health"]
			damage_dict.erase(damage)
``` 

Now, once latency and clock are managed, things start to look much better.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Some progress on the latency and damage<br> 🙃<a href="https://twitter.com/hashtag/GodotEngine?src=hash&amp;ref_src=twsrc%5Etfw">#GodotEngine</a> <a href="https://t.co/gSpsrwXFMz">pic.twitter.com/gSpsrwXFMz</a></p>&mdash; Giuseppe Moretti (@gmoretti) <a href="https://twitter.com/gmoretti/status/1549111954689736704?ref_src=twsrc%5Etfw">July 18, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

# Notes on Godot

What I liked the most was the speed of iteration, Godot is very lightweight and launching a scene is super quick. Since I did not know about any engines, I first tried Unity and I felt like everything was too big, at least for what I needed. 
I had to get use to how a tiling system works and although it's not specific from Godot the features for tiling were quite easy to work with once you know how of course.

On the other hand, there is a lot of use of String to connect things, string paths to refer to nodes and to signal callbacks. This is cool at the beginning, but when refactoring it makes it a little more cumbersome.


I was interesting to notice how the whole communication was simpler than it felt it would be. The way one server instance process handles one game. If you want more rooms, you need more instances of the server game process. After this principle, you can then use all sort of techniques to spawn your server processes, like basic scripting or containerized solutions. But the complexity at the bottom remains the same. One instance of the server handles one game.

# What have I learned

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">multiplayer from 2 different machines, server handling collisions and interpolation between states. I do have some syncing issues after 10+min because of drfting clock tho <a href="https://twitter.com/hashtag/GodotEngine?src=hash&amp;ref_src=twsrc%5Etfw">#GodotEngine</a> <a href="https://twitter.com/hashtag/gamedev?src=hash&amp;ref_src=twsrc%5Etfw">#gamedev</a> 🤓 thanks <a href="https://twitter.com/GamedevStefan?ref_src=twsrc%5Etfw">@GamedevStefan</a> for the amazin tutorials 👏👏 <a href="https://t.co/zKdkcuIR4j">pic.twitter.com/zKdkcuIR4j</a></p>&mdash; Giuseppe Moretti (@gmoretti) <a href="https://twitter.com/gmoretti/status/1553760728582168579?ref_src=twsrc%5Etfw">July 31, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 

    - Basics on the high level networking implementation in Godot: Remote Procedure Calls
    - Keeping clients and server synchronized, signaling back and forth
    - Have a lobby stage where game starts and can be sychronized
    - Have the server deal with collisions as a source of truth and inform clients
    - Tilemaps for creating 2D levels

# Whats next
The most difficult part for me was the clock and interpolation part. i would like to create a new project and just code again the multiplayer sync part, understanding how to improve it and why I am getting sync issues after some time connected into a game.  

Also, I would like to maybe try a local co-op style focusing in just gameplay and not networking. The idea would be to tryout different mechanics  and test them in different test project, as assignments  to grasp concepts.

# Links and thanks!
Links to the github repos with the server and client projects.  
[https://github.com/gmoretti/godot-online-multiplayer-fighter-client](https://github.com/gmoretti/godot-online-multiplayer-fighter-client)  

[https://github.com/gmoretti/godot-online-multiplayer-fighter-server](https://github.com/gmoretti/godot-online-multiplayer-fighter-server)  

The amazing series where all this is actually explained , from Game Development Center
[https://www.youtube.com/watch?v=lnFN6YabFKg&list=PLZ-54sd-DMAKU8Neo5KsVmq8KtoDkfi4s](https://www.youtube.com/watch?v=lnFN6YabFKg&list=PLZ-54sd-DMAKU8Neo5KsVmq8KtoDkfi4s)  

The amazing series where all this is actually explained , from Game Development Center
[https://www.youtube.com/watch?v=lnFN6YabFKg&list=PLZ-54sd-DMAKU8Neo5KsVmq8KtoDkfi4s](https://www.youtube.com/watch?v=lnFN6YabFKg&list=PLZ-54sd-DMAKU8Neo5KsVmq8KtoDkfi4s)  

Another Multiplayer series for Godot from Rayuse, from which a learned how to create the lobby and waiting room
[https://www.youtube.com/watch?v=MGo06JvYkrA&list=PLRe0l8OGr7rcFTsWm3xyfCOP4NpH72vB1](https://www.youtube.com/watch?v=MGo06JvYkrA&list=PLRe0l8OGr7rcFTsWm3xyfCOP4NpH72vB1)  
