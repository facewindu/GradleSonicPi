# Gradle music with Sonic Pi

## How to use

1. Install [SonicPi](https://sonic-pi.net/)
1. Copy the following code block in a SonicPi buffer
   ```
   # Gradle music
   use_osc_logging false
   use_debug false
   use_cue_logging false
   use_midi_logging false
   
   use_bpm 90
   use_timing_guarantees true
   use_osc "localhost", 4560
   
   #########################################################
   # Melody of the music
   #########################################################
   chords = [(chord :C, :minor7), (chord :Ab, :major7), (chord :Eb, :major7), (chord :Bb, "7")].ring
   c = chords[0]
   
   #########################################################
   # INIT: when the Gradle builds emits an init event
   #########################################################
   
   define :initStarted do
     live_loop :init do
       cue :metro
       use_synth :fm
       use_octave -2
       3.times do
         play c[0]
         sleep 1
         cue :metro
       end
       play c[2]
       sleep 0.5
       play c[1]
       sleep 0.5
       cue :metro
       c = chords.tick
     end
   end
   
   define :initFinished do |live_loop_name|
     setKillLiveLoopSwitch(live_loop_name)
   end
   
   #########################################################
   # BUILD : when the Gradle builds emits an build event
   #########################################################
   
   define :buildStarted do |live_loop_name, args|
     loop1 = live_loop_name + "_blade"
     loop_1_kill_switch = loop_kill_switch(loop1).to_sym
     live_loop loop1.to_sym do
       sync :metro
       maybeKillLiveLoop(live_loop_name, loop_1_kill_switch)
       use_synth :blade
       play c, amp: 3
       sleep 2
     end
   end
   
   define :buildFinished do |live_loop_name, args|
     setKillLiveLoopSwitch(live_loop_name + "_blade")
   end
   
   #########################################################
   # TASK: : when the Gradle builds emits an task event
   #########################################################
   
   define :taskStarted do |live_loop_name, args|
     loop1 = live_loop_name + "_blade"
     loop_1_kill_switch = loop_kill_switch(loop1).to_sym
     live_loop loop1.to_sym  do
       sync :metro
       maybeKillLiveLoop(live_loop_name, loop_1_kill_switch)
       use_synth :tb303
       r = [0.25, 0.25, 0.5, 1].choose
       play c.choose, attack: 0, release: r
       sleep r
     end
   end
   
   define :taskFinished do |live_loop_name, args|
     setKillLiveLoopSwitch(live_loop_name + "_blade")
   end
   
   #########################################################
   # Helper methods
   #########################################################
   
   # OSC bus live loop
   # Receives OSC events and trigger live_loop creation or stopping based on the passed data
   live_loop :osc_bus do
     use_real_time
     # Wait on an OSC event at that address
     args = sync "/osc*/event/**"
     input_event = parse_sync_address "/osc*/event/**"
     puts "Received event: " + input_event.to_s
     type = input_event[2]
     unique_id = input_event[3]
     action = input_event[4]
     
     loopName = loop_name(type, unique_id)
     case action
     when "started"
       event_started(type, loopName, args)
     when "finished"
       event_finished(type, loopName, args)
     else
       puts "Unknown action: " + action
       stop
     end
     # Ping back that we are ready to receive a new OSC event
     # The Gradle builds waits for this 'ack' OSC message before moving on
     osc_send "localhost", 57110, "/ack"
   end
   
   
   # Creates a thread creating a live_loop which is monitoring when it should stop itself
   define :event_started do |type, live_loop_name, args|
     in_thread(name: (live_loop_name + "_createthread").to_sym) do
       case type
       when "build"
         buildStarted(live_loop_name, args)
       when "task"
         taskStarted(live_loop_name, args)
       else
         puts "Unknown type: " + type
         stop
       end
     end
   end
   
   # Creates a thread that is eventually going to stop a live_loop after some sleep
   define :event_finished do |type, live_loop_name, args|
     in_thread(name: (live_loop_name + "_killthread").to_sym) do
       case type
       when "build"
         buildFinished(live_loop_name, args)
       when "task"
         taskFinished(live_loop_name, args)
       else
         puts "Unknown type: " + type
         stop
       end
     end
   end
   
   define :parse_sync_address do |address|
     # puts address #-> ""/osc*/event/**""
     # puts get_event(address).to_s.split(",")[6] #-> " \"/osc:127.0.0.1:4560/event/project/0/started/bd_haus\""
     v = get_event(address).to_s.split(",")[6]
     if v != nil
       # puts v[3..-2] #-> "osc:127.0.0.1:4560/event/project/0/started/bd_haus"
       puts v[3..-2].split("/") #-> "["osc:127.0.0.1:4560", "event", "project", "0", "started", "bd_haus"]"
       return v[3..-2].split("/")
     else
       puts "Error in :parse_sync_address: v=nil"
       stop
     end
   end
   
   # function to define a unique name based on the type and unique id
   define :loop_name do |type, unique_id|
     return type + "_" + unique_id
   end
   
   # function to define the unique live_loop kill switch
   define :loop_kill_switch do |live_loop_name|
     return live_loop_name + "_kill"
   end
   
   # function to define the unique live_loop kill switch
   define :loop_kill_delta do |live_loop_name|
     return live_loop_name + "_delta"
   end
   
   define :setKillLiveLoopSwitch do |live_loop_name|
     loop_kill_switch = loop_kill_switch(live_loop_name).to_sym
     set loop_kill_switch, 1
   end
   
   define :maybeKillLiveLoop do |live_loop_name, loop_kill_switch|
     if get(loop_kill_switch) == 1
       set loop_kill_switch, 0
       stop
     end
   end
   #########################################################
   
   ## Start the background loop
   initStarted()
   ```
1. Run SonicPi. You should hear a background melody at 90 bpm
1. In a terminal window, run `./gradlew music --no-daemon`
    > This should start a Gradle build. The first time it executes will be a bit longer as it will download:
    >  - The Gradle distribution
    >  - Necessary dependencies 
    >    - 'com.illposed.osc:javaosc-core:0.7' -> Java OSC library
    >    - 'com.google.guava:guava:30.0-jre' -> Google Guava
1. You should eventually see something like
```
$ gw music --no-daemon
Using gradle at '/Users/facewindu/git/GradleSonicPi/gradlew' to run buildfile '/Users/facewindu/git/GradleSonicPi/build.gradle':

To honour the JVM settings for this build a new JVM will be forked. Please consider using the daemon: https://docs.gradle.org/6.3/userguide/gradle_daemon.html.
Daemon will be stopped at the end of the build stopping after processing

> Configure project :
NEW SonicPiChannel, with a semaphore with 1 permits
Configured SonicPi receiver: [OSCPortIn: listening on "0.0.0.0/0.0.0.0:57110"]
Configured SonicPi sender: [OSCPortOut: sending to "0.0.0.0/0.0.0.0:4560"]
build started
build finished

BUILD SUCCESSFUL in 21s
1 actionable task: 1 executed
```
The build just executes a single `music` task, in which I 'simulate' the creation of 100 tasks
running in parallel, each having a random duration between 0 and 2 seconds.

Normally, you should observe some 'thread got too far behind time' exceptions in SonicPi.

You can play with the settings in the `build.gradle` file. See the `tasks.create('music')`, to control
the number of parallel tasks and the artificial sleep for each of them.

