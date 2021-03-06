IPROF: A Portable Industrial-Strength Interactive Profiler for C++ and C
by Sean Barrett

Version 0.2


CONTENTS
  Overview
  User Manual
    Platform
    Instrumentation
      Private Zones
      Public Zones
    Initialization
    Processing Data
    Displaying Results
    Controlling Display
    Understanding CALL GRAPH output
    Performance Expectation
  Implementation Notes
  Version History

 
OVERVIEW

IProf is an interactive profiler which works by intrusively instrumenting 
code. Code is divided into zones by programmer-inserted statements. Zones 
are both lexically and dynamically scoped--all time spent within a 
lexically scoped zone, and any code which it calls which is not itself 
zoned, is attributed to that zone.

Profiling occurs interactively; time is divided into "frames", and the
profiler shows time spent on the previous frame (or a smoothed average
or possibly even a frame a second or two ago).

Like a traditional profiler, IProf records or can compute the number of 
times a zone is entered, the amount of time spent in the zone ("self 
time"), and the amount of time spent in the zone and its descendents 
("hierarchical time" -- "self + child time" in gprof).

Furthermore, IProf computes information along the lines of gprof--number of 
times a given zone is entered from any other specific zone; self and 
hierarchical time spent in a given zone on behalf of a specific parent 
zone, etc. (However, where gprof estimates this information based only on 
call counts, IProf measures the actual values. So, for example, IProf will 
accurately report if a ray-casting routine called by both physics and AI 
always spends longer per AI-call because the casts are longer.)

Precise information is available for recursive routines, including call 
depths etc. [The current version of IProf does not yet completely handle
reporting of recursive data, although it is measured correctly.]

Additionally, IProf provides all numbers in instantaneous form or as two 
differently weighted moving averages. It's easy to pause the profile 
updating so that you can switch between multiple views of the paused data 
set. Two optional flags allow trading off memory for deeper historical 
views. The cheaper option provides only zone-self-time history, suitable 
for a real-time graph of behavior. The more memory-expensive flag keeps a 
history of all the data for a certain number of frames, allowing full 
profile analysis of old frames.

IProf is designed for its monitoring/gathering mode to be "always on", even 
in release/optimized builds. The monitoring routines are designed to be 
reasonably efficient--the full hash on every function entry required by 
gprof is avoided in most cases--and the programmer can minimize the impact 
by limiting the instrumentation to relatively large routines. (One could 
certainly instrument a vector add function and possibly get useful call 
count data from it, but the monitoring overhead would be significant and 
noticeable in that case.) In combination with history information, it 
becomes possible to run an application, notice poor behavior, pause the 
(always on) profiling and the application, and start browsing through the 
historical profiling information.

IProf uses both per-call monitoring and a separate per-frame 
gathering/analysis phase. The latter is itself instrumented so the overhead 
due to it is easy to see.


USER MANUAL

These sections document the necessary code you must use and code changes 
you must make to use the profiler.

The profiling system expects to be able to use any identifier which is 
prefixed with "Prof_" with exactly that pattern of uppercase/lowercase 
(i.e. "PROF_" and "prof_" can be used freely by other code).


COMPILING THE PROFILING SYSTEM

The profiler was developed using MSVC 6.0, but should be reasonably 
portable. The implementation files are provided as .C files so they can be 
used with C compilers; however, they can be renamed to C++ files and 
compiled in that form. The implementations automatically insert extern "C" 
on the public routines. Internal routines will use either C or C++ linkage 
depending on which way you compile them; you must compile all the profiler 
files as either C or C++, without intermixing.

[[ NOTE: Originally the code was written in C++, and then it was
   converted to compile with C, and then some additional small changes
   were made. As of this writing, I haven't actually tested compiling
   everything as C++ again. Feel free to test for me. Or just compile
   the C files as C--you can still USE the C++ interfaces fine.]]

Needed files:
   prof_win32.c    -- Win32 implementation of seconds-based timer
   prof_gather.c   -- raw data collection
   prof_process.c  -- high-level data collection, report generator
   prof_draw.c     -- opengl rendering interface

   prof.h          -- public front-end
   prof_win32.h    -- Win32 implementation of fast integer timestamp
   prof_gather.h   -- instrumentation macros (included by prof.h)
   prof_internal.h -- private interfaces


PLATFORM SUPPORT

IProf requires a small amount--less than fifty lines--of platform-specific
code.

Win32 under MSVC is automatically supported with no further effort on your
part, using the files prof_win32.c and prof_win32.h

To use other platforms, just create equivalent files for your platform. The 
C file contains a routine for getting an accurate floating point time 
reading; the H file contains the definition of a 64-bit integer type and a 
fast routine for reading a timestamp of that size. If 64-bit math isn't 
available on your platform, or if your timestamp is only 32-bit, you can 
replace the 64-bit type with a 32-bit type, as long as that item won't 
overflow in the course of running the application. (A 31-bit millisecond 
timer is good for 24 days, but is very imprecise for this application.) If
reading the timestamp is slow, you will want to minimize how often the zone
entry and exit points are called.

Also required is a display interface; an opengl one is provided, although
others would be easy to code. (The primary display is purely textual, and
is available through a text interface.)


INSTRUMENTATION

First, #include "prof.h" in files that need profiling.

The flag Prof_ENABLED determines whether the monitoring code is compiled or 
not, to make it easy to turn off all profiling code for final shippable 
builds. Additional flags controlling amount of history data and memory
usage therein are defined at the top of the file prof_process.c and should
just be changed there since they affect no other files.

There are two main ways of instrumenting, and each offers a C++ interface
and a C interface.

  Private zones
     C++    Prof(zone);

     C      Prof_Begin(zone)
            Prof_End

  Public zones
            Prof_Define(zone);
            Prof_Declare(zone);

     C++    Prof_Scope(zone);

     C      Prof_Region(zone)
            Prof_End

Zone names--indicated by "zone" above--must obey the rules for identifiers, 
although they can begin with a number, and they exist in a separate 
namespace from regular identifiers.

So these are valid zone names:
    my_zone_2
    2_my_zone
    __

And these are NOT valid zone names:
    "my_zone"
    my_class::my_zone


PRIVATE ZONES

The simplest, and highly recommended, approach to instrumentation is to 
create a private zone which only exists in a single location. In C++, you 
do this by declaring a lexically scoped zone with a statement which behaves 
semantically like a variable declaration:

   // C++ instrumentation
   void my_routine()
   {
      Prof(my_routine_name);
      ... my code ...
   }

This will cause all time spent after Prof(my_routine_name) to accumulate in 
a zone in the profiling reports labeled "my_routine_name". The zone ends 
when the name goes out of scope, that is, when a destructor would be called 
corresponding to this declaration.

Zones don't have to appear at routine-level function scope; for example:

   // C++ instrumentation
   void my_routine()
   {
      Prof(my_routine);
      ...                // zone my_routine
      if (...)
      {
         Prof(my_routine_special_case);
         ...             // zone my_routine_special_case
      }
      ...                // zone my_routine
   }

Instrumenting in C requires more work, because C doesn't provide 
destructors, so it's not possible to lexically scope zones automatically. 
Instead, the programmer must insert Begin/End pairs and make sure those 
pairs are accurately balanced. All paths out of a function must be 
accounted for. A crash or severe slowdown is likely to occur with 
unbalanced pairs.

   // C instrumentation
   void my_routine(void)
   {
      Prof_Begin(my_routine)
      int x = some_func();
      if (x == 0) {
         Prof_End
         return;
      }
      ...
      Prof_End
   }

Prof_Begin() is declaration-like; however, it takes no trailing semicolon. 
(This is necessary so it can be compiled out; C doesn't allow the empty 
statement ";" to precede variable declarations.) Prof_End takes no 
trailing semicolon or parentheses to help remind you of this. (You can 
change the definition of Prof_End in prof_gather.h if you don't like that.)

Profiling instructions like Prof() and Prof_Begin() can be placed anywhere 
that variable declarations are legal; generally you want to define them 
before other variables so the variable initializations are profiled.

The C interfaces are also available in C++ if you should want to use a not-
exactly-lexically-scoped zone, e.g. end a zone before the destructor would 
go out of scope. (You can't, however, end Prof() with Prof_End.)


PUBLIC ZONES

If you define multiple private zones with the same name, they will be 
treated as entirely unrelated zones that happen to have the same name, and 
you will see the same name multiple times in the profiling output.

Instead, you probably want to use public zones, to use the same zone in 
multiple regions of code. For example, we might have two routines that 
serve the same purpose which we always want to measure as one. Or we might 
have two blocks of code within a single routine which we want to credit to 
the same zone.

To do this, first define the zone with Prof_Define(zone), and then use it 
with Prof_Scope(zone) [C++] or Prof_Region(zone) ... Prof_End [C].

   // C++ instrumentation
   Prof_Define(my_routine);

   void my_routine_v1()
   {
      Prof_Scope(my_routine);
      ...
   }

   void my_routine_v2()
   {
      Prof_Scope(my_routine);
      ...
   }

or

   // C instrumentation
   Prof_Define(my_routine);

   void my_routine_v1(void)
   {
      Prof_Region(my_routine)
      ...
      Prof_End
   }

   void my_routine_v2(void)
   {
      Prof_Region(my_routine)
      ...
      Prof_End
   }

Because Prof_Define defines an actual global symbol (if used at file 
scope), the symbol can even be referenced from other files by saying:

   extern Prof_Declare(my_routine);

   void my_routine()
   {
      Prof_Scope(my_routine);
   }

You can use 'extern "C" Prof_Declare()' or Prof_Define() to share a zone 
between C and C++ code.


USER MANUAL - INIIALIZATION

The profiling system is self-initializing.


USER MANUAL - PROCESSING DATA

Every frame, you should call Prof_update(). Prof_update() will gather 
results and record frame-history information on the assumption that each 
call is a frame. Prof_update() takes a boolean flag which indicates whether 
to update the history or not; passing in false means profiling is "paused" 
and doesn't change.

You might wire this to its own toggle, or you might simply pass in a pre-
existing flag for whether the simulation itself is active or not, thus 
allowing you to pause the simulation and automatically pause the profiling. 
(On the other hand, if you're profiling a renderer, you might want to 
pause the simulation and keep profiling.)


USER MANUAL - DISPLAYING RESULTS

IProf offers two separate types of display: the report, which is primarily 
textual, and the graph, which is entirely graphical.

If you're using OpenGL, output is straightforward. For the text report, 
call Prof_draw_gl() with the display set to a 2d rendering mode--one that 
can use integer addressing, e.g. integers the size of pixels, virtual 
pixels (e.g. a 640x480 screen regardless of actual dimension), or even 
characters. Set the blending state to whatever blending mode you want for 
the report display. For the  graphics report, call Prof_draw_graph_gl(). 
Details of the parameters to these functions are available in the header 
file.

For other output devices (Direct3D, text), you'll have to write your own 
functions equivalent to Prof_draw_gl() and Prof_draw_graph_gl(). These 
should not be too difficult; these functions don't compute any of the 
profiling information; they simply format a text report or dataset to the 
screen. The text report format consists of several title fields to be 
printed, and then a collection of data records. Each data record has a name 
and an indentation amount for that name (used for call graph 
parent/children formatting), a collection of unnamed data "values", and a 
flag field indicating which of the data values should be displayed. 
Additionally, data records have a "heat" which indicates how rapidly 
changing they are, and one record may be "highlighted" indicating a virtual 
cursor is on that line.

[[ In practice, Prof_draw_gl makes few enough GL calls that maybe it's 
worth modularizing things out further. ]]


USER MANUAL - CONTROLLING DISPLAY

IProf features some easy-to-use UI elements that allow program-direct 
control or user-interaction-based control over what data is reported. 
Simply hook these calls up to hotkey presses to complete your working 
profile system. (You could even write code to support mouse clicking on the 
report by calling Prof_set_cursor and on the graph by calling 
Prof_set_frame, but the hit detection is up to you.)

These are in rough order of the priority with which you might want to 
implement them.

Most important

   Prof_set_report_mode(enum ...)
      Selects what to show in the report:
         Prof_SELF_TIME: flat times sorted by self time
         Prof_HIERARCHICAL_TIME: flat times sorted by hierarchical time
         Prof_CALL_GRAPH: call graph parent/children information

   Prof_move_cursor(int delta)
      Move the cursor up-or-down by delta lines

   Prof_select(void)
      Switch to call graph mode on whichever zone is currently selected

   Prof_select_parent(void)
      Go to largest-hierarchical-time parent of the active zone in
      the call graph. (Roughly like "go up a directory".)

Important if you support history

   Prof_move_frame(int delta)
      Move backwards or forwards in history by delta frames

Not too important

   Prof_set_average(int type)
      Selects which moving average to use (0 == instantaneous, 1=default);
      only meaningful if frame# = 0; when looking at history, instantaneous
      values are always used.

   Prof_set_frame(int frame)
      Selects which history entry to view (0==current, 1==previous, etc.)

   Prof_set_cursor(int pos)
      Set the position of the up-and-down cursor.

   Prof_set_recursion(enum ...)
      Selects whether to show recursive routines as a single zone or
      as a series of distinct zones for each recursion level.
      [[ currently unimplemented ]]


UNDERSTANDING CALL GRAPH OUTPUT

The call graph output focuses on a single zone, and provides information 
about the parents (callers) and children (callees) of that zone.

The general format is something like this:

     zone                self   hier   count
       +my_parent1       0.75   2.50     4.0
       +my_parent2       1.00   3.25     6.0
    -my_routine          1.75   5.75    10.0
       +my_child1        1.00   2.00    15.0
       +my_child2        0.25   1.50   500.0
        my_child3        0.50   0.50     3.0

"self" indicates self-time (time in this zone), "hier" is hierarchical-time 
(time in this zone or its descendents), and "count" is the number of times 
the zone was entered. (Entry counts are inherently integral, but are shown 
as floating point since they may be a moving average of several integers.)

Currently the zone "my_routine" is being examined. It accounts for 5.75 
milliseconds of time between itself and the zones it calls. 1.75ms are 
spent in itself. The zone was entered (called) 10 times this frame.

The difference between my_routine's self time and hierarchical time is 
4.00ms; that much time must be being spent in its descendents. Its 
immediate children--the zones that my_routine calls directly--appear below 
it on the table. The hierarchical times of each child represents the time 
spent in that child and all its descendents *on behalf of my_routine*--
other calls to that child are not counted. Thus, the sum of all the 
children's hierarchical time should account for all time spent in 
descendents of my_routine; hence, the sum of the child hier times is 4.00, 
identical to the difference between self and hier for my_routine.

Above "my_routine" in the chart is information about the callers of 
my_routine. However, the timings and counts in this section are not the 
self and hierarchical times of the parent functions themselves--there is no 
sensible meaning of "on behalf of my_routine" for the parents. Instead, the 
self, hier, and count fields show the time spent *in my_routine* on behalf 
of those parents. Thus, for each field, all of the parent entries sum to 
the corresponding entry in my_routine. Again, these are computed exactly. 
If my_routine was the public interface to a raycaster called by both AI and 
physics, but it passed the raycast on to further routines which were 
themselves explicitly zoned, then most of the my_routine time would be 
spent in descendents. This would show up in the "hierarchical time" field, 
and the parent zones, AI and physics, would show that hierarchical time 
attributed accurately.

There is additional data available in the system--it would be possible to 
drill down into lower-level functions and still attribute them to zones 
several parent levels above; there just isn't currently any user interface 
or computation functionality to do it.


PERFORMANCE EXPECTATION

Except for recursive routines (see Implementation Notes section), the 
expected performance on zone entry comes from running roughly the following 
code:

   extern Something *p0,*p1;
   if (p0->ptr_field != p1) { ... /* rarely runs */ }
   p0->int64_field0  = RDTSC; // read timestamp counter
   p0->int32_field  += 1;
   p1->int64_field1 += p0->int64_field0 - p1->int64_field0;
   p1 = p0;

Zone exit costs a bit less.


IMPLEMENTATION NOTES

IProf uses two relatively unknown techniques to produce accurate call 
information with minimal overhead. The first technique produces accurate 
call information at a similar cost to gprof's mcount monitoring; the second 
reduces the overhead.

_Zone Stack Tracking_

gprof's mcount technique combines two separate measurements. At every 
function entry, the function and the caller (grabbed from the return 
address on the stack) are hashed to determine a unique "data-gathering 
slot", and an integer in that slot is incremented. Thus, exact pairwise 
call counts are computed. Simultaneously, gprof periodically samples the 
instruction pointer to measure the time spent in any given routine--"self 
times". Hierarchical times are computed by distributing the self times up 
the tree based on the call graph counts. (If routine X is called 9 times 
from routine Z, and one time from routine Y, then 90% of X's time is 
attributed to Z, and 10% to Y.)

An intrusive profiler which samples a timer at zone entry and again at zone 
exit will compute accurate hierarchical times. By keeping a stack of zones, 
it's possible to compute accurate hierarchical and self times. The stack of 
zones also provides caller information, so hierarchical and self times can 
be attributed to each unique pair of caller & callee zones (via hashing). 
This will allow much more accurate attribution. In fact, it is sufficient 
to compute exact values for all the information gprof outputs, except in 
the face of recursion. Performance is fairly good; unlike a single-zone 
intrusive profiler, which must measure both self and hierarchical time, 
since neither can be derived from the other, the zone-pair profiler can 
only measure hierarchical time; self-time can be derived from hierarchical 
time (but not vice versa).

A further improvement is, instead of having one data-gathering slot per 
zone--that is, representing the state of the top of the zone stack--and 
instead of having one data-gathering slot per caller/callee zone pair--that 
is, representing the state of the top two entries of the zone stack--to 
have one data-gathering slot per unique full stack state. This can be done 
straightforwardly by building the stack as a linked list (creating an 
inverted tree--a tree of all stack states with only parent-pointer links), 
and hashing the "zone to be pushed" and the current stack to find the new 
stack. Thus the cost of the hash computation is essentially identical to 
the previous case. If every zone is only called from one specific place, 
there will still only be one data-gathering slot per zone; if a routine is 
recursive, it will create a large number of data-gathering slots, one for 
each depth of recursion. (A complex mutually recursive program like a 
compiler might generate an unreasonable number of unique states.)

With zone-stack tracking, it's possible to measure only either hierarchical 
time or self-time and derive the other. Hierarchical time is actually more 
efficient to measure, but it leaves handling the top-level overarching 
global state as a special case (since it will have a timer that starts but 
never ends). It's easier to instead measure self-time and rederive 
hierarchical time. Moreover, a recursive routine will automatically 
"overcount" hierchical time (it's accrued at each level of the hierarchy), 
requiring significant fixup. It's more straightforward to just compute the 
recursive data correctly from the self times in the first place.


_Hash Cacheing_

Although the hash lookup described above is coded to proceed as quickly as 
possible if the hash hits on the first probe, it still requires enough 
computation and a function call that it is worth avoiding if possible. To 
that end, each zone-entry location declares a hidden static variable 
private to that zone-entry point which caches the hash lookup. At zone-
entry, the code checks the cache's "next node in the linked list" field 
with the current stack state. If the two are equal, then the cache is 
valid, and no hash lookup occurs. If it does not much, then the cache is 
wrong, and the hash lookup proceeds, and updates the cache. The cache is 
initialized to a impossible value, so the first time the code is run a hash 
lookup always occurs.

The result is that in the normal case, a routine called from a single 
place, the cache is always valid (after the very first call). Furthermore, 
the branch will always predict correctly, since it always branches 
identically. However, for a routine that is called from several places, 
there is a "switching" overhead each time it's called from a different 
place. So, for example, a raycaster called by both physics and AI might pay 
the overhead twice per frame, if all the AI calls occur before all the 
physics calls. However, a common low-level routine (e.g. a vector add) 
called alternately from two different zones would have to perform the hash 
lookup every time.

The actual common "failure" case is a recursive routine, for which, each 
time the routine is entered, the state of the call stack is different from 
the last time, thus almost always paying the hash lookup case. For 
something like a recursive linked list traversal, the hash occurs every 
time. (It doesn't matter if the routine is tail-recursive; once you insert 
the profiling instrumentation, it's no longer tail-recursive.) A full 
binary tree traversal will always enter a different zone-stack-state from 
last time, except after reaching a left-child leaf. (The recursion then 
returns and then goes down to the right child, which is at the same height 
as the left child.) So a full binary tree traversal will have to hash about 
3/4 of the time. A full quadtree traversal will have to hash about 2/5 of 
the time. If the traversal is doing anything complicated, this should not 
be a problem; but if it's a simple traversal, the performance overhead may 
be significant. Like the vector add case, it may be better to remove 
instrumentation from low-inherent-cost recursive routines except when 
absolutely needed. Of course, it's easy enough to compare performance 
behavior before and after adding the instrumenting and see if the overhead 
is acceptable.


VERSION HISTORY

version 0.2 -- 2003-02-06 STB
   - Significant interface changes to Prof_draw_gl:
      - accepts floating point instead of int for 2d screen metrics
      - accepts a total width and height of the display and conforms
        to that
      - accepts a precision specification for display of time values
   - added little '+' and '-' signs reminiscent of list displays
     so you know which ones can be drilled down on
   - expanded this doc's description of what's legal for a zone-name
   - fixed an error trying to compile the C files as C++
   - added Prof_select_parent() for moving up the tree

version 0.1 -- 2003-02-05 STB
   - First public version, heavily refactored, 1500 lines
   - win32 timing interface and smooth "moving average" code derived
        from Jonathan Blow's Game Developer Magazine articles
   - missing functionality:
      - correct attribution of time to zones that are parents of
        recursive zones in call graph view (hierarchical times don't
        bubble up correctly)
      - spread recursion display (displaying each depth of a recursive
        zone as if it were a separate zone)