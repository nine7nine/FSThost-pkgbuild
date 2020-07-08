# Wine-NSPA + Linux-NSPA

Git Repo for my wine-nspa / linux-nspa flavours for Archlinux.

That said, nothing makes this exclusive to Arch. For example, Rolesyuk has kindly
packaged binaries of Wine-NSPA for Ubuntu 20.04 + He has binaries for a PREEMPT_RT kernel with
futex-multiple-wait to support fsync in Wine-NSPA. 

https://github.com/rolesyuk/rt_audio/releases

Furthermore, he has scripts to build these packages + grab jack2 1.9.14 deb on 20.04;

https://github.com/rolesyuk/rt_audio

_______
 # Linux-NSPA; my custom kernel.

Nothing magical here, just some patchwork that I use on my Laptop / on Arch. focused on 
low-latency, some optimizations and improvements -- but the main reason for it's existence is
for futex-multiple-wait for wine-nspa. 

- contains futex-multiple-wait patch(actually, 2 versions).
- some of intel's patchwork. 
- gcc optimization
- the percpu-rwsem rewrite (lives in -rt 5.6.x, as well).
- swait stuff from -rt. (not really -rt specifc though. in either case.).
- a couple fixes for pesky issues (for me).

set to 1000hz. I use threaded interrupts. etc. i'm not using preempt-rt right now. the mainline
kernel is fairly good these days for proaudio. that said; i still add the -- threadirqs -- kernel 
boot parameter. set to ensure all interrupts are force threaded (and can be prioritized), 
except those marked IRQF_NO_THREAD. that's also why I've patched the designware i2c and AMD pinctrl 
drivers -- they puke into the kernel ring buffer on boot, about not wanting to be threaded.

mainly posting for backup for myself => However, it's also here for anyone wanting to use my wine-nspa 
builds; as you will need kernel support. so the futex-multiple-wait support is available. 5.2.x/5.5 diff version.
this kernel runs very well, for my purposes.

i also only 'loosely' follow Arch kernel updates. I don't need to chase updates and i need a consistent
system for testing changes - rolling can introduce regressions or other issues, so I'm sticking with 
5.6.x for a short-while, while i hack on wine a bit more. but I will update, obviously. (linux 5.8?).
_____
# Wine-NSPA; wine geared for proaudio. (*WIP*/alpha)

this patchwork and hacking is all geared towards improving prouadio / VST support. Because of that, one
of the central focuses is improving RT/threading support and leveraging fsync to improve synchronization.
These issues tend to be the biggest problems with running proaudio software in Wine.

I'm also willing to accept some hacks or workarounds that may allow specifc VSTs to work, as long as
they don't break things in Wine-NSPA.. and I will implement other useful stuff, if/when it happens.
(I'm sitting on some patchwork that I haven't added yet, but will eventually).
_____  
WINE-STAGING: 

Built on Wine-Staging (5.9 is my base, for now).

I use these staging settings. (wine environment variables)
  
  * STAGING_SHARED_MEMORY=1
  * STAGING_WRITECOPY=1
  
The shared memory code indicates that when things get of out synchronization, this feature can cause 
instability. However, given that synchronization is improved by fsync and additionally made more robust
by Wine-NSPA's RT support; it makes sense to enable this.
____
FSYNC: 

Fsync support aka: hybrid synchronization in Wine. (requires kernel support).
 
  Enabled With: (wine environment variables)
  * WINEFSYNC_SPINCOUNT=128
  * WINESYNC=1 (0 - disables).
_____
WINE-NSPA's RT SUPPORT: 

Initial implementation of Windows' PROCESS_PRIOCLASS_* (process priority classes), 
combined with PROCESS_PRIOCLASS_REALTIME thread prorities mapped to SCHED_FIFO or SCHED_RR. This allows 
far better prioritization of threads. 

  Enabled with: (wine environment variables)
  * WINE_RT_POLICY=RR (SCHED_RR) or FF (SCHED_FIFO) 
  * WINE_RT_PRIO=75
  
Both of these env variables MUST be set. 
____
WINE-NSPA's THREAD PRIORITIES & DESIGN:
  
Unlike the wine-staging RT patch, I don't allow setting the wineserver thread priority (independently). I'm doing 
this  for good reason. more on that below... Instead, we want this priority/thread placement;
  
 - MAX PRIORITY = FF
 
   * Wineserver, Kernel-mode APC && RT prioclass THREAD_PRIORITY_TIME_CRITCAL threads.
   
 - HIGH PRIORITY = RR or FF (WINE_RT_POLICY) 
 
   * any PROCESS_PRIOOCLASS_REALTIME threads, just below THREAD_PRIORITY_TIME_CRITICAL threads.
   
 - NORMAL/TS = SCHED_OTHER (+ niceness, if applicable)
 
   * everything else

This ensures that the kernel-mode APC can preempt user APCs. it also ensures that any critical threads in 
an app will have the same priority as those in Wine-NSPA's core / Wineserver. -- while also making sure that in both 
cases; 1). they can preempt the less important RT prioclass threads in apps 2). they won't be preempted. 
Synchronization is another very important consideration, here.  

Another difference; WINE_RT_PRIO is a MAX value and decrements; this may be personal taste, but it's a
strong preference. I prefer to set the MAX priority level for Wine-NSPA. Pin all of those MAX priority threads, then 
the rest decrement, just below.. 
  
  NOTE: I actually do set WINE_RT_PRIO very high (78) on my machine. just below Jack. for DAWs, you might 
  have to be careful how high you set WINE_RT_PRIO, as you don't want to interfere with your DAW's most
  important threads (really though, these will probably be Jack audio threads)... fyi, htop with t + k options
  easily shows this stuff.
____  
WINE-NSPA's RT POLICIES & DESIGN:

Wine-NSPA allows setting the RT policy, as shown above (WINE_RT_POLICY). For Wine-NSPA's core / most critical threads,
we force SCHED_FIFO + highest priority. For slightly lower priority realtime threads, we can set RT policy to SCHED_RR 
or SCHED_FIFO. I think with so many threads at the same/similar high priority and due to Windows scheduler's own 
behaviour, SCHED_RR is likely more appropriate for some of these proaudio plugin's realtime threads.

Note: Windows' highest priority threads use Round-Robin and therefore have a quantum / timeslice. 

Really, SCHED_RR is designed for running many threads at the same priority. -- When a RR thread's  quantum/timeslice 
expires, the scheduler puts the thread at the tail/end of the runnable list, This allows other RR threads of 
the same priority to run -- which results in RR threads having more even/equal CPU time... So I'm trying to better 
emulate the scheduling/thread behaviour of Windows, to some extent. -- while ensuring our RT threads are serviced
more evenly. 

I think it's important, espcially when running lots of VSTs; as separate processes and/or with big phat 
mult-threaded, DSP heavy plugins like Kontakt (which is a plugin i use/do daily, along with many NI plugins). 

 - SCHED_RR also has a tunable; /proc/sys/kernel/sched_rr_timeslice_ms

by default, sched_rr_timeslice_ms = 100 (on my 1000hz kernel). I don't recommend messing around with this, if
you don't know what you are doing, adjusting the timeslice can easily hurt performance. This works two ways; If 
you set this value too small/short; you will cause overhead with too much switching between threads. If the value 
is too large; you will end up delaying other RR threads from having time on the CPU (for each thread!)... At best, 
you might be able to 'hedge your bets' a bit, slightly reducing the value - somewhere between 50-100.

The application threads in Wine-NSPA that can use RR; don't actually need to be on the CPU non-stop. So having a
time quantum is a good thing. I've selectively/carfeully chosen the threads that I allow this behaviour with and 
I believe that it's a good fit for Wine and our RT use-case... These aren't critical threads or synchronization 
threads (which should be FF), but these absolutely want to be RT threads.  

That all said - I still have preserved the ability to switch them to FF, if desired.

NOTE: Jack related threads (from a VST) are still SCHED_FIFO. changing the WINE_RT_POLICY, 
only affects a subset of Windows' PROCESS_PRIOCLASS_REALTIME threads, whose RT policy I allow to be set. 
____
  VERY IMPORTANT NOTE:
  
  If using an older/out-dated build of JACK (anything below v1.9.13), then (unfortunately) you are required to 
  execute the below commands;

  * sudo setcap cap_sys_nice+ep /usr/bin/wine64-preloader
  * sudo setcap cap_sys_nice+ep /usr/bin/wine-preloader  
  * sudo setcap cap_sys_nice+ep /usr/bin/wineserver
  
  If using Rolesyuk's Ubuntu packages (Ubuntu versions below "Groovy" definitely has this issue);
  
  * sudo setcap cap_sys_nice+ep /opt/wine-nspa/bin/wine64-preloader
  * sudo setcap cap_sys_nice+ep /opt/wine-nspa/bin/wine-preloader
  * sudo setcap cap_sys_nice+ep /opt/wine-nspa/bin/wineserver
  
  else;
  
  * find the path yourself! you get the idea ;-)
  
  Additionally, execute the (above) commands; after installation and also updates.
  
  NOTE: This is due to a bug in Jack2 that was fixed in v1.9.13. If using Jack2 v1.9.13/v1.9.14 -> you shouldn't need to 
  run the above commands. -> Therefore, I'd strongly recommend not using old versions of Jack. That said; if you 
  have no choice, then the above workaround exists for you.
  
  that all said, it's possible to get newer Jack2 packages in Ubuntu. Rolesyuk has done just that;
  
  https://github.com/rolesyuk/rt_audio/tree/master/jack
  
  Building Jack2 from source is relatively easy, as well.
    
____
OTHER ENVIRONMMENT VARIABLES / FEATURES:

Wine-NSPA contains some other stuff, as well. 
  
I disable the update window, by default. It can be enabled with; (yup, env variables)
  
  * ENABLE_UPDATE_WINDOW=1
  
  NOTE: You will need to enable this to create a wine-prefix. It will fail to create a prefix, if
        this env variable is set =0. Currently, =0 isn't handled gracefully; but I'll be revising the patch 
        to not even attempt to create a wine-prefix if =0 is set.
  
I also disable the crash dialog... In both cases, I don't like just find them annoying and disruptive. This
also tends to make things feel more integrated, as you don't get random windows dialogs opening, say, after
you've updated wine-nspa, then launch a DAW.

i will also be adding an xembed hack to allow resizing support. same goes with adding suuport for allowing 
32-bit applications to use large address spaces (cross the 4GB RAM boundary). 

i'm also planning to add Dtrace support && looking into the PE tracing code that's floating around for 
the linux kernel... i have patches for Dtrace for wine. Oracle has the linux support. the PE stuff, i haven't
delved into yet; but i'm.aware that it exists... some of this stuff is beyond my skill level to fully utilise,
but it may come in handy, nonetheless.
____
Finally, here is a full example of how i setup my wine env variables;

 - export WINE_RT_PRIO=78
 - export WINE_RT_POLICY="RR"
 - export STAGING_SHARED_MEMORY=1
 - export STAGING_WRITECOPY=1
 - export WINEFSYNC_SPINCOUNT=128
 - export WINESYNC=1
 
 it's also good practice to disable debugging in wine, unless you are actually debugging;
 
 - export WINEDEBUG=-all

these could be put into .bashrc, or made into a shell script - for launching a specific app, etc. 

If I also happen to be using different wine prefixes;

 - export WINEPREFIX=/path/to/wineprefix
____
OTHER NOTES/THOUGHTS:  

This build should make winelib / bridge and VST people happy. Not sure about the 
wine/non-native DAW scenario, beyond testing thread priorities in Reaper + running Kontakt/Reaktor.
Unfortunately, non-native DAWs are more complex and the issues are harder to solve.
  
We can't handle wait-on-multiple-objects in wine, due to wineserver 
being single-threaded and a limitation of hybrid synchronization. It's possible
a DAW in wine might might run better - it's possible it's performance may be hurt, or there could be
threading issues... I'm more interested in improving things for the VST bridge people, and individuals
like myself - who are playing live (with many heavy VSTi's at once, often layered).

Wine-NSPA certainly improves things for me, in my use (no question. it's proving to work well). However, 
ideally we want further tweaks and optimizations. For example, the legacy set-realtime-without-wineserver.patch 
could to be ported  or re-implemented using futex-multiple-wait / fsync. The arguably most important detail 
in that patch is how it was not only able to set realtime priorites - but actually move threads completely 
out of wineserver, meaning they did not wait on user threads. These threads were tagged/hooked in 
SetThreadPriority() (WinAPI), then caught in ntdll and handled differently. 

(just look at the functions in the patch's ntdll and kernel32 parts).
  
I'm probably not smart enough to re-implement this, but I'm stubborn -- so we shall see. lol regardless, I think
that I am correct that a re-implementation of this patch, or some optimization like it; would be killer -- as 
we could hook these threads, which would get rid of a lot of traffic in wineserver...

----
Sometime in the not-so-distant-future, I will also be releases a tool called "Winebox". It's a set of
shell scripts for Managing (pseudo) wine-prefixes. However, It goes beyond just managing wine-prefixes. 
What it actually does is allow having Shared User Data is Wine (across prefixes).

Rather than needing to have multiple prefixes, for different software that may need specific dll overrides
ro registry settings, etc -- instead, you install all software into a single prefix. Then use Winebox to
create psuedo wine-prefixes from the actual wine-prefix. This basically means a combination of copying 
and symlinking, in a clever way -- that makes each psuedo wine-prefixes unique, while also making it 
impossible to corrupt the real wine-prefix... It also works around "Singleton Apps" and their related
lock files. (GuitarRig being one example).

I'm in the process of reworking and updating my Winebox scripts, after a few years of having not
used them. But once I get them sorted out -- I believe other people will find Winebox to be extremely 
useful. I'll provide a git repo, documentation and links, when it's ready.

*lots more TODO/WIP.
