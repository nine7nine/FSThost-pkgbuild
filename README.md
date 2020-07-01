# pkgbuilds_nspa

Git Repo for my wine-nspa / linux-nspa flavours for Archlinux
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
# Wine-NSPA; wine geared for proaudio. (*WIP*, like alpha)

this patchwork and hacking is all geared towards improving prouadio / VST support. Because of that, one
of the central focuses is improving RT/threading support and leveraging fsync to improve synchronization.
These issues tend to be the biggest problems with running proaudio software in Wine.

I'm also willing to accept some hacks or workarounds that may allow specifc VSTs to work, as long as
they don't break things in Wine-NSPA.. and I will implement other useful stuff, if/when it happens.
(I'm sitting on some patchwork that I haven't added yet, but will eventually).

_____  
WINE-STAGING: 

Built on Wine-Staging (5.9 is my base, for now)
_____
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
  
 - WINE_RT_PRIO/SCHED_FIFO(forced) = Wineserver, Kernel-mode APC && RT prioclass THREAD_PRIORITY_TIME_CRITCAL threads.
 - WINE_RT_PRIO (- 1 priority) = any PROCESS_PRIOOCLASS_REALTIME threads, just below THREAD_PRIORITY_TIME_CRITICAL threads.
 - SCHED_OTHER = everything else

This ensures that the kernel-mode APC can preempt user APCs. it also ensures that any critical threads in 
an app will have the same priority as those in Wine-NSPA's core / Wineserver. -- while also making sure that in both 
cases; they can preempt the less important RT prioclass threads in apps... Synchronization is another important 
consideration, here.

Another difference; WINE_RT_PRIO is a MAX value and decrements; this may be personal taste, but it's a
strong preference. I prefer to set the MAX priority level for wine. Then the less important RT threads are 
simply the WINE_RT_PRIO - 1 priority..
  
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

by default, sched_rr_timeslice_ms = 100 (on my 1000hz kernel). I haven't played with the timeslice
too much, but it's possible there might be some benefit to tuning the timeslice/quantum.

NOTE: Jack related threads (from a VST) are still SCHED_FIFO. changing the WINE_RT_POLICY, 
only affects Wine's internal RT threads and our apps/VST's RT threads. -- just in case that isn't obvious.
____
  oh yeah, we can also handle niceness, but it requires running...
  
  * sudo setcap cap_sys_nice+ep /usr/bin/wineserver
  
  ...after installation or updates.
____
OTHER ENVIRONMMENT VARIABLES / FEATURES:

I also use these staging settings. (again, wine environment variables)
  
  * STAGING_SHARED_MEMORY=1
  * STAGING_WRITECOPY=1

Wine-NSPA contains some other stuff, as well. 
  
I disable the update window, by default. It can be enabled with; (yup, env variables)
  
  * ENABLE_UPDATE_WINDOW=1
  
I also disable the crash dialog... In both cases, I don't like just find them annoying and disruptive. This
also tends to make things feel more integrated, as you don't get random windows dialogs opening, say, after
you've updated wine-nspa, then launch a DAW.

Finally, here is a full example of how i setup my wine env variables;

 - export WINE_RT_PRIO=78
 - export WINE_RT_POLICY="RR"
 - export STAGING_SHARED_MEMORY=1
 - export STAGING_WRITECOPY=1
 - export WINEFSYNC_SPINCOUNT=128
 - export WINESYNC=1

these could be put into .bashrc, or made into a shell script - for launching a specific app, etc. If I
also happen to be using different wine prefixes;

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

Anyway, that's all for now.

*lots more TODO/WIP.
