include $(top_srcdir)/build/modmake.rulesam
MODNAME=mod_callstats

mod_LTLIBRARIES = mod_callstats.la
mod_callstats_la_SOURCES  = mod_callstats.c
mod_callstats_la_CFLAGS   = $(AM_CFLAGS)
mod_callstats_la_LIBADD   = $(switch_builddir)/libfreeswitch.la
mod_callstats_la_LDFLAGS  = -avoid-version -module -no-undefined -shared
