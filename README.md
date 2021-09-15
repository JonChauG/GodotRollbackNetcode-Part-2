# GodotRollbackNetcode-Part-2

Repositories:

https://github.com/JonChauG/GodotRollbackNetcode-Part-1

https://github.com/JonChauG/GodotRollbackNetcode-Part-2

https://github.com/JonChauG/GodotRollbackNetcode-Part-3-FINAL

---

Tutorial Videos:

Part 1: Base Game and Saving Game States - https://www.youtube.com/watch?v=AOct7C422z8

Part 2: Delay-Based Netcode - https://www.youtube.com/watch?v=X55-gfqhQ_E

Part 3 (Final): Rollback Netcode - https://www.youtube.com/watch?v=sg1Q_71cjd8

---

Part 2 Video Transcript:

### INTRO

  So now, we’re going to add delay-based netcode to the game we previously created. Here is the final product of this video: A player controls the bottom green square in one game that corresponds to the top yellow square in the other game. In a good connection, both games are synchronized and gameplay is smooth. During a poor connection, the games may delay until the necessary communication between them is fulfilled so that they remain synchronized.

### TREE

  Compared to our game scene in the last video, we now add another Player object to use as the networked player (or NetPlayer), which runs its own NetPlayer script. From now on to avoid semantic confusion, I will refer to the NetPlayer object in the scene as the “NetPlayer,” and the game instance that we intend to network with as the “networked game”.

  Remember that InputControl is set up to control all player objects that are its children, so like the Local Player, the NetPlayer is also a direct child of InputControl. When we play the game, the LocalPlayer is at the bottom, and the NetPlayer is at the top (and will turn yellow).
  
  I have also added a Label here, the StatusLabel, that will provide the current status of the game during networking.

### INPUTCONTROL.GD

  In our InputControl script, we have many new variables in the global scope of the script.

  This enumerator and related variables here are used to track the game status during networking through the StatusLabel in the scene. When WAITING, the game is waiting for a connection handshake with another game instance. When PLAYING, the game is communicating with the other game instance (so this would be the networked game). And when the status is END, the game has stopped communication.

  To incorporate networking, we will be using another thread to handle receiving data from the network. If you’re not familiar with multithreading, I would encourage you to do a little research so that you’ll have some understanding of it before we move on. We will use Godot’s PacketPeerUDP object to send and receive packets through the network by UDP.

### PRESENTATION: NETWORKING OVERVIEW
  
  So now, we’ll start diving into how our game will communicate with a networked game instance. To summarize, most of the packets we will send and receive through the network will contain the recorded inputs (or key press check data) of multiple frames. These are the same inputs that we use to control our LocalPlayer. We send our input data packets to the networked game so that it may use them to control its own NetPlayer.

  So let us say we are Game A, and the networked game is Game B.
  
  We first record our inputs to move our own LocalPlayer, and then we send these inputs through the network to Game B.
  
  Game B receives the inputs and uses them to move its NetPlayer.
  
  And at the same moment, Game B is similarly recording its own inputs to move its own LocalPlayer and sending these inputs for Game A to use for its Netplayer.
  
  The expected behavior is for the NetPlayer in Game B to mirror the movement of the LocalPlayer in Game A, and vice versa. 

  In order for our game to process and move both of its Player objects in a given frame, our game needs the NetPlayer input of that frame from the networked game to know how to operate the NetPlayer. However, what happens when our game arrives on and tries to proceed on a frame that still needs an NetPlayer input from the networked game? Well, our game will not have the information needed for our NetPlayer to properly mirror the networked game’s LocalPlayer. So, our game in this video will not try to proceed and instead wait (or delay) until the needed input arrives.

### PRESENTATION: PACKET ORGANIZATION

  When we send inputs between game instances, we need to decide how to organize this input data in packet form. Packets are made up of bytes, so to send a single frame’s input, we could assign one byte per key so that each byte represents a boolean for whether the key is pressed or not. Because we are only using WASD and Space, we end up with five bytes so far for each individual frame’s recorded input. Then, to indicate which frame this input is intended for, we include another byte that contains the input’s frame number. Remember from the last video that we have a frame number given by a 256 frame cycle that continuously increments from 0 to 255. And 255 is the highest value that can be stored in a byte. So with our current organization, we use up six bytes total for each frame.
  
  However, we can be more efficient. Recall that a byte is eight bits. We can instead assign one bit per key so that we use up five bits of a single byte for the input checks, having some bits left over. So in the end with this organization, each individual frame’s input takes up two bytes in the packet. We can then also include multiple frames’ worth of inputs in a single packet more efficiently.
  
### PRESENTATION: UDP AND MEASURES AGAINST UNRELIABILITY

  Although UDP is unreliable, that is, all packets sent by UDP are not guaranteed to be successfully delivered, we will use UDP instead of the reliable TCP because UDP transmits 
packets more quickly. We will make up for the unreliability of UDP in three ways:

First, when we send packets every frame, we will not only send the inputs recorded in the current frame, we will also send the inputs of past frames in the same packet. This is to increase the likelihood that a game still receives needed inputs for all frames even if some packets are dropped in the network.

Second, every frame, we will send multiple duplicates of this aforementioned packet containing our inputs, further increasing the likelihood that a game gets inputs for all frames even if packets are dropped.

Finally, we will implement an input request system so that even if all packets containing the input for a given frame are not successfully sent, the game that is waiting for them to arrive can transmit a request for resending those inputs.

### INPUTCONTROL.GD

  So, back to the InputControl script. These two boolean arrays here, the input_arrival_array and the input_viable_request_array, and this boolean, input_received, are used for communication between the main game thread and the networking thread.

  The input_received boolean allows our two threads to communicate if a new input has arrived from the networked game instance.
  
   The networking thread will set this to true whenever a new input arrives.

  If the main game thread needs an input that has not arrived yet to proceed on the current frame, it will set the boolean to false and wait until the networking thread sets the boolean to true to check again if the needed input has arrived.

  The input_arrival_array is used to check if a net input for a given frame has arrived, using the array index to represent the frame number in the 256 frame cycle.

  The networking thread sets the boolean for a given frame in the cycle to true when an input arrives for that frame.

  If the main game thread sees that the boolean for the current frame is false, indicating that the input for the frame has not arrived yet, the game will wait until the needed input arrives.

  Because we are using a frame number cycle, the main game thread also sets booleans for old frames in the array to false so that before a complete cycle is made, the game will accept inputs for reused frame numbers as new inputs instead of throwing them away as duplicate inputs.

  Similarly, the input_viable_request_array uses the array index to represent frame numbers and is used to check if a local input for a given frame can still be used to fulfill requests or if it is too old in the cycle to use.

  When the main game thread creates a new local_input by recording key presses for the current frame, the main game thread will set the boolean in the array for the new input’s frame number to true

  When the networking thread handles requests, it will only give requested inputs from frame numbers whose booleans are true in the array. For example, if the networked game is ahead of our game and requests inputs for frames that our game has not recorded yet, the networking thread will not fulfill these requests. As with the input_arrival_array, before a complete frame cycle is made, the main game thread also sets booleans for old frames to false.

  And when we have multiple threads that access the same resources, we enclose these resources in their respective mutexes when accessing them so that only one thread can access a given resource at a time. Otherwise, unintended behavior may occur.

  The dup_send_range gives the range of past inputs to send with the current frame’s input in the same packet. So if the current frame has frame number 100 and the duplicate send range is 5, in addition to the input for frame 100, we send the inputs for frames 99 to 95 inclusive.

  The packet_amount gives the total number of the same input packet to send every frame. As we have here, sending more than one allows for duplicates that help overcome the unreliability of UDP.

  For the classes we made in the last video, we only build upon our Inputs class.

  We add the net_input variable to hold the inputs given by the networked game for our NetPlayer object to use.

  We also include the encoded_local_input, which is the 8-bit or one byte form of the local_input that is sent over the network.
  
  Here, we have the function that is run by the networking thread. In summary, we constantly wait for and process packets from the networked game. In total, the packets that we send and receive can contain four different kinds of data. We distinguish between these kinds of data by including a unique value as the first byte in the packet for each data type. I have organized them as follows:

  If the first byte is 0, then this contains input data for multiple frames from the networked game. The data is organized in the frame_number and encoded input pairs that we went over earlier. The first byte in each pair gives the frame number the input is intended for and the second byte contains the encoded input. We decode the input and write it into the net_input variable of an Inputs class instance in the input_array, the array index used being given by the input’s frame_number. Then, in the input_arrival_array, we note that a new input has arrived for a frame number by setting the frame number’s boolean to true, so the game can now proceed on that frame. If the game had been waiting for a missing input, we now tell the main game thread that it may be ok to proceed by setting the input_received boolean to true because the newly arrived input could be the input the game had been waiting for.
  
  If the first byte is 1, we have received a request for inputs, consisting of two other bytes. These two bytes give the range in frame numbers that inputs are requested for. The end of the range is exclusive, so the game will not send inputs for the frame number given by the third, last byte. We do not want to send non-existent future inputs that we haven’t recorded yet, so we use the input_viable_request_array to track which Inputs in the input_array are OK to send. When we start the game, we initially will send a handshake signal and wait for a reply from another game instance.

If the first byte is 2, we receive this connection handshake and begin communication with the networked game.

If the first byte is 3, it is just a signal indicating that the networked game has chosen to end and disconnect. The game is set to end as well when this signal is received. When we want to send the signal and end our own game, we press Escape.

So, overall, we can visualize each packet data type like this:
- 0 as the first byte indicates an input
- 1 as the first byte indicates an input request
- 2 indicates a game-start handshake
- 3 indicates a game-end signal

In our ready() function, we initialize our newly added arrays and also initialize the networking thread.

  In my current setup, I have our game instance send inputs to itself through localhost so that the NetPlayer in our scene mirrors our own LocalPlayer just for simple testing. We listen from any source on port 240 and send packets to localhost on the same port. (Edit: After recording, I learned that my understanding of “reserved” port numbers was incorrect . Instead of choosing port 240, you should choose a good random port number like a 4-digit value above 1024 that doesn’t collide with another application you have.)
  
  Previously, our physics_process() function merely called handle_input, but the function can now decide to delay the game if it does not have the needed net_input to proceed forward on the current frame. This is where the possible “delay” of our delay-based netcode is.

  Initially when the game proceeds after the networking handshake, we assume that the game is receiving inputs and input_received is true.

  If inputs have already arrived for the current frame, the game can proceed as normal by calling handle_input().
  
  Otherwise, the main game thread will set input_received to false and send a request for a needed input.

  As long as input_received is false, the main game thread will not call handle_input, so it will not get key presses nor operate on the Player objects, causing the game to do nothing but send requests every frame. 

  We also add our simple way of sending a game-end signal to the networked game and ending our own game by pressing Escape.

  In our handle_input() function, we now also encode the local_input as we record our key presses to save in the encoded_local_input variable of the Inputs class instance given by frame number the input is intended for. Each bit in a byte can be seen as representing an increasing power of 2, so we encode the input by adding a respective power of 2 for each pressed key.

  Here, we send multiple duplicates of inputs for the current frame and for past frames.

  Then, we set the boolean for an old frame in the input_arrival_array to false so that the old frame number can be reused in the 256 frame cycle.
  
  In the input_viable_request_array, we set the boolean for the intended frame of the input we just recorded in the current frame to true so that the input can now be sent by request.

  And, for an old frame, we set the boolean in the array to false so that the frame number can be reused.

### PRESENTATION: RESET BOOLEANS FOR OLD FRAMES

  So, why specifically do we choose these array index values as our old frame numbers to be reset to false in the input_arrival_array and input_viable_request_array?

  To demonstrate, here is a Game A on a frame number timeline with its current frame number here. I use red to show that Game A has not yet proceeded on its current frame because it has not yet run the handle_input() function, possibly waiting for a needed net input to proceed. Here, I use blue to show that Game A has proceeded on its current frame, and I show the intended frame of the input that Game A has recorded. Remember that we reset booleans as part of proceeding forward on a frame when we call handle_input(), so we reset our array booleans as well when blue is shown.

  Here are two network-connected game instances: Game A and Game B, with the relative frame number positions that each game is currently on. In this scenario, the games are far apart in terms of frame number. We could say that Game A has not received any inputs from Game B, so Game A cannot proceed on its current frame number. Game A will send requests to Game B for needed inputs, so Game B should still fulfill requests for inputs from at least up to (input_delay)-many frames in the past so that Game A can proceed. We have the input_viable_request_array reset booleans for old frame numbers at that index

  In this scenario, we could say that Game A and Game B are both proceeding on each of their current frames. Game B gives valid new inputs to Game A, so to accept all inputs that Game B sends, Game A should accept inputs intended for at least up to (input_delay + input_delay)-many frames away in the future. But, why do we add one to this value here in our input_arrival_array index that gives the boolean to be reset? Well, new valid inputs could have already arrived to Game A before it resets the boolean to false in the handle_input() function, and Game A in that case would treat these inputs as duplicate inputs to be discarded because the input_arrival_array boolean for that frame is still true. For Game A to accept these inputs at any point during its current frame, we reset the boolean for one frame further ahead than (input_delay + input_delay) in the future.

  A note: when games get very far apart like in the scenarios I have given, both games may continuously delay in a stop-and-go manner because they may always be waiting on inputs from each other. For example, if Game A is waiting, it won’t be sending any inputs to Game B outside of by request. So Game B then has to wait for Game A to get inputs, for Game A to proceed, for Game A to send its own inputs. But while Game B is waiting, it isn’t sending inputs, so A has to wait. And so on and so forth. I don’t resolve this issue in this video series, but solutions include having the game that is ahead delay for an extended time so that the game that is behind can catch up, resulting in the gap or rift between the games to be reduced.

### INPUTCONTROL.GD

  So, back to the InputControl script. Overall, the usage of the functions to operate on all of the children of InputControl have not changed. Whenever we call frame_start_all, input_update_all, and frame_end_all, this time we also operate on the NetPlayer because the NetPlayer is a child of InputControl. We have added the net_input variable to the Inputs class, so the current_input variable we use that contains an Inputs class instance now has the information needed to move the NetPlayer.

  We don’t do anything with our saved states in this video, but in the next video, we’ll combine them with the delay-based netcode we’ve made to form rollback netcode.

### LOCALPLAYER.GD

  In our LocalPlayer script, the only thing we are adding is this new collision detection code. We’re checking if the LocalPlayer object overlaps with any other child of InputControl by checking the intersection of their Rect2 CollisionMasks that we’ve defined. So when the LocalPlayer and NetPlayer overlap with each other, their counter test variables increase. I use the counter variable as a cheap way to test if two network-connected game instances are synchronized by checking if the counter variable value of a LocalPlayer matches the value of its associated NetPlayer in the networked game instance.

### NETPLAYER.GD

  For our NetPlayer script, which is run by the NetPlayer object in the scene, we extend the LocalPlayer script, but we override the \_ready() and input_update() functions.

  We have the NetPlayer use the net_input variable of the Inputs class, and because the net_input consists of the direct key press checks from the networked game, we also change the signs of the movement so that the NetPlayer movement mirrors that of the networked game’s LocalPlayer.

### STATUSCHECK.GD

  This StatusCheck script run by the top, root node of the scene is just to display the current status of the game given by the game variable in InputControl.

### DEMO

  So for a quick test, I’ll run one game instance connected to itself by localhost so that the game is just sending inputs to itself. The NetPlayer will mirror whatever the LocalPlayer does. And because it’s localhost, there won’t be any real network latency, so we’ll basically see what our best case connection looks like.

  Now I’m going to comment out the part of the code in the handle_input() function that sends inputs through the network every frame and see what happens using our localhost setup again. By doing so we can see a worst-case scenario in which all input packets we send every frame have been dropped to test our request system. The game delays every (input_delay)-many frames (so every 5 frames in our case) because it needs to send requests for needed inputs to proceed. 

### TREE
 
  An important note: In our simple scene, we should be careful that the positioning of objects are symmetrical horizontally and vertically relative to the center to eliminate a source of desynchronization between game instances. I’ll move this wall a little bit so that it’s no longer symmetrical to demonstrate.

### EDITED SCENE DEMO
  So here, a LocalPlayer object has stopped at near a wall in the left game, but its corresponding NetPlayer in the right game has not. If the left game continues to move the LocalPlayer towards the wall, the position of the LocalPlayer in the left game and that of the NetPlayer in the right game would not mirror each other correctly, resulting in unintended behavior. So this is to demonstrate that if opposite boundary walls or Player object starting positions were different distances from the center by even 1 unit, we could easily have some desynchronization. 

### DEMO

  So now, I’m running two separate game instances with pre-programmed moves so we can see if they are consistently synchronized in good and bad connections. This is the game in good connection, in which input packets arrive on time for all frames. This is the game in a bad connection, in which input packets often do not arrive on time.

  Again, in the next video, we’ll be turning our delay-based netcode into rollback netcode.
