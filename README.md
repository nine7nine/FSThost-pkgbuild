# pkgbuilds_nspa

Git Repo for my wine / linux flavours for Archlinux

  Linux-NSPA; my custom kernel. 

- contains futex-multiple-wait patch(actually, 2 versions). needed for wine+fsync.
- some of intel's patchwork. 
- gcc optimization
- the percpu-rwsem rewrite (lives in -rt 5.6.x, as well).
- swait stuff from -rt. (not really -rt specifc though. in either case.).
- a couple fixes for pesky issues (for me).

set to 1000hz. I use threaded interrupts. etc. not using preempt-rt right now.

mainly posting for backup, but also for anyone wanting to try my wine-nspa builds, will need
kernel support. so the futex-multiple-wait support is available. 5.2.x/5.5 diff version

i also only 'loosely' follow Arch kernel updates. I don't need to chase updates.

  Wine-NSPA; wine geared for proaudio. (*WIP*, like pre-alpha).
  
- Built on Wine-Staging
- Fsync support aka: hybrid synchronization in Wine. (reuires kernel support).

  Enabled With:
 
  * WINEFSYNC_SPINCOUNT=128
  * WINESYNC=1 (0 - disables).

- Initial implmentation of PROCESS_PRIOCLASS_* combined with implementing; 
  PROCESS_PRIOCLASS_REALTIME thread prorities mapped to SCHED_FIFO.

  Example:
  
  * WINESERVER_RT_PRIO=72
  * WINE_RT_PRIO=68
  
  I'm going to shink it down to just WINE_RT_PRIO soon. There's no need to allow
  the user to decide this.
  
  note the space of about 4-6 prio between WINE_RT_* and WINSERVER prios. that's important.
  WINE_RT_PRIO isn't a max - a couple of threads will be (slightly) higher, so will the APC and thus
  wineserver must be higher still. Regardless, it's intended to keep them close / not much of a 
  spread within the system. (the above example is perfect, regardless of priority base level.

- I also use these staging settings.
  
  * STAGING_SHARED_MEMORY=1
  * STAGING_WRITECOPY=1

  Wine-NSPA contains some other stuff, as well.
  
  this build should make winelib / bridge and VST people happy. Not sure about the 
  DAW scenario, beyond testing thread priorities in Reaper + running Kontakt/Reaktor.
  Unfortunately, DAWs are more complex and the issues are harder to solve. 
  
  We can't handle wait-on-multiple-objects in wine, due to wineserver 
  being single-threaded and a limitation of hybrid synchronization. It's possible
  a DAW might run better - it's possible it's perfomrance may be hurt, or there could be
  threading issues. 
  
  lots more TODO/WIP.
