# Real Server Conditions
#### Overview
In this doc I explain what I learned going from an editor-only prototype to an online playable demo with >100 matches.

**Relevant Project Info:**
* Servers: *AWS ec2* instances (t2.micro, t2.small, and t2.medium)
* Networking: Default Unreal Engine RPCs and Repvars
* CSP: Custom NetTick system (not the Character Movement Component)

>I do not use the CMC, but instead created my own CSP solution (Net Tick), but this doc is geared toward using the CMC.

**Resources:** [Discord Community](https://discord.gg/uQjhcJSsRG), [Custom Movement Data](https://docs.google.com/document/d/1UO6Ww6Lfpti3YElVdo9uioTUtQJQ9CoSLvd9kF8hvJo/edit?usp=drive_link)

I also created a qualitative demo showing how my game feels under different networking conditions for you to gauge your game’s networking: [Astro Networking Performance Demo](https://youtu.be/NcQWXgmGsKM)
### Simulation Accuracy
So the biggest part of networking is just minimizing the experience of lag and rubberbanding, the goal is to make players feel like the experience is just as smooth as offline. We know that this means combating packet loss and lag. Lag is easier to fight, as the entire CSP framework of the CMC is built around combating lag. As long as all variables that affect movement or simulation in general are movement safe, then lag won’t ever be an issue, since the server might as well be simulating ten years in the past, but the CMC state is still identical. Packet loss is way harder, this is the result of packets getting dropped like a jump input or W key. To an extent, you can’t fully protect against packet loss, inevitably if you drop an input, the simulation can’t reconcile and the client will need to be corrected. The main challenge is fighting against loss, detecting when loss happens, and gracefully recovering to the server's authoritative state (on the client). Packet loss can not be more overstressed! It is the main cause of glitches, poor simulation accuracy, and a buggy feeling game. Always emulate packet loss!
#### Fighting Loss 
Simple, just send duplicate message of the same inputs, but if your using the CMC, don’t worry as this is already done for you through Old Move replication (sends most important, redundant, old move)
#### Detecting Loss
The framework to do this is already built into the CMC, and it's the [Custom Movement Data](https://docs.google.com/document/d/1UO6Ww6Lfpti3YElVdo9uioTUtQJQ9CoSLvd9kF8hvJo/edit?usp=drive_link) system. Basically, if you add additional movement data (eg SprintSpeed) and it's not simply a compressed flag, then this value could (through packet loss) get out of sync between server and client, but the server will not know to correct the client unless they know about the error. By serializing additional prediction data (eg SprintSpeed) as client predicted data, the server can use this to check its authoritative value and then possibly issue a correction. That said, usually the default prediction data is enough because something like SprintSpeed will pretty obviously affect the prediction Location, which is already being sent as default prediction data, so sending SprintSpeed would likely be redundant since anytime SprintSpeed is incorrect, Location is also incorrect. Also, prediction data is sent very frequently (up to three sets of prediction data per frame), so it's not a good idea to send too much data here.
#### Recovering Loss
 To recover from loss (after it has been detected) means also using [Custom Movement Data](https://docs.google.com/document/d/1UO6Ww6Lfpti3YElVdo9uioTUtQJQ9CoSLvd9kF8hvJo/edit?usp=drive_link). So for example if SprintSpeed becomes desynced because the server dropped the prediction where the client pressed the sprint key (and now client is sprinting but server isn’t), then a default correction wouldn’t suffice because the server would correct the client’s Location, but their SprintSpeed wouldn’t change, and then in the next frame the client would sprint again and get corrected again. So you need to send SprintSpeed in the Custom Response Data. For this, send all variables, corrections happen infrequently and it is of the utmost importance for the server to correct all variables on the client that affect simulation so the client gets back in sync instantly. You also have to make sure beyond just sending the data to the client that the client will correctly apply these values, but this is usually as simple as SprintSpeed = CorrectedSprintSpeed (where CorrectedSprintSpeed came from the serialized Response Data). 
#### Debugging Tip
Make a debug function called ForceDesync() which runs on the client and purposefully mixes up a movement safe variable (for example randomly changing movement mode), then set simulation settings to >100ms of lag but 0 packet loss, and test the function. What you are looking for is the client to get a big desync, but then instantly stabilize. A good idea would also be to print “CORRECTION” when the client gets a correction, to ensure that you only get one singular correction after triggering the desync. If your code isn’t movement safe or you aren’t serializing all the required Prediction / Response data, you will get multiple corrections, or possibly even an irreconcilable state (meaning the client keeps correcting forever). To debug these issues, isolate which variable causes the desync, what values cause it, etc. Bonus, you can parameterize the ForceDesync function to take in a string called PropertyName and then only mix up the variable corresponding to PropertyName (eg ForceDesync(“MovementMode”)), or you could make presets like ForceDesync(“Default”) which would mix location, rotation, mode. 
### Network Conditions
So the biggest thing I wanted to know before I got Astro on real servers was:

>What are real network conditions like?

That means packet loss, lag, condition spikes, how this scales with player count, distance from server, everything! 
The reality is bittersweet. I don’t have exact numbers right now, since it will vary greatly depending on what server solution you choose, but I do have a general ballpark estimate which you can use in the editor to test conditions. First, if you don’t know already there is a specific feature of the UE Editor which allows you to simulate network conditions (if you didn’t know about this, you're welcome).

`Editor Preferences > Play > Multiplayer Options > Network Emulation`

![[Average Net Emulation Settings.png|550]]
Packet Lag = Minimum / Maximum Latency

These are the settings I generally test at, but more explanation is needed.

#### Packet Lag
We already know about this as just ping, and most people have a good idea of realistic pings. On AWS I was able to get about 30-40ms ping when the server was in my region (US-East), and I had people connected from all over the globe (literally Australia), and they reported max of about 200 or peaking at 300ms of ping. (Measured using unreal engine’s PlayerState::GetPingMilliseconds)
#### Packet Loss
I haven’t measured great numbers on this but my estimates were around 2-3% packet loss when calculated based off of the number of sent vs received unreliable prediction RPCs. 
#### UE Inaccurate Average Conditions
So those numbers might sound bad, but the reality is quite good. Unreal’s Packet Lag is not accurate. When you use “average” network emulation conditions it is simulating with “1%” packet loss. In reality, this is more like 7-10% packet loss from what I have observed from calculations. The takeaway is that real servers have way less packet loss than Unreal’s “1%” packet loss.

Based on my estimates a real value for unreal’s “Packet Loss” would be about .4% but unfortunately you can’t enter a float value here, so just know that when simulating with 1% packet loss this is actually significantly worse than real conditions.

Why is it like this? I think that Unreal’s system for emulating packet loss actually takes into account how much data you are sending. When I set incoming and outgoing packet loss to both 1%, I observe calculated outgoing packet loss of 7-10% but only about 2-3% incoming loss (outgoing is predictions, incoming is corrections). This is likely due to the fact that I am sending near 10x the bitrate of predictions than corrections (predictions happen every frame but corrections are seldom). So an additional important thing to keep in mind is that higher a bitrate leads to higher packet loss!
#### Crashes and Bad Conditions
While real network conditions are actually better than you might have expected, there is a catch. I playtested Astro individually and over LAN and stomped many bugs, mostly due to networking, and I also emulated networking conditions. Despite all these efforts, putting my game in the hands of 2000 people resulted in hundreds of new bugs, almost all due to networking, that I didn’t catch.

Why? **Probability**

What I mean by this is that many net desyncs can be so rare and in such a strange order that you might never get them through regular testing, but after enough time and potentially with a certain odd network condition that you might not even be able to reproduce, the game will crash.

But there is hope!
I realized I was able to reproduce lots if not all of the crashes on my own device by emulating Bad network conditions. 

![[Bad Net Emulation Settings.png|650]]

It says here 5% loss in and out, what a joke, my calculations found this is more like 30-40 or sometimes even 50% packet loss. It might seem ridiculous to test on such unrealistic conditions, but its not about testing if the game feels smooth, its about finding more crashes.
If your game crashes ever in any scenario on Bad Conditions, it will happen on real servers too, its just a matter of time!
So the best advice I can give is to frequently test on Bad Conditions anytime you add a new feature and treat all crashes with priority, don’t dismiss them as “not likely to happen in Release”.

#### Player Count
I don’t have much to say here, but in my findings, player count did not noticeably affect conditions. Astro is a match fps, so games are around 10 players max, that said I tested up to 20 players in a lobby at once and the server started having issues but this was mainly a hardware issue (ram and cpu usage was peaking out since I was using some of the worst possible AWS machines). 
*If you are making a MMO, I would definitely suggest researching / testing large player counts!*
### Replication Usage

#### Reliable RPCs
>Don’t use Reliable RPCs.

That’s harsh, but try to never use them, I keyword searched my entire codebase and I only use 3 reliable RPCs, when the player gets eliminated (Client RPC tells owning client they were killed), when a player takes damage (Client RPC tells owning client they were damaged), and when a client predicts an enemy hit (Server RPC tells server that client hit player). However, I am actively phasing out the last RPC in favor of a unreliable system I am working on. Never use Reliable Multicasts, just as a rule of thumb. I only use one multicast (unreliable) to let all players know when another player is eliminated. I also follow it up with a replicated variable, so if the multicast is dropped the OnRep will be triggered later. This is a great solution for events where you want to update clients instantly (like letting them know a player died), but don’t want to use a reliable rpc.

>Note: Originally I didn’t use an unreliable multicast, but only a repvar to tell players when a player died, but people complained that they would be shooting at someone, that player would freeze, and then someone else would get the kill. This is because repvars take a little longer to send, and that time difference usually doesn’t matter, but when a player is focused on shooting an enemy, they will notice even a .1s delay in the kill registering, so repvars just don’t cut it. 

#### Unreliable RPCs
I also keyword searched my codebase for Unreliable and found that I only use them three times, Client Predictions, Server Responses, and the aforementioned Multicast Elimination Event. If you are using the CMC, the unreliable Prediction and Response RPCs are already built in for you in the ACharacter class, so you don’t need to use them. Generally the client should only talk to the server via the CSP pipeline (compressed flags, acceleration vector, additional custom data), everything else should be the server talking to the client, and this should be done with repvars.

#### Networking Cost
So something you might not have considered yet is the cost of running multiplayer servers, and this actually factors into how you design your netcode. I have only used AWS, so I can’t speak to other systems, but I’d highly recommend researching the server solution you plan to use. Some people from my discord hosted Astro continuously for a couple weeks and found that they were getting a rate of about $15 per week. (t2.micro Windows) This is for a single ec2 instance. Prices will increase if you use a better machine and opt into matchmaking services. (But price will decrease if you compile for Linux)

For AWS, networking cost is split into two rates
- Hourly Rate: A fixed cost based on the device (more powerful servers have a higher hourly rate)
- Data Rate: There is a fixed (~ $0.09) cost per GB of data sent from the AWS server

The important takeaway here is the direction of your data matters!

>Server -> Client:   \$\$\$
>Client -> Server:  Free

This means that, where possible, you should try to optimize your Server send bitrate, but you can be a lot more lenient with the client send bitrate. This already fits really nicely with the CMC, since the client is sending frequent predictions, but the server is seldomly sending corrections. (But keep in mind the server still sends rpcs every frame acking the received predictions, but these acks are very small (<100 bits). In my nettick system I leveraged this fact by actually not sending ack responses for every client prediction, but randomly letting some predictions go unacked to decrease server send bitrate and then on the client if a move is acked, then I also ack all moves before it, assuming that the server acked but didn’t send a response for those moves. This seems to work fine without any noticeable simulation degradation.

  
### Takeaways
- **Emulate Net Conditions**
	its a must, emulate on average for normal testing, but expect that this is going to still be significantly worse than real conditions
- **Stress Test**
	Emulate on bad conditions after adding a new feature to stress test crashes
- **Minimize RPCs**
	Minimize usage of all RPCs, but most importantly reliable ones
- **Minimize Bitrate**
	Minimize Server Send bitrate to reduce costs and client bitrate for more net accuracy

### Other Thoughts

#### Server Build
To actually get your game on servers you need to compile your game to a Server build. There are several guides on how to do this. Its not that bad, but a lot of “click this and then that and if you mess up one step you might not get and error but it won’t work in the end” sigh The biggest thing is you have to download and compile the engine from source (and its massive and takes like 2 hours).

I found there were like 1 or 2 bugs that occurred from running the server build on my local machine, that didn’t happen from running the standard packaged build, but generally if the regular build runs fine with no crashes the server should also boot. I’d also highly recommend configuring VS or Rider to launch with different profiles.

If you switch the Target to Server or Game, you can launch Rider directly in the packaged version
You won’t have all these versions until you build from source and compile for server


>[!info] Caution
>In order to run your packaged game in-ide, you must first open the game in the editor and cook your content, or to be safe, fully package the game.

This is really helpful for debugging. What I did is package my game, and then open an instance of the game on my desktop just by clicking the exe, and then I used rider to launch a Server instance of my game. Then I connected my exe instance to the rider server instance, and now I can debug the server right inside rider, so when the server crashes I get to see what failed super easily.
	**Note**: The only reason this works is because I have the option in my game to “connect to an IP” which just uses the “open X.X.X.X” command so I can run the client exe and then connect to 127.0.0.1 (local host), which will connect me to the server.
#### Linux
Linux servers are significantly cheaper (for their hourly rate, bandwidth rate doesn’t change) on AWS so it's a great idea to compile the server build for Linux (it will still work with Windows clients).
#### Cheap Bandwidth
You can pay significantly cheaper than AWS rates if you go with other options, but you're getting cheap bandwidth. I haven’t tested other solutions but from what I’ve read, 3D multiplayer games need high quality bandwidth because of the large bitrate required, so you probably can’t get away with some of the rip digital ocean server and expect valorant level networking.

#### Blueprints
I don’t use blueprints pretty much ever. I only use them for UI and some very slight gamemode scripting, but pretty much 99.9% of the game logic is in c++. This is seriously helpful in release builds, because you can get a log file which is way easier to interpret than a BP error. Log files show the line that the crash occurred, and maybe even some data about it. I’ve gotten some BP crashes that barely tell you anything. Also in my opinion it's too hard to maintain BP at scale, and just forget using BP for netcode. This is maybe a harsh take but if I had to guess, Valorant, Fortnite, and lots of other big titles in UE put the bulk of their game in C++ and only surface some light scripting in BP, like it was meant for.

#### Checks
A check is a macro in C++ which makes sure a condition is true.
```cpp
check(IsNetMode(NM_Client))
```

I use them aggressively and I think its a really good strategy to do this because you will find errors faster and more to the source. UE has a memory management feature which automatically nulls out a pointer when its object is destroyed, but this can be much to your detriment if things are getting destroyed or not initialized at the proper time. I rarely use conditional logic to “safeguard” a pointer reference, for example:
```cpp
if (WeaponActor) 
{
	WeaponActor->DoSomething();
}
```

If I expect a pointer to not be null, I don’t safeguard it will a conditional check anyway, this means you code will fail silently, and this is especially prevalent in netcode because it will often lead to another crash later down the line, which was originated all because some important logic didn’t run because you conditionally missed it because of extra safeguarding. Likewise in BP, I almost exclusively use purecasts. Don’t use impure casts unless absolutely necessary because you’re just asking for a silent failure of logic to not run. You want crashes, they explain what line, or which node failed. The hardest thing to debug is silent failures of a feature, such as the player not respawning, but no error log, all because your player controller cast failed because you didn’t plug in the correct player controller class, but without the crash, you have no clue where to start looking!
#### Build Privacy
You are making a game with proprietary code, features, logic, assets, etc and it is important that you don’t get your work stolen. First regarding assets, use Unreal’s encryption (Project Settings > Encryption) to sign and encrypt your “pak” files which contain all your assets. Apparently this can be broken pretty easily though. Even AAA games like Valorant and Fortnite have already had all their assets “cracked” so you will have to accept this as a risk from using UE. Code is a different story however. If you are going to release a public build of your game, make sure to use Shipping and check the For Distribution checkbox. This will ensure that no code remains in your build, and everything is compiled into the exe which can’t really be cracked. I think it's okay to use Development, as long as you have For Distribution checked, but I haven’t looked into this as much. Alternatively, if you trust the people you are distributing your build to, you don’t have to do any of this.

It’s never too early to package your game and put it on an aws server. You get a good amount of free credit on AWS, and it will help you determine sooner what features are / aren’t possible under network conditions!