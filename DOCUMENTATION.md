Icarus OBC Round 2 - Documentation

1.Overview

I approached the exercise by first reproducing the flight software system onto my system locally, by following the steps which were given.Then I inspected the source code and looked at the runtime errors in the logs. Debugging played a major part in this problem statement. I used GDB for the same and debugged the code in the CLI. After every fix, I rebuilt and reran the complete 2250 tick simualtion.

The final solution branch completes all 2250 ticks without any major runtime warnings, memory corruption or segmentation fault. There were a few other warnings which were already present in the existing codebase like:
_FORTIFY_SOURCE redefined
address-of-packed-member
gets() dangerous / implicit declaration
physics_link_check
unused variable
Some of these warnings were not related to the mission faults I had identified and changing them would add unnecessary code changes. Also they were already present in the codebase, therefore fixing them wasn't my main priority.

The major issues that I identified and fixed were:
1) Incorrect physics library linker path
2) Sensor copy buffer overflow
3) Telemetry ring buffer overrun that was causing scheduler corruption and a tick 1026 crash
4) Thermal scheduler runtime counter overflow

The separate fault-injection branch successfully triggered the intentional failure case at tick 325.

2.Build(Linker) issue

Observation:-
The first time I ran make, it failed at the linker stage with:
cannot find -lobc_physics

The Makefile searched -L./lib/static while the actual binary was lib/libobc_physics.a and lib/static/ did not exist.

Root Cause:-
The linker path which was present did not match the location of the physics library.

Fix:-
I changed -L./lib/static to -L./lib. Then the link was successful.

3.Sensor Copy Buffer Overflow

After running the simulation this was another warning I came across:[ WARN ] Sensor copy canary modified at tick 16
The relevant code was inside temp_guarded_copy().
The destination buffer was : uint8_t dest[ SENSOR_COPY_DEST ];
with SENSOR_COPY_DEST = 16. The SENSOR_COPY_DEST is a preprocessor macro or constant that defines the allocated size of the destination buffer in memory and it is set to 16 bytes.
The copy length was calculated as:
uint16_t copy_len = (uint16_t)(32 - temp);

Investigation:
I used GDB and debugged at tick 16 by placing a breakpoint there.
This was what I found:
temp = -2
copy_len = 34
destination size = 16

Therefore, the code attempted to copy 34 bytes into a 16-byte destination which was already set.
Then I verified the corruption by looking at the adjacent canary.
I could see that:
Before memcpy():
6A 6A 6A 6A 6A 6A 6A 6A

After memcpy():
AB AB AB AB AB AB AB AB

So this gave me the confirmation that there was an overflow and it was outside the boundaries that were already set.

Fix:
The copy was limited to the destination capacity. The following was the change I had made:
if (shifted < gate && copy_len <= sizeof(frame.dest)) {
    memcpy(frame.dest, source, copy_len);
}

After this code change, the tick 16 canary warning disappeared and the 2250 tick simulation ran successfully as well.

4.Telemetry Ring Buffer Overrun

Overrun basically means overflow.
When I came across this crash at tick 1026, I tried to understand and study about this, then I found out its called a telemetry ring buffer overrun.
It occurs when  a system generates flight or sensor telemetry data faster than it can process it, causing new data to wrap around and overwrite old data before it has been read.
I was also curious what exactly a ring buffer was and I read quite a lot about it.The following are the conclusions I came to:
A ring buffer also called circular buffer is a fixed size memory array that treats the end of the array as if it connects back to the beginning, forming a loop. It uses two pointers to manage data flow:
Write Pointer: Moves forward as new telemetry data (like speed, temperature, or pitch) is generated
Read Pointer: Follows behind , moving forward as the system reads, processes, or transmits that data to a ground station or log file

Because a ring buffer has a fixed capacity, the Write pointer can theoretically catch up to the Read pointer from behind if data is coming in too quickly.

Observation:
I ran the simulation and it crashed at tick 1026 inside:- state->tasks[ pick ].run_count++;
The scheduler was dereferencing an invalid state->tasks pointer.

Investigation:
I put a watchpoint on state->tasks and caught the pointer changing from the valid task array address to 0xeee25d9400.

The backtrace(bt is the shortcut for it that I used while debugging) showed the write occurring during telemetry insertion.

The telemetry buffer was:

#define QUEUE_SIZE 1024
TelemetryFrame queue[ QUEUE_SIZE ];
and the size of the telemetry frame was set to 24 as follows:
sizeof(TelemetryFrame) = 24

The insertion code advanced i.e got incremented
state->telemetry_cursor++;

but it never got sent back to the beginning of the queue.

After the final valid slot, telemetry writes continued beyond the queue and it eventually overwrote adjacent AppState memory, including state->tasks.

Fix:
I fixed it so that once the counter hits the end of the line, it gets sent right back to the beginning:

state->telemetry_cursor++;

if (state->telemetry_cursor >= state->shared.queue + QUEUE_SIZE) {
    state->telemetry_cursor = state->shared.queue;
}

Then the tick 1026 segmentation fault disappeared and the full simulation of 2250 ticks completed.

5.Thermal Scheduler runtime overflow

Observation:
After fixing the memory corruption issues, I saw that THERM_STALE became 1 around tick 1700.

Root Cause:
ThermalTaskState.runtime_ms was stored as:
uint16_t
while the scheduler increased it by 38 ms every tick.
A 16-bit counter can represent only up to 65,535, so the runtime counter overflowed during the simulation. The wrapped value changed the scheduler comparison and caused the thermal monitor to use the previous sample that was present.

Fix:
I changed uint16_t runtime_ms to uint32_t runtime_ms

After I made this change, THERM_STALE: 1 no longer appeared and the simulation ran successfully.
The following was what I got in my terminal after it ran successfully:
TICK:2250 | ORBIT:15 | TEMP:-15 | VBAT:2.438 | SAFE:1 | THERM_STALE: 0

6.Final solution

The final corrected build completed the full task as follows:
TICK:2250 | ORBIT:15 | TEMP:-15 | VBAT:2.438 | SAFE:1 | THERM_STALE: 0

7.Fault Injection

For the required fault injection branch, I used the provided actuator interface that was present:
cmd_set_actuators(100.0, 5000.0);

I turned the heater up to 100% and cranked the flywheel speed up to 5000 rpm.
The system is designed to trigger a structural failure only if both of these dangerous conditions happen at the same time:
1) The temperature climbs above 85 degrees celsius i.e actual_temperature > 85.0
2) The shaking (vibration level) goes above 12.0 i.e vibration_amplitude > 12.0

Spinning at 5000 RPM instantly created a vibration level of 13.This meant the shaking limit was breached right away, well before the machine got hot:
0.5 + 5000 * 0.0025 = 13.0
so the vibration threshold is exceeded before the thermal threshold.
The heater raises temperature by approximately:
0.25 - 0.05 = 0.20 degree Celsius/tick

Then I rebuilt the system and ran the simulation on this branch and this was what I came across:

FINAL TICK: 325 | TEMP: 85.00 | VIB: 13.00
The process exited with status:
42
which was matching the intentional structural failure path.

Then I had the json response recorded in fault_manifest.json and the following gets displayed:
{"expected_crash_tick":325}












