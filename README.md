# Wine-NSPA + Linux-NSPA

Git Repo for my wine-nspa / linux-nspa flavours for Archlinux.

That said, nothing makes this exclusive to Arch. For example, Rolesyuk has kindly
packaged binaries of Wine-NSPA for Ubuntu 20.04 + He has binaries for a PREEMPT_RT kernel with
futex-multiple-wait to support fsync in Wine-NSPA. 

https://github.com/rolesyuk/rt_audio/releases

Furthermore, he has scripts to build these packages + an automated script to grab jack2 1.9.14 deb on 20.04;

https://github.com/rolesyuk/rt_audio

_______
 # Linux-NSPA; my custom kernel.

Nothing magical here, just some patchwork that I use on my Laptop / on Arch. focused on 
low-latency, some optimizations and improvements -- but the main reason for it's existence is
for futex-multiple-wait for wine-nspa. 

- contains futex-multiple-wait patch(actually, 2 versions).
- some of intel's patchwork. 
- gcc optimization
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
system for testing changes - rolling can introduce regressions or other issues. So i tend to bump the kernel
packages as i see fit / after some testing.
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

Write copy should reduce memory usage, as a single dll will be loaded into memory, which applications
share. If an app tries to modify them, then it will be copy-on-write... This could possibly introduce
bugs - as some of the gamers hit bugs with it - but I have yet to... setting it to =0 disables it.

____
FSYNC: 

Fsync support aka: hybrid synchronization in Wine. (requires kernel support).
 
  Enabled With: (wine environment variables)
  * WINEFSYNC_SPINCOUNT=128
  * WINESYNC=1 (0 - disables).
  
NOTE: Technically, I support EventFD Synchronization (Esync) too. But i do not use or test it anymore.
      Fsync seems to be the better choice. with Esync you can exhaust a system's open file limit (rlimit)
      and i've read about VSTs that have done just that.
_____
WINE-NSPA's RT SUPPORT: 

Initial implementation of Windows' PROCESS_PRIOCLASS_* (process priority classes), 
combined with PROCESS_PRIOCLASS_REALTIME thread prorities mapped to SCHED_FIFO or SCHED_RR. This allows 
far better prioritization of threads. 

  Enabled with: (wine environment variables)
  * WINE_RT_POLICY=RR (SCHED_RR) or FF (SCHED_FIFO) 
  * WINE_RT_PRIO=75
  
Both of these env variables MUST be set. 

IMPORTANT NOTE: Wine-NSPA v5.9.18+ (v2.0 RT support)

Below isn't entirely accurate. In 5.9.18+, currently I'm forcing the policy/priority of some the high  
priority RT threads. I'm leaving below intact, as it's mostly relevant. just some of the details about what's
happeneing in Wine are slightly different. (below is < 5.9.17 implementation)... but I will likely bring
back switching RT policy, for select threads, once i get some other bits implemented and can allow it again.

in NSPA RT v2.0, even if you set RR it'll be FF for now. Other than that, basic usage is the same.  

____
WINE-NSPA's THREAD PRIORITIES & DESIGN:
  
Unlike the wine-staging RT patch, I don't allow setting the wineserver thread priority (independently). I'm doing 
this for good reason. more on that below... Instead, we want this priority/thread placement;
  
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
  
  If using Rolesyuk's Ubuntu packages, with Jack 1.9.12 or under;
  
  * sudo setcap cap_sys_nice+ep /opt/wine-nspa/bin/wine64-preloader
  * sudo setcap cap_sys_nice+ep /opt/wine-nspa/bin/wine-preloader
  * sudo setcap cap_sys_nice+ep /opt/wine-nspa/bin/wineserver
  
  else;
  
  * find the path yourself! you get the idea ;-)
  
  Additionally, execute the (above) commands; after installation and also updates.
  
  NOTE: This is due to a bug in Jack2 that was fixed in v1.9.13. If using Jack2 v1.9.13/v1.9.14 -> you shouldn't need to 
  run the above commands. -> Therefore, I'd strongly recommend not using old versions of Jack. That said; if you 
  have no choice, then the above workaround exists for you.
  
  that all said, it's possible to get newer Jack2 packages in Ubuntu. Rolesyuk has done just that, with an automated
  script that grabs and replaces the older ubuntu jack2 packages, with new ones;
  
  https://github.com/rolesyuk/rt_audio/tree/master/jack
  
  Building Jack2 from source is relatively easy, as well.
    
____
OTHER FEATURES:

Wine-NSPA contains some other stuff, as well. 
  
  - I disable the wine's update window completely. 
  
  - I also disable the crash dialog... In both cases, I don't like just find them annoying and disruptive. This
    also tends to make things feel more integrated, as you don't get random windows dialogs opening, say, after
    you've updated wine-nspa, then launch a DAW.

  - I've added LFH support (Low Fragmentation Hep) with a Thread Local Implementation (similar kind of thing to tcmalloc). 
    This is a bit experimental and is missing a few bits, from what i gather. However, I haven't noticed any regressions
    and with what testing i have done is does seem to be beneficial. 
    
    this is enabled globally, but can be disabled with;
    
    - export WINELFH=0
    
  - Implementation of Get/SetProcessWorkingSetSize() mapped to RLIMIT_MEMLOCK... This is enabled in v5.9.19+.
  
    You likely won't bump into this often in VSTs, but some do make these calls.
    
    Much like how one needs to setup resource limits to give JACK / your user realtime privileges, you can specify the
    memlock and other rlimits too. This can done via DefaultLimitMEMLOCK= in systemd for your user. 
    
    /etc/systemd/user.conf.d/user_limits.conf
    
    if it doesn't exist, you can create it. here is what an example config looks like;
___   
 - DefaultLimitRTPRIO=98
 - DefaultLimitNOFILE=500000
 - DefaultLimitNPROC=500000
 - DefaultLimitSIGPENDING=286,816
 - DefaultLimitMEMLOCK=unlimited 
 - DefaultLimitLOCKS=unlimited
 - DefaultLimitNICE=unlimited
___

   in my case, I have other maximum (or infinite) values for other resources as well. RT priorities, nice values, 
   number of open files, etc.  But as you can see; i have DefaultlimitMEMLOCK=unlimited set. Distributions should
   provide info on how to configure / learn about the various settings, so I'm not going to cover that here.
   
   a logout/in is required for setting changes to take effect. You can verify changes via; ulimit -a
   
  - fixes for NI Access / newer NI (sub)installers, one-offs and hacks for different programs. 

Another feature on the horizon will be trying to filter RT threads better, as well as moving certain threads completely
out of wineserver (this will take a bit to sort out).    

i'm also planning to add Dtrace support && looking into the PE tracing code that's floating around for 
the linux kernel... i have patches for Dtrace for wine. Oracle has the linux support. the PE stuff, i haven't
delved into yet; but i'm aware that it exists... some of this stuff is beyond my skill level to fully utilise,
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

this could be put into made into a shell script - for launching a specific app, etc. 

If I also happen to be using different wine prefixes;

 - export WINEPREFIX=/path/to/wineprefix
 
___
WARNING/IMPORTANT:

I currently do not support Ilok Manager, as it will cause issues with threading. In fact, it's best to
avoid applications that like to try and add their own services and/or (unsupported) drivers... They tend
to cause problems and generally stuff like that also tends to have problems in Wine anyway.

Another important thing; Wine (any version) will cause some xruns, usually on app launch - sometimes
with heavy disk i/o. (loading large sample libraries, some session or project with a bunch of VSTs). This 
is pretty common, I'm not sure of the best solution to this - it's possible playing some games with
thread prioritization could reduce this, but it's tricky to figure out.

I've limited the scope of what threads are allowed to be RT, to help minimize xruns and other undesired
behaviour. It's a tricky balance... Eventually, when i get the next iteration of RT support implemented, I
should be able to support things like iLok Manager and other services that may cause issues now. 

All in good time.

____
OTHER NOTES/THOUGHTS:  

This build should make winelib / bridge and VST people happy. Not sure about the 
wine/non-native DAW scenario, beyond testing thread priorities in Reaper + running Kontakt/Reaktor. 
Non-native DAWs are more complex and the issues are harder to solve. That said, Reaper actually runs 
very well (has for years in wine), as do some other VST hosts.
  
Unfortunately, one challenge with wine is that we can't handle wait-on-multiple-objects in wine, due to wineserver 
being single-threaded and a limitation of hybrid synchronization. It's possible a DAW in wine might might run better, 
it's possible it's performance may be hurt, or there could be threading issues...

Wine-NSPA certainly improves things for me, in my use (no question. it's proving to work well - especially as
I'm progressing with my patchwork / features). But i should also note, all of my patchwork is subject to
change from build to build. Sometimes, i may have regressions - as I'm implementating different things, or
trying different schemes - especially with the RT support. (which is definitely WIP).

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
