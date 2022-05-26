# FSThost

Git Repo for FSThost and old classic Windows VST plugin host for linux. this codebase is a little rough 
these days and a bit out of date, but it builds / runs.

FSThost was the first Winelib VST host to run 64bit plugins and it was the first to also make use of 
synchronization related wine patches (the old muse receptor wine patches). Performance was fairly 
impressive, back in the day.

the software is no longer developed. but it could be picked up and worked on; possibly strip out all of 
the perl code and session managamenet stuff, turn it into a clean single VST host. work on performance 
related features and VST3 support (it can load some vst3).
