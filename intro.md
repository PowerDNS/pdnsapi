Everything open for discussion.

TODO:

  * All the italic stuff
  * All the DECIDE, FIXME, XXX, TBD stuff
  * SHOULD, and so on.
  * Finish data management (tsigkeys, …)

Big Picture
===========

* HTTP with SSL in-process in Auth & Recursor
* JSON API
  * make it really great for us and other consumers
  * “unified” API across Daemons and Console
* pdnsmgrd
  * cease to do SSL proxying
  * become completely optional component
  * only for “meta” features
* Console
  * get rid of all the API hacks
  * new features as detailed below
* CLI tool
  * should talk to daemons and Console (if there)
* “Pure” OOTB install
  * miniature single page js app for users not installing pdnscontrol

“Secondary” goals
=================

* keep everything lean
* minimal intrusions into existing code
