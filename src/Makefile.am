lib_LTLIBRARIES         = libiperf.la                                   # Build and install an iperf library
bin_PROGRAMS            = iperf3                                        # Build and install an iperf binary
if ENABLE_PROFILING
noinst_PROGRAMS         = t_timer t_units t_uuid t_api t_auth iperf3_profile   # Build, but don't install the test programs and a profiled version of iperf3
else
noinst_PROGRAMS         = t_timer t_units t_uuid t_api t_auth                  # Build, but don't install the test programs
endif
include_HEADERS         = iperf_api.h                                   # Defines the headers that get installed with the program

AM_CFLAGS = -D_GNU_SOURCE
AM_CPPFLAGS = -I$(top_srcdir)/src

AM_CFLAGS += $(TCPX_CFLAGS) $(CUDA_CFLAGS)
AM_LDFLAGS = $(CUDA_LDFLAGS)

# Specify the source files and flags for the iperf library
libiperf_la_SOURCES     = \
                        cjson.c \
                        cjson.h \
                        flowlabel.h \
                        iperf.h \
                        iperf_api.c \
                        iperf_api.h \
                        iperf_error.c \
                        iperf_auth.h \
                        iperf_auth.c \
                        iperf_client_api.c \
                        iperf_locale.c \
                        iperf_locale.h \
                        iperf_server_api.c \
                        iperf_tcp.c \
                        iperf_tcp.h \
                        iperf_udp.c \
                        iperf_udp.h \
                        iperf_sctp.c \
                        iperf_sctp.h \
                        iperf_util.c \
                        iperf_util.h \
                        iperf_time.c \
                        iperf_time.h \
                        iperf_pthread.c \
                        iperf_pthread.h \
			dscp.c \
                        net.c \
                        net.h \
                        portable_endian.h \
                        queue.h \
                        tcp_info.c \
                        timer.c \
                        timer.h \
                        units.c \
                        units.h \
                        version.h

# Link libiperf with libtcpx and CUDA
libiperf_la_LIBADD      = $(TCPX_LIBS) $(CUDA_LIBS)

# Specify the sources and various flags for the iperf binary
iperf3_SOURCES          = main.c
iperf3_CFLAGS           = -g
iperf3_LDADD            = libiperf.la $(TCPX_LIBS) $(CUDA_LIBS)
iperf3_LDFLAGS          = -g $(CUDA_LDFLAGS)

if ENABLE_PROFILING
# If the iperf-profiled-binary is enabled
# Specify the sources and various flags for the profiled iperf binary. This
# binary recompiles all the source files to make sure they are all profiled.
iperf3_profile_SOURCES  = main.c \
                          $(libiperf_la_SOURCES)

iperf3_profile_CFLAGS   = -pg -g
iperf3_profile_LDADD    = libiperf.la $(TCPX_LIBS) $(CUDA_LIBS)
iperf3_profile_LDFLAGS  = -pg -g $(CUDA_LDFLAGS)
endif

# Specify the sources and various flags for the test cases
t_timer_SOURCES         = t_timer.c
t_timer_CFLAGS          = -g
t_timer_LDFLAGS         = $(CUDA_LDFLAGS)
t_timer_LDADD           = libiperf.la $(TCPX_LIBS) $(CUDA_LIBS)

t_units_SOURCES         = t_units.c
t_units_CFLAGS          = -g
t_units_LDFLAGS         = $(CUDA_LDFLAGS)
t_units_LDADD           = libiperf.la $(TCPX_LIBS) $(CUDA_LIBS)

t_uuid_SOURCES          = t_uuid.c
t_uuid_CFLAGS           = -g
t_uuid_LDFLAGS          = $(CUDA_LDFLAGS)
t_uuid_LDADD            = libiperf.la $(TCPX_LIBS) $(CUDA_LIBS)

t_api_SOURCES           = t_api.c
t_api_CFLAGS            = -g
t_api_LDFLAGS           = $(CUDA_LDFLAGS)
t_api_LDADD             = libiperf.la $(TCPX_LIBS) $(CUDA_LIBS)

t_auth_SOURCES           = t_auth.c
t_auth_CFLAGS            = -g
t_auth_LDFLAGS           = $(CUDA_LDFLAGS)
t_auth_LDADD             = libiperf.la $(TCPX_LIBS) $(CUDA_LIBS)



# Specify which tests to run during a "make check"
TESTS                   = \
                        t_timer \
                        t_units \
                        t_uuid  \
                        t_api \
			t_auth

dist_man_MANS          = iperf3.1 libiperf.3
