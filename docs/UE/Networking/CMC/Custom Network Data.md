# Custom Network Data
#### Overview
This document will cover how to use and send custom network data between the server and client through the Character Movement Component, beyond just custom flags. This could include sending additional compressed flags, additional input vectors, etc. Note: You must be using Packed Movement RPCs for this to work! (This is determined by console variable NetUsePackedMovementRPCs)

Resources: [Discord Community](https://discord.gg/uQjhcJSsRG)
### Defining State
So I’m going to use the terms input state, intermediate state, and output state a lot. Input state means anything value that is used to calculate a move and is purely Client Authoritative. Currently in the engine there are only three input states: Acceleration, Control Rotation, and CompressedFlags. These are driven directly by player input like key presses or mouse movements, and the server has no ability to change them. Output state is any data that is the end result of performing a move. For 99% of projects, this is mainly just going to be capsule location and movement mode. These are the “end values” that affect what the player sees in the world. Intermediate state is in the middle of input and output because it is modified by input but also modifies output state. The perfect example of this is Velocity. Velocity is affected by Acceleration, but it in turn modifies location. It is important to note that all types of state should be stored in the SavedMove, but only input state should be replayed in the PrepMoveFor function.

Other important notes:
- I’m going to refer to client and server a lot, when I say client, I specifically mean owning client.
- Net Correction and Client Adjustment are synonymous
- Unreal Engine internally refers to custom CMC network data as PackedMovementRPCs because I think this system was originally intended to just use more compressed flags as triggers instead of RPCs (which is a very good use, but not the only use)

### What is Custom Network Data
Using custom network data in the CMC means overriding the actual bits that are sent between client and server to inject (or remove) data. This is done through the CMC’s NetworkMoveData system, which will be explained in detail. There are two different types of network data that you can override: CharacterNetworkMoveData and CharacterMoveResponseData. The NetworkMoveData is sent from the client to server every single frame and contains input state, output state (Location, Movement Mode), and the TimeStamp of the performed move. The server will then perform the move with the given input state, and check if its computed output state is close enough to the client’s sent output state. If the client sends an output state that differs too much from the server’s computed output state, it sends a net correction. The MoveResponseData is sent from server to client and is in one to one correspondence with each received move. This means for every NetworkModeData sent by the client, the server will send back a corresponding MoveResponse. The MoveResponse can be one of two things, depending on if the server is net correcting the client or not. If the server agrees with the client’s output state for a given move, it will simply acknowledge the move and the MoveResponse will only be a couple of bits (pretty much just a bool saying bAckGoodMove). If, however, a desync is detected, the server sends a MoveResponse with all intermediate and output state (location, velocity, rotation, movement mode, etc.). When the client receives the move response data, it adjusts to the server’s values and re-simulates the pending moves.
### Why would you want to use Custom Network Data
There are three main reasons why you might need to use custom network data: Sending extra input state, sending extra output state, achieving frame zero desync correction, or removing unused state.
#### Sending Extra Input State
Currently the CMC sends extremely minimal input state, and you only really get to play around with CompressedFlags, so sometimes you might need more data. Maybe you want an extra 8 compressed flags, maybe you need to send an extra joystick input or some other type of input data. Note of caution here that if you think you need to send a lot of input state (like vectors or transforms), you are probably thinking about it wrong. The CharacterNetworkMoveData is sent from the client to the server every single frame, so if you want to send data for a one-off event like Entering a Slide where you include a bunch of floats, vectors, etc, do this in a reliable RPC.
#### Sending Extra Output State
As previously mentioned, the client will send input state, but also a minimal version of the output state of a move. The goal of sending a little bit of output state is to make sure the client is in sync with the server. Currently, the client pretty much just sends location and movement mode, but sometimes this is not enough. If you had a custom state that was critical to performing moves, you might want to also send this too. The goal here is that the server can use this as an additional way to detect if the client is out of sync, and issue a net correction faster. If the server issues the net correction quick enough, the client will be smoothly corrected, meaning the player might not even notice. Note that the actual line between intermediate state and output state is a bit arbitrary. You can promote intermediate state to output state if you think it's important enough. The main difference is just that output state is what gets sent from the client every frame, whereas intermediate state is not.
#### Frame Zero Desync Correction
So the two above reasons to use Custom Network Data targeted sending data from the client to the server, but its also important to consider overriding the data being sent from the server to the client. This is only really applicable in the case of a net correction. Frame Zero Desync correction means that, no matter what kind of desync happens, when the client receives an update from the server it will recover back to the server’s state in one ClientAdjustment call. If you have created any additional intermediate or output state whatsoever, then to get frame zero desync correction you need to send custom MoveResponse data. To see why, just imagine you have the smallest of intermediate states: an enum indicating the walk type (slow, normal, sprint). If the client gets a desync, and this state is not in sync with the server, we will get a correction from the server containing info on our new position, rotation, movement mode, etc, but not anything about the walk type. Then we will re-simulate these moves, but with the incorrect walk type again. This will cause us to have more net corrections! If we achieve frame zero desync correction, by sending all intermediate and output states to the client, then even every state in the CMC could get horribly out of sync by thousands of units, but they would all be brought back into sync in just one net correction!
#### Removing Unused State
So this is more of an optimization, but if you look into the MoveData being sent, you might notice some values your game doesn’t utilize, so to save some extra bandwidth, you can simply not send these values. Some common values you might not use are: MovementBase info, ControlRotation Roll (most games only need Pitch and Yaw), or certain compressed flags like crouch or Engine Reserved might not be in your game.
>Note: It will make more sense to read Client Move Data before Server Response Data, even if you only want to override the implementation of Server Response Data.
### Client Move Data (Client -> Server)
#### Default CMC Pipeline
First I’ll go over the process of sending custom data from client to server. Let’s go over this process first so we’re on the same page. During the client CMC’s TickComponent, the function ReplicateMoveToServer is called. This function does three things in order:

- Creates and Sets a new SavedMove
- Performs the move ( PerformMovement() )
- Calls  CallServerMovePacked()

However, the last step, CallServerMovePacked() is not that simple. If you remember from the tutorials, when we added variables to the SavedMove struct, we had to update the function CanCombineWith(). This function checks to see if it can combine itself (the current SavedMove), with another SavedMove. The reason is that the client doesn’t actually call CallServerMovePacked() every frame, but every other! The first time, it actually just saves the SavedMove as the PendingMove, and then in the next frame it checks to see if it can combine the PendingMove with the New Move. If it can, we can just send one move instead of two, which is obviously way better for bandwidth. Note that you should only combine moves if they basically have all the same input state. The best example of this is just if the player is running in a straight line. If the acceleration is pretty much the same, velocity doesn’t change much, and only location is changing, we can combine the moves and send them as one to the server, we use the output state of the New Move since it is most recent, but we add the delta times of the Pending Move and New Move together, so that its effectively one big move. If, however, we were wrong and the moves are different (say the player pressed jump on New Move, but wasn’t pressing jump in Pending Move), then we need to send both moves, separately, but on the same frame. You can see how most of the time, this system works really well because in a 60fps game, the player is probably holding down the same arrow keys for the majority of frames and only changes button states every couple seconds. Now there is actually one more move that we also send and this is the Old Move. The Old Move is the oldest unacknowledged important SavedMove. So remember how the server also sends back an ack for every move, but sometimes moves get dropped or arrive late or sometimes responses from the server get dropped. To try and combat this, the client will send the oldest move that isn’t acked, assuming that it was probably dropped the first time. So in total on a given frame there could be up to three different moves being sent: New Move, Pending Move, and Old Move, but there could also be no moves sent if there aren’t any unacked SavedMoves and we are holding off on sending by putting it into PendingMove for the next frame.

So why did we need to learn all of that? Well now let's see what actually happens inside of CallServerPackedMove(). The official signature of this function is in good agreement with what we learned

```cpp
virtual void CallServerMovePacked(
	const FSavedMove_Character* NewMove, 
	const FSavedMove_Character* PendingMove, 
	const FSavedMove_Character* OldMove
);
```

I encourage everyone to go into the source code and read this function, as it is 49 lines long, but there are only a few lines that we actually care about. First, we get a reference to something called a 

`FCharacterNetworkMoveDataContainer`, and fill it with client data:

```cpp
// Get storage container we'll be using and fill it with movement data
FCharacterNetworkMoveDataContainer& MoveDataContainer = GetNetworkMoveDataContainer();
MoveDataContainer.ClientFillNetworkMoveData(NewMove, PendingMove, OldMove);
```


So the `FCharacterNetworkMoveDataContainer`, or MoveContainer for short, is a struct which holds three network moves. These network moves are not SavedMoves, but another new struct called FCharacterNetworkMoveData, or NetworkMove for short. In ClientFillNetworkMoveData, we pass along the saved move to the three NetworkMoves to fill them with data, these are the actual network “move” structs.

```cpp
NewMoveData->ClientFillNetworkMoveData(*ClientNewMove, FCharacterNetworkMoveData::ENetworkMoveType::NewMove);
bDisableCombinedScopedMove |= ClientNewMove->bForceNoCombine;
```
(this logic is repeated for PendingMoveData and OldMoveData)

In ClientFillNetworkMoveData, we take only the necessary information out of the SavedMove and fill the NetworkMove.

```cpp
TimeStamp = ClientMove.TimeStamp;
Acceleration = ClientMove.Acceleration;
ControlRotation = ClientMove.ControlRotation;
CompressedMoveFlags = ClientMove.GetCompressedFlags();
MovementMode = ClientMove.EndPackedMovementMode;
```

So to summarize, we are calling `ClientFillNetworkMoveData()` on the MoveContainer, which is just a struct that holds three NetworkMoves, and it calls `ClientFillNetworkMoveData()` on each move with its corresponding SavedMove. Now we have (up to) three populated NetworkMoves in our MoveContainer. We’re still just passing around data, however, the next step is to serialize it. If you're not familiar, serializing data just means converting it to raw bits. We do this for two reasons. 

1. Serializing our NetworkMove means we can optimize it and only send as few bits as possible
2. We can serialize a variable number of bits, meaning you can add more bits for your custom data

>This was not possible until recent versions of Unreal Engine (right before UE5).

After we have filled the MoveContainer, we next Serialize Container
```cpp
MoveDataContainer.Serialize(*this, ServerMoveBitWriter, ServerMoveBitWriter.PackageMap);
```

So let's jump into the Serialize function.

```cpp
// Base move always serialized.
if (!NetMoveData->Serialize(CharacterMovement, Ar, PackageMap, FCharacterNetworkMoveData::ENetworkMoveType::NewMove)) 
{
	return false;
}
```

Unsurprisingly we just call the Serialize function on our three NetworkMoveData’s (screenshot only shows call to one Move.Serialize call). So how do we serialize? With an archive. An archive is a bitstream that allows you to “push” bits onto or off of it. If you haven’t seen a bitstream, don’t worry since you don’t really need to know how it works, just how to use it. In our case, the archive is ServerMoveBitWriter. So let’s jump into the MoveData Serialize function.

```cpp
NetworkMoveType = MoveType;  
  
bool bLocalSuccess = true;  
const bool bIsSaving = Ar.IsSaving();  
  
Ar << TimeStamp;  
  
// TODO: better packing with single bit per component indicating zero/non-zero  
Acceleration.NetSerialize(Ar, PackageMap, bLocalSuccess);  
  
Location.NetSerialize(Ar, PackageMap, bLocalSuccess);  
  
// ControlRotation : FRotator handles each component zero/non-zero test; it uses a single signal bit for zero/non-zero, and uses 16 bits per component if non-zero.  
ControlRotation.NetSerialize(Ar, PackageMap, bLocalSuccess);  
  
SerializeOptionalValue<uint8>(bIsSaving, Ar, CompressedMoveFlags, 0);  
  
    SerializeOptionalValue<UPrimitiveComponent*>(bIsSaving, Ar, MovementBase, nullptr);  
    SerializeOptionalValue<FName>(bIsSaving, Ar, MovementBaseBoneName, NAME_None);  
    SerializeOptionalValue<uint8>(bIsSaving, Ar, MovementMode, MOVE_Walking);  
  
return !Ar.IsError();
```
*entire function present*

Here you can see in only a few lines all of what the CMC is sending from client to server. Pay attention to this carefully since this is where we actually control what data is being sent to the server. The Ar is the FArchive object, and we can add data to it in several showcased ways. At the top, the TimeStamp is pushed onto the archive using the << operator, this just means that the entire 4 byte float is added to the bits. Then we see some helper functions on the Acceleration, Location, and ControlRotation variables which are adding themselves to the Archive under the hood. Note that the Location variable is a FVector_NetQuantized100 and the Acceleration variable is a FVector_NetQuantized10, ControlRotation is just an FRotator. We also have another helper function: SerializeOptionalValue(), which allows us to optionally serialize a value based on a bool.

There is one more very helpful way to add bits to the bitstream:
```cpp
Ar.SerializeBits(&CompressedMoveFlags, 4); // 4
```
This allows you to enter the length or number of bits to serialize, which I use in Astro because I can send less bits than the actual variable type (if I don’t use all the bits). Every bit counts.
So now we have filled three NetworkMoves and serialized all of them into a single FArchive. The last step is just to send them to the server.

```cpp
// Send bits to server!
ServerMovePacked_ClientSend(PackedBits);
```
*Last part of `CallServerMovePacked()`*

This function is a wrapper function for the RPC on the Character class 
`CharacterOwner->ServerMovePacked(PackedBits);`
Which, once executed on the server, calls `ServerMovePacked_ServerReceive()`, back in the CMC. If your wondering why we go from 
`CMC -> Character -> *Over Network* -> Character -> CMC`, 
this is because the CMC is actually by default not replicated. All CMC related rep vars and RPCs are actually on the Character. This is because replicating a component adds additional network overhead, so UE is saving you a bit of bandwidth by doing this.

So now we are on the server and we just received a client move. The first thing we do is get the MoveContainer. Don’t forget that this system we just made is on the client and the server, so the server also has a valid MoveContainer, but it is currently empty.
```cpp
FCharacterNetworkMoveDataContainer& MoveDataContainer = GetNetworkMoveDataContainer();
```
Now we have a MoveContainer and some bits from the client, so we need to deserialize these bits into data. To do this we call the Container’s Serialize function.
```cpp
MoveDataContainer.Serialize(*this, ServerMoveBitWriter, ServerMoveBitWriter.PackageMap);
```

Okay you might be confused at first. Didn’t we use this to serialize data into bits, but now we are trying to deserialize bits into data? Well the FArchive does a little trick where it actually has a “direction”. If it is saving, it will write variables into bits, if it is loading, it will write the bits into the variables. Basically, all you need to know is that you don’t need to write a different deserialization function since the archive will automatically populate your custom MoveContainer and its custom NetworkMoveData objects. This is really helpful because it eliminates the possibility of you misinterpreting bits which could be really annoying to debug.

So now we are on the server and we have a MoveContainer filled with (up to) three moves from the client which are populated with all the same values on the client. We then call:

```cpp
ServerMove_HandleMoveData(MoveDataContainer);
```

This is mainly a wrapper function which calls:
```cpp
ServerMove_PerformMovement(*OldMove);
```

For each of OldMove, PendingMove, and NewMove. Inside of ServerMove_PerformMovement, we call MoveAutonomous, which performs the move, and then ServerMoveHandleClientError, which calls ServerCheckClientError to see if the location and movement mode that the client sent are the same (or close enough) to how the resulting values after the server performed the move in MoveAutonomous. If everything is good, we just ack the move, if there is a desync, we queue up a PendingAdjustment (this is explored more in the Server Response section). 

This is the end of the pipeline, after the server has performed and acked a move, that move has officially ended and we repeat the whole process over for the next frame.To recap, we started on the client on TickComponent, we performed the move, recorded the state values in our SavedMove, checked if / how many moves to send to the server, filled those SavedMoves into our MoveContainer, which filled into three NetworkMoveData objects, and then serialized this information and sent it to the server. Once on the server we deserialized the bits back into NeworkMoveData objects with readable variables with some clever logic by reusing the serialize function, then ServerMove_PerformMovement was called for each of the received moves from the client, which performed the movement on the server (with the client’s input state), and then checked if the client’s reported output state was in agreement with the server’s computation, issuing a correction if the client was out of sync.

#### Adding Custom Data
So that was an overview of how move data is sent from client to server, but how do we add our own custom data to this? We need to override the two structs: FCharacterNetworkMoveDataContainer and FCharacterNetworkMoveData as well as several CMC functions.
When I say “override” a struct, this means creating a new struct that is derived from the default one. First, create a custom NetworkMove struct:
```cpp
struct FZippyNetworkMoveData : public FCharacterNetworkMoveData 
{
	uint8 MoreCompressedFlags; // your custom data
	FZippyNetworkMoveData() : FCharacterNetworkMoveData(), MoreCompressedFlags(0) {}
	virtual bool Serialize(...) override;
	virtual void ClientFillNetworkMoveData(...) override;
};
```

Fill it with variables corresponding to the additional data you want to send over the network (like another uint8 MoreCompressedFlags). Also override two methods from this struct: ClientFillNetworkMoveData and Serialize. In the ClientFillNetworkMoveData function, call the Super function (note that the Super keyword is not valid here, you have to call FCharacterNetworkMoveData::ClientFillNetworkMoveData) and then cast the SavedMove to your custom SavedMove class and copy the additional data that you want to send over the network from the custom saved move into the member variables of your new custom NetworkMove struct. In the Serialize method, you actually shouldn’t call the Super function, but go into the source code and copy and paste the default implementation of FCharacterNetworkMoveData::Serialize into your new Serialize function. Then serialize any additional data using the above explained methods, and remove any lines that serialize data you don’t need. This way we only send the minimum number of bits and keep our network usage optimal.

For the MoveContainer, we make FZippyNetworkMoveDataContainer, which is derived from the original struct FCharacterNetworkMoveDataContainer in much the same way as before. This struct is much easier because we actually don’t even need to override any methods, just make a member variable that is an array of three custom NetworkMoveData structs. Then in the constructor, don’t call the Super constructor but set the three base class NetworkMoveData pointer variables to references from your custom NetworkMoveData array. Note here I am making use of an inline constructor. Also, you don’t have to make an array of NetworkMoveDatas, you could make three distinct variables, all that matters is you have three valid references to custom NetworkMoveData objects which will be guaranteed to exist for the lifetime that the container exists.

```cpp
struct FZippyNetworkMoveDataContainer: public FCharacterNetworkMoveDataContainer
{
	FZippyNetworkMoveData CustomMoves[3];
	FZippyNetworkMoveDataContainer() {
		NewMoveData = &CustomMoves[0];
		PendingMoveData = &CustomMoves[1];
		OldMoveData = &CustomMoves[2];
	}
};
```
So we’ve linked usage of our custom NetworkMoveData struct into our custom MoveContainer, but how do we link the custom MoveContainer to our CMC? First, make a private variable in your custom CMC that holds a custom MoveContainer:
```cpp
private:
	FZippyNetworkMoveDataContainer ZippyNetworkMoveDataContainer;
```
and then in the your custom CMC constructor, set the CMC’s NetworkMoveDataContainer to your custom container variable.
```cpp
SetNetworkMoveDataContainer(ZippyNetworkMoveDataContainer);
```
With this in place, we will use our custom MoveContainer to fill and serialize data meaning our custom data will be sent across the network! This is great but just sending extra data to the server doesn’t mean anything if we don’t use it in some way. 

Remember the pipeline on the server is:
1. Deserialize
2. ServerMove_HandleMoveData
3. ServerMove_PerformMovement
4. MoveAutonomous
5. ServerMoveHandleClientError
6. ServerCheckClientError

We already wrote the deserialization function (its just the serialize function), and steps 2 and 3 just handle generic move logic that will probably be the same for all projects. Its MoveAutonomous and ServerCheckClientError that we are interested in overriding, but this is where your intentions with custom movement data come into play.

If you are adding custom output state that you will use to detect more net desyncs, then you want to override ServerCheckClientError. This method returns true if there is an error and false if there isn’t, so simply override this function and add a check with your custom data.
```cpp
ServerCheckClientError(...)
{
	return Super(...) || ServerCheckCustomDataError();
}
```

If you are adding custom input state, you need to process this data into your server’s CMC to perform the move correctly. (We’ll use MoreCompressedFlags example) To do this just add a line before the Super implementation to update the CMC values from the received values:
```cpp
MoveAutonomous(...) 
{
	UpdateFromMoreCompressedFlags();
	Super(...);
}
```
Notice I defined two vague functions for each use case (ServerCheckCustomDataError and UpdateFromMoreCompressedFlags). So how do you go about correctly reading the custom net data values? The key is that the CMC will automatically store the currently simulating NetworkMoveData in a private member variable called CurrentNetworkMoveData. This is helpful because it allows us to bypass the fact that these functions we override (ServerCheckClientError or MoveAutonomous) have fixed signatures which don’t include any of our custom data. Just simply access the current NetworkMoveData through the public method GetCurrentNetworkMoveData() and then cast it to your custom NetworkMoveData class:
```cpp
FZippyNetworkMoveData* CustomMoveData = static_cast<FZippyNetworkMoveData*>(GetCurrentNetworkMoveData());
```
Hopefully this gives you a good understanding of how to move custom networked data from client to server and process that data accordingly to either allow more complex moves or enhance desync detections.
### Server Response Data (Server -> Client)
#### Default CMC Pipeline
Server Response is the method by which the server tells the client that its move was good or bad, and if bad, gives details on the correct move state (net correction). While you might need to add some custom info about good moves, most likely you want to send more data on a net correction to fix the client’s state. Regardless, both good and bad move responses follow the same pipeline, which we are going to get into.

Before we jump in, I want to briefly recap how a net correction is actually applied to the client (if you haven’t watched the [architecture](https://youtu.be/dOkuIvKCvpg) video, definitely do that first). So when the client receives a net correction, it gets all of the state for a move (location, velocity, rotation, etc), and the TimeStamp that the move was performed on. The client is in the future, however, because of Client-Side Prediction, so this net correction is 2 x Ping milliseconds old, meaning if we just use the server’s position, the character will be back snapped back into the “past” which is very jarring for the player. To fix this we save the move data for every frame (input state: acceleration, compressed flags, intermediate state: velocity, out state: location, etc). These are called pending moves, since we performed them on the client (in the future), but the server is yet to have the final say on the output state of the character. Then once the server performs the move and sends a response, it will acknowledge (ack) the move, and the client will remove the corresponding move from the SavedMoves array, since it is no longer a “pending” move. When the client gets net corrected, it will find the corresponding SavedMove and correct it with the new server state. Then it will replay all the still pending SavedMoves to move the client back into the future. Here are some useful links to visualize this: [article](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html), [interactive demo](https://gabrielgambetta.com/client-side-prediction-live-demo.html).

If you try to go searching through the engine for where the server actually sends a MoveResponse, you might have a difficult time, because it is not obvious at all. First and foremost, it doesn’t send an adjustment immediately after processing a client move. When the server detects a desync at the end of the ServerMove_PerformMove function (as covered in previous section), it queues a PendingAdjustment in the ServerData object. This PendingAdjustment is of the type: FClientAdjustment and contains all relevant state data to correct the client (location, velocity, rotation, etc).

We actually serialize and send this information through a completely different process that originates from the NetDriver. The NetDriver is an object responsible for managing the net connection and you don’t really need to know how it works, but on its TickFlush function, it will loop over all player controllers and call SendClientAdjustment which calls the function SendClientAdjustment on the CMC which calls the function  SendServerMoveResponse (if using packed rpcs). You don’t need to worry too much about these functions, but importantly, TickFlush is a tick function so it’s called every frame, and TickFlush happens after all the ticking groups so its at the very end of a frame. Also, this terminology is a bit misleading by epic: SendClientAdjustment doesn’t always send a net correction, it should really be called SendClientResponse or SendMoveResponse, since it can ack a good move or send a net correction.

So from the UNetDriver::TickFlush, we now made it to UCharacterMovementComponent::ServerSendMoveResponse,which is where the important logic occurs. It's important to understand the Client Move Data pipeline because Server Response Data will follow the exact same process.
```cpp
// Get storage container we'll be using and fill it with movement data  
FCharacterMoveResponseDataContainer& ResponseDataContainer = GetMoveResponseDataContainer();  
ResponseDataContainer.ServerFillResponseData(*this, PendingAdjustment);
``` 

First we get our `MoveResponseDataContainer`, and fill it with a PendingAdjustment . This will move all the values from the PendingAdjustment into the ResponseDataContainer. It's important to do this because the PendingAdjustment is of type `FClientAdjustment` which isn’t serializable.
```cpp
ResponseDataContainer.Serialize(*this, MoveResponseBitWriter, MoveResponseBitWriter.PackageMap)
```
Then, unsurprisingly, we call the Serialize function to move all the data into the `FArchive MoveResponseBitWriter`.
```cpp
// Send bits to client!  
MoveResponsePacked_ServerSend(PackedBits);
``` 
Then we just call the wrapper function MoveResponsePacked_ServerSend which calls the Character function ClientMoveResponsePacked which is an unreliable RPC. 

So the process from queued PendingAdjustment -> TickFlush -> MoveResponsePack_ServerSend, was relatively straightforward, but that’s just sending a move response from the server. Now we need to look at how this move is received and applied by the client. Also, remember that this move response can just be a good move ack or a net correction.

The client first receives this RPC in the character and then immediately calls the corresponding CMC function MoveResponsePacked_ClientReceive. Similar to the Client Move Data pipeline, we get a reference to the FCharacterMoveResponseDataContainer (on the client now), and use it to deserialize the bits from the server into variables in the ResponseDataContainer (using the clever double use Serialize function). 

```
ClientHandleMoveResponse(ResponseDataContainer);
```
Lastly, we call ClientHandleMoveResponse with the filled data container. This function is a gateway to several functions that respond to different types of MoveResponses. If the move is good, we call the ClientAckGoodMove function:
```cpp
if (MoveResponse.IsGoodMove())  
{  
    ClientAckGoodMove_Implementation(MoveResponse.ClientAdjustment.TimeStamp);  
}
```

If the move is *not* good we will call one of the three:
- `ClientAdjustRootMotionSourcePosition_Implementation`
- `ClientAdjustRootMotionPosition_Implementation`
- `ClientAdjustPosition_Implementation`

Unless you need some special behavior for correcting RootMotion, I’m going to safely ignore the root motion corrections. Let jump into the most common type of correction: 
```cpp
ClientAdjustPosition_Implementation()
```

This function takes in a ton of parameters
```cpp
void UCharacterMovementComponent::ClientAdjustPosition_Implementation  
    (  
    float TimeStamp,  
    FVector NewLocation,  
    FVector NewVelocity,  
    UPrimitiveComponent* NewBase,  
    FName NewBaseBoneName,  
    bool bHasBase,  
    bool bBaseRelativePosition,  
    uint8 ServerMovementMode,  
    TOptional<FRotator> OptionalRotation /* = TOptional<FRotator>()*/  
    );
```
These obviously correspond to the net state that the server has just sent.

So inside this function, we first ack the move. You may be surprised because normally we only ack good moves, but since we just received the exact state of the move from the server, we don’t need to worry about ever replaying this move again, which is all that acking really means.

```cpp
ClientData->AckMove(MoveIndex, *this);
```
Then, and this might also be surprising, we simply set all the movement data equal to the server state and return. You might think that we will replay the pending moves, but we actually defer that to the next tick.

```cpp
UpdateComponent->SetWorldLocation(WorldShiftedNewLocation);
Velocity = NewVelocity;
ApplyNetworkMovementMode(ServerMovementMode);
```
Before we finish, however, we set one very important variable:
```
ClientData->bUpdatePosition = true;
```
This will tell the CMC that we have a PositionUpdate, which is the internal name for a net correction. This essentially “queues” the SavedMove replay process. Then in the next frame on TickComponent, we check if we had a position update:
```cpp
FNetworkPredictionData_Client_Character* ClientData = GetPredictionData_Client_Character();  
if (ClientData && ClientData->bUpdatePosition)  
{  
    ClientUpdatePositionAfterServerUpdate();  
}
```
So `ClientUpdatePositionAfterServerUpdate()` is the function for replaying saved moves after a net correction has been received.

Inside ClientUpdatePositionAfterServerUpdate, we loop through all the SavedMoves and perform them using MoveAutonomous, and then set ClientData->bUpdatePosition back to false, and proceed on to perform the move for the current tick.

In summary, the server detects a desync from ServerCheckClientError returning true, and accordingly fills a FClientAdjustment. Then on the next TickFlush (which will be later in the same frame), the SendClientAdjustment function gets called, and fills and serializes a MoveResponseDataContainer with the current server state. Then we send this to the client. When the client receives the update, it sets all its state directly to the server state (position, velocity, etc), and then flips the flag bUpdatePosition. In the next tick, we check for a position update and call ClientUpdatePositionAfterServerUpdate to replay all the SavedMoves, and then continue on with the current tick. This is the full net correction pipeline.
#### Adding Custom Data
It is recommended to have first read the Client Move Data section, since this follows pretty much the exact same workflow (so I won’t be as specific). The key is creating a custom struct that is derived from the default:  `FCharacterMoveResponseDataContainer`. Unlike the Client Move Data, we don’t need two different structs, just a single container. Create this in your CMC header file with variables corresponding to the extra state data you want to send to the client, and override the two important functions: ServerFillResponseData, and Serialize. All custom variables in your `MoveResponseDataContainer` should correspond to custom state data member variables in your CMC.
```cpp
struct FZippyMoveResponseDataContainer : public FCharacterMoveResponseDataContainer
{
	int CustomState; // your custom data
	FZippyMoveResponseDataContainer();
	virtual void ServerFillResponseData(...) override;
	virtual bool Serialize(...) override;
}
```

Then make a private member variable in your CMC for your new struct:
```cpp
private:
	FZippyMoveResponseDataContainer ZippyMoveResponseDataContainer;
```  
To get the CMC to use our new custom move response container, we need to use the CMCs move response setter function:
```cpp
SetMoveResponseDataContainer(ZippyMoveResponseDataContainer);
```
This can be done anywhere but best to do it in the CMC constructor.

Once you’ve added custom variables to your MoveResponseContainer, implement the ServerFillResponseData and Serialize function in very much the same way as the Client Move Data process. By default you’re given a reference to the UCharacterMovementComponent, so just cast this to your custom CMC to get your custom state data. For Serializing, just follow the same process as the Client Mode Data examples.

Now we are done setting up the server to send our custom data, and all we need to do is enable the client to use this data to set their own state. The function we need to override is:
```cpp
virtual void ClientAdjustPosition_Implementation(...) override;
```
Remember this function doesn’t do any replaying of SavedMoves, it just sets its state equal to the resulting state it got from the server. First call the Super function to let the default state data (location, velocity, etc), get set to the server state, then just set your custom CMC variables equal to the values you sent from the server MoveResponseDataContainer. You can access the values by simply using the private member ZippyMoveResponseDataContainer that you created, since this is the same object which will be used for all net corrections.

```cpp
Safe_CustomState = ZippyMoveResponseDataContainer.CustomState;
```

