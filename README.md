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
       use_synth :fm
       use_octave -2
       3.times do
         play c[0]
         sleep 1
         cue :init
       end
       play c[2]
       sleep 0.5
       play c[1]
       sleep 0.5
       c = chords.tick
     end
   end
   
   #########################################################
   # BUILD : when the Gradle builds emits an build event
   #########################################################
   
   define :buildStarted do |thread_name, args|
     sync :init
     loop do
       use_synth :tb303
       r = [0.25, 0.25, 0.5, 1].choose
       play c.choose, attack: 0, amp: 0.3, decay: 0.1
       sleep r
       maybeKillThread(thread_name)
     end
   end
   
   #########################################################
   # TASK: : when the Gradle builds emits an task event
   #########################################################
   
   define :taskStarted do |thread_name, args|
     sync :init
     loop do
       use_synth :piano
       play c, attack: 0, decay: 1
       sleep 0.1
       maybeKillThread(thread_name)
     end
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
     
     threadName = thread_name(type, unique_id)
     case action
     when "started"
       event_started(type, threadName, args)
     when "finished"
       event_finished(threadName)
     else
       puts "Unknown action: " + action
       stop
     end
     # Ping back that we are ready to receive a new OSC event
     # The Gradle builds waits for this 'ack' OSC message before moving on
     osc_send "localhost", 57110, "/ack"
   end
   
   
   # Creates a thread creating a live_loop which is monitoring when it should stop itself
   define :event_started do |type, thread_name, args|
     in_thread do
       case type
       when "build"
         buildStarted(thread_name, args)
       when "task"
         taskStarted(thread_name, args)
       else
         puts "Unknown type: " + type
         kill
       end
     end
   end
   
   # Instruct the started thread for that name to be killed
   define :event_finished do |thread_name|
     setKillThreadSwitch(thread_name)
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
   define :thread_name do |type, unique_id|
     return type + "_" + unique_id
   end
   
   define :setKillThreadSwitch do |thread_name|
     puts "set kill switch for: " + thread_name
     set thread_name, 1
   end
   
   define :maybeKillThread do |thread_name|
     if get(thread_name) == 1
       puts "killing: " + thread_name
       Thread.kill(Thread.current)
     end
   end
   #########################################################
   
   ## Start the background loop
   initStarted()
   ```
1. Run SonicPi. You should hear a background melody at 90 bpm
1. In a terminal window, run `./gradlew play`
    > This should start a Gradle build. The first time it executes will be a bit longer as it will download:
    >  - The Gradle distribution
    >  - Necessary dependencies 
    >    - 'com.illposed.osc:javaosc-core:0.7' -> Java OSC library
    >    - 'com.google.guava:guava:30.0-jre' -> Google Guava
1. You should eventually see something like
   ```
   $ gw play
   Using gradle at '/Users/facewindu/git/GradleSonicPi/gradlew' to run buildfile '/Users/facewindu/git/GradleSonicPi/build.gradle':
   
   To honour the JVM settings for this build a new JVM will be forked. Please consider using the daemon: https://docs.gradle.org/6.3/userguide/gradle_daemon.html.
   Daemon will be stopped at the end of the build stopping after processing
   
   > Task :music_0
   NEW SonicPiChannel, with a semaphore with 1 permits
   Configured SonicPi receiver: [OSCPortIn: listening on "0.0.0.0/0.0.0.0:57110"]
   Configured SonicPi sender: [OSCPortOut: sending to "0.0.0.0/0.0.0.0:4560"]
   build finished
   
   BUILD SUCCESSFUL in 35s
   500 actionable tasks: 500 executed
   ```
   The build just executes the `play` task, which depends on 500 other tasks.
1. If something goes wrong, execute `./gradlew --stop`
