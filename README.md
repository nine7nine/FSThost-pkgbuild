# pkgbuilds_nspa

Git Repo for my wine / linux flavours for Archlinux

 # Linux-NSPA; my custom kernel.

Nothing magical here, just some patchwork that I use on my Laptop / on Arch. semi-focus
on low-latency, some optimizations and improvements -- but the main reason for it's existence is
for futex-multiple-wait for wine-nspa. 

- contains futex-multiple-wait patch(actually, 2 versions). needed for wine+fsync.
- some of intel's patchwork. 
- gcc optimization
- the percpu-rwsem rewrite (lives in -rt 5.6.x, as well).
- swait stuff from -rt. (not really -rt specifc though. in either case.).
- a couple fixes for pesky issues (for me).

set to 1000hz. I use threaded interrupts. etc. not using preempt-rt right now.

mainly posting for backup. However, it's also here for anyone wanting to try my wine-nspa builds; 
as you will need kernel support. so the futex-multiple-wait support is available. 5.2.x/5.5 diff version

i also only 'loosely' follow Arch kernel updates. I don't need to chase updates and i need a consistent
system for testing changes - rolling can introduce regressions or other issues, so I'm sticking with 
5.6.x for a short-while. but I will update, obviously.

# Wine-NSPA; wine geared for proaudio. (*WIP*, like alpha)

this patchwork and hacking is all cetnered around improved prouadio / VST support. Because of that, one
central focus is improving RT/threading support and leveraging fsync to improve synchronization. Other 
useful stuff, if/when it happens... 
  
- Built on Wine-Staging (5.9 is my base, for now)
- Fsync support aka: hybrid synchronization in Wine. (requires kernel support).
 
  Enabled With: (wine environment variables)
 
  * WINEFSYNC_SPINCOUNT=128
  * WINESYNC=1 (0 - disables).

- Initial implmentation of Windows' PROCESS_PRIOCLASS_* (process priority classes) combined with having
  the ROCESS_PRIOCLASS_REALTIME thread prorities mapped to SCHED_FIFO or SCHED_RR. This allows far 
  better prioritization of threads. 

  Example: (wine environment variables)
  
  * WINE_RT_POLICY=RR (SCHED_RR) or FF (SCHED_FIFO) 
  * WINE_RT_PRIO=75
  
  Both of these must be set. 
  
  unlike wine-staging - I don't allow setting the wineserver thread, independently. Instead, we want this;
  
  * #1 wine-nspa's most important/highest priority threads to all be prioritized the exact same. 
  * #2 the applications most critical threads (TIME_CRITICAL) at the same priority as #1, while 
  * #3 the apps/VSTs other realtime threads are set just below.
  
  Another difference; WINE_RT_PRIO is a MAX value and decrements; this may be personal taste, but it's a
  strong preference. I prefer to set the highest possible priority level for wine -- and I know it's lower
  RT threads, are just below.
  
  NOTE: I actually do set WINE_RT_PRIO very high (78) on my machine. just below Jack. for DAWs, you might 
  have to be careful how high you set WINE_RT_PRIO, as you don't want to interfere with your DAW's most
  important threads (really though, these will probably be Jack audio threads).
  
  Lastly, we allow changing the RT policy, as mentioned. I'm not convinced that SCHED_FIFO is the best fit
  for wine. I'm currently testing both, so adding the needed bits is appropriate. 
  
  we can also handle niceness, but it requires running...
  
  * sudo setcap cap_sys_nice+ep /usr/bin/wineserver
  
  ...after installation or updates.
  
  NOTE: while wine-nspa certainly improves things for me. ideally, the legacy set-realtime-without-wineserver.patch needs
  to be ported / re-implemented using futex-multiple-wait. The arguably most important detail in that patch is
  how it was not only able to set realtime priorites - but actually move threads completely out of wineserver,
  meaning they did not wait on user threads. These threads were tagged/hooked in SetThreadPriority() (WinAPI),
  then caught in ntdll and handled differently. (just look at the functions in the patch's ntdll and kernel32 parts).
  
  I'm probably not smart enough to re-implement this, but I'm stubborn -- so we shall see. lol regardless, I am
  correct that a re-implmenetation of this patch would be killer -- as we could hook these threads, which would
  get rid of a lot of traffic in wineserver...

- I also use these staging settings. (again, wine environment variables)
  
  * STAGING_SHARED_MEMORY=1
  * STAGING_WRITECOPY=1

  Wine-NSPA contains some other stuff, as well. 
  
  I disable the update window, by default. It can be enabled with; (yup, env variables)
  
  * ENABLE_UPDATE_WINDOW=1
  
  I also disable the crash dialog... In both cases, I just find them annoying and disruptive.
  
  the other patches are hacks or workarounds. useful though.
    
  this build should make winelib / bridge and VST people happy. Not sure about the 
  wine/non-native DAW scenario, beyond testing thread priorities in Reaper + running Kontakt/Reaktor.
  Unfortunately, non-native DAWs are more complex and the issues are harder to solve.
  
  We can't handle wait-on-multiple-objects in wine, due to wineserver 
  being single-threaded and a limitation of hybrid synchronization. It's possible
  a DAW in wine might might run better - it's possible it's performance may be hurt, or there could be
  threading issues... I'm more interested in improving things for the VST bridge people, and individuals
  like myself - who are playing live (with many heavy VSTi's at once, often layered).
  
  lots more TODO/WIP.
