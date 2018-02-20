cmake_minimum_required(VERSION 2.8.6)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/Modules)

project(tcpdump)

#
# Try to enable as many C99 features as we can.
# At minimum, we want C++/C99-style // comments.
# (Sadly, this won't work with CMake prior to 3.1.)
#
set(CMAKE_C_STANDARD 99)

set(LIBRARY_NAME netdissect)

###################################################################
#   Parameters
###################################################################

#option(WITH_SMI "Build with libsmi, if available" ON)
#option(WITH_CRYPTO "Build with OpenSSL/libressl libcrypto, if available" ON)
#option(WITH_CAPSICUM "Build with Capsicum security functions, if available" ON)
#option(WITH_CAP_NG "Use libcap-ng, if available" ON)
#option(ENABLE_SMB "Build with the SMB dissector" OFF)

#
# String parameters.  Neither of them are set, initially; only if the
# user explicitly configures them are they set.
#
# WITH_CHROOT is STRING, not PATH, as the directory need not exist
# when CMake is run.
#
set(WITH_CHROOT CACHE STRING
    "Directory to which to chroot when dropping privileges")
set(WITH_USER CACHE STRING
    "User to whom to set the UID when dropping privileges")

######################################
# Project settings
######################################

add_definitions(-DHAVE_CONFIG_H)

include_directories(
    ${CMAKE_CURRENT_BINARY_DIR}
    ${tcpdump_SOURCE_DIR}
)

###################################################################
#   Detect available platform features
###################################################################

include(CMakePushCheckState)
include(CheckIncludeFile)
include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckLibraryExists)
include(CheckSymbolExists)
include(CheckStructHasMember)
include(CheckVariableExists)
include(CheckTypeSize)

#
# Header files.
#
check_include_file(fcntl.h HAVE_FCNTL_H)
if(NOT WIN32)
    check_include_files("sys/types.h;sys/socket.h;net/if.h;net/pfvar.h" HAVE_NET_PFVAR_H)
    if(HAVE_NET_PFVAR_H)
        check_include_files("sys/types.h;sys/socket.h;net/if.h;net/pfvar.h;net/if_pflog.h" HAVE_NET_IF_PFLOG_H)
        if(HAVE_NET_IF_PFLOG_H)
            set(LOCALSRC print-pflog.c ${LOCALSRC})
        endif(HAVE_NET_IF_PFLOG_H)
    endif(HAVE_NET_PFVAR_H)
endif(NOT WIN32)

#
# Functions.
#
check_function_exists(strlcat HAVE_STRLCAT)
check_function_exists(strlcpy HAVE_STRLCPY)
check_function_exists(strdup HAVE_STRDUP)
check_function_exists(strsep HAVE_STRSEP)

#
# Find library needed for gethostbyaddr.
# NOTE: if you hand check_library_exists as its last argument a variable
# that's been set, it skips the test, so we need different variables.
#
set(TCPDUMP_LINK_LIBRARIES "")

check_function_exists(gethostbyaddr STDLIBS_HAVE_GETHOSTBYADDR)
if(NOT STDLIBS_HAVE_GETHOSTBYADDR)
    check_library_exists(socket gethostbyaddr "" LIBSOCKET_HAS_GETHOSTBYADDR)
    if(LIBSOCKET_HAS_GETHOSTBYADDR)
        set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} socket)
    else(LIBSOCKET_HAS_GETHOSTBYADDR)
        check_library_exists(nsl gethostbyaddr "" LIBNSL_HAS_GETHOSTBYADDR)
        if(LIBNSL_HAS_GETHOSTBYADDR)
            set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} nsl)
        else(LIBNSL_HAS_GETHOSTBYADDR)
            message(FATAL_ERROR "gethostbyaddr is required, but wasn't found")
        endif(LIBNSL_HAS_GETHOSTBYADDR)
    endif(LIBSOCKET_HAS_GETHOSTBYADDR)
endif(NOT STDLIBS_HAVE_GETHOSTBYADDR)


#
# This may require additional libraries.
#
cmake_push_check_state()
set(CMAKE_REQUIRED_LIBRARIES ${TCPDUMP_LINK_LIBRARIES})
check_function_exists(getservent STDLIBS_HAVE_GETSERVENT)
if(STDLIBS_HAVE_GETSERVENT)
    set(HAVE_GETSERVENT TRUE)
else(STDLIBS_HAVE_GETSERVENT)
    #
    # Some platforms may need -lsocket for getservent.
    #
    set(CMAKE_REQUIRED_LIBRARIES socket ${TCPDUMP_LINK_LIBRARIES})
    check_function_exists(getservent LIBSOCKET_HAS_GETSERVENT)
    if(LIBSOCKET_HAS_GETSERVENT)
        set(HAVE_GETSERVENT TRUE)
        set(TCPDUMP_LINK_LIBRARIES socket ${TCPDUMP_LINK_LIBRARIES})
    endif(LIBSOCKET_HAS_GETSERVENT)
endif(STDLIBS_HAVE_GETSERVENT)
cmake_pop_check_state()

check_function_exists(getopt_long HAVE_GETOPT_LONG)
#
# For Windows, either
#
#   1) we're using VS 2015, in which case we have both snprintf()
#      and vsnprintf(), and they behave in a C99-compliant fashion,
#      so we use them
#
# or
#
#   2) we're not, and we don't have snprintf(), and we either don't
#      have vsnprintf() or we have one that *doesn't* behave in a
#      C99-compliant fashion, but we *do* have _snprintf_s() and
#      _vsnprintf_s(), so we wrap them with #defines
#
# and we test for both of them at compile time, so we don't need to
# check for snprintf() or vsnprintf() here.
#
# XXX - do we need to care about UN*Xes that don't have snprintf()
# or vsnprintf() any more?
#
# We also don't need to waste time checking for fork() or vfork().
#
if(NOT WIN32)
  check_function_exists(vsnprintf HAVE_VSNPRINTF)
  check_function_exists(snprintf HAVE_SNPRINTF)
  check_function_exists(fork HAVE_FORK)
  check_function_exists(vfork HAVE_VFORK)
endif(NOT WIN32)
check_function_exists(strftime HAVE_STRFTIME)
check_function_exists(setlinebuf HAVE_SETLINEBUF)

#
# Some platforms may need -lnsl for getrpcbynumber.
#
cmake_push_check_state()
set(CMAKE_REQUIRED_LIBRARIES ${TCPDUMP_LINK_LIBRARIES})
check_function_exists(getrpcbynumber STDLIBS_HAVE_GETRPCBYNUMBER)
if(STDLIBS_HAVE_GETRPCBYNUMBER)
    set(HAVE_GETRPCBYNUMBER TRUE)
else(STDLIBS_HAVE_GETRPCBYNUMBER)
    set(CMAKE_REQUIRED_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} nsl)
    check_function_exists(getrpcbynumber LIBNSL_HAS_GETRPCBYNUMBER)
    if(LIBNSL_HAS_GETRPCBYNUMBER)
        set(HAVE_GETRPCBYNUMBER TRUE)
        set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES} nsl)
    endif(LIBNSL_HAS_GETRPCBYNUMBER)
endif(STDLIBS_HAVE_GETRPCBYNUMBER)
cmake_pop_check_state()

#
# This requires the libraries we require, as ether_ntohost might be
# in one of those libraries.  That means we have to do this after
# we check for those libraries.
#
# You are in a twisty little maze of UN*Xes, all different.
# Some might not have ether_ntohost().
# Some might have it and declare it in <net/ethernet.h>.
# Some might have it and declare it in <netinet/ether.h>
# Some might have it and declare it in <sys/ethernet.h>.
# Some might have it and declare it in <arpa/inet.h>.
# Some might have it and declare it in <netinet/if_ether.h>.
# Some might have it and not declare it in any header file.
#
# Before you is a C compiler.
#
cmake_push_check_state()
set(CMAKE_REQUIRED_LIBRARIES ${TCPDUMP_LINK_LIBRARIES})
check_function_exists(ether_ntohost HAVE_ETHER_NTOHOST)
if(HAVE_ETHER_NTOHOST)
    #
    # OK, we have ether_ntohost().  We don't check whether it's buggy,
    # as we assume any system that has CMake is likely to be new enough
    # that, if it has ether_ntohost(), whatever bug is checked for in
    # autotools is fixed; we just decide to use it.
    #
    set(USE_ETHER_NTOHOST TRUE)

    #
    # Is it declared in <net/ethernet.h>?
    #
    # This test fails if we don't have <net/ethernet.h> or if we do
    # but it doesn't declare ether_ntohost().
    #
    check_symbol_exists(ether_ntohost net/ethernet.h NET_ETHERNET_H_DECLARES_ETHER_NTOHOST)
    if(NET_ETHERNET_H_DECLARES_ETHER_NTOHOST)
        #
        # Yes - we have it declared.
        #
        set(HAVE_DECL_ETHER_NTOHOST TRUE)
    endif()
    #
    # Did that succeed?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No - how about <netinet/ether.h>, as on Linux?
        #
        # This test fails if we don't have <netinet/ether.h>
        # or if we do but it doesn't declare ether_ntohost().
        #
        check_symbol_exists(ether_ntohost netinet/ether.h NETINET_ETHER_H_DECLARES_ETHER_NTOHOST)
        if(NETINET_ETHER_H_DECLARES_ETHER_NTOHOST)
            #
            # Yes - we have it declared.
            #
            set(HAVE_DECL_ETHER_NTOHOST TRUE)
        endif()
    endif()
    #
    # Did that succeed?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No - how about <sys/ethernet.h>, as on Solaris 10 and later?
        #
        # This test fails if we don't have <sys/ethernet.h>
        # or if we do but it doesn't declare ether_ntohost().
        #
        check_symbol_exists(ether_ntohost sys/ethernet.h SYS_ETHERNET_H_DECLARES_ETHER_NTOHOST)
        if(SYS_ETHERNET_H_DECLARES_ETHER_NTOHOST)
            #
            # Yes - we have it declared.
            #
            set(HAVE_DECL_ETHER_NTOHOST TRUE)
        endif()
    endif()
    #
    # Did that succeed?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No, how about <arpa/inet.h>, as on AIX?
        #
        # This test fails if we don't have <arpa/inet.h>
        # or if we do but it doesn't declare ether_ntohost().
        #
        check_symbol_exists(ether_ntohost arpa/inet.h ARPA_INET_H_DECLARES_ETHER_NTOHOST)
        if(ARPA_INET_H_DECLARES_ETHER_NTOHOST)
            #
            # Yes - we have it declared.
            #
            set(HAVE_DECL_ETHER_NTOHOST TRUE)
        endif()
    endif()
    #
    # Did that succeed?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No, how about <netinet/if_ether.h>?
        # On some platforms, it requires <net/if.h> and
        # <netinet/in.h>, and we always include it with
        # both of them, so test it with both of them.
        #
        # This test fails if we don't have <netinet/if_ether.h>
        # and the headers we include before it, or if we do but
        # <netinet/if_ether.h> doesn't declare ether_ntohost().
        #
        check_symbol_exists(ether_ntohost "sys/types.h;sys/socket.h;net/if.h;netinet/in.h;netinet/if_ether.h" NETINET_IF_ETHER_H_DECLARES_ETHER_NTOHOST)
        if(NETINET_IF_ETHER_H_DECLARES_ETHER_NTOHOST)
            #
            # Yes - we have it declared.
            #
            set(HAVE_DECL_ETHER_NTOHOST TRUE)
        endif()
    endif()
    #
    # After all that, is ether_ntohost() declared?
    #
    if(NOT HAVE_DECL_ETHER_NTOHOST)
        #
        # No, we'll have to declare it ourselves.
        # Do we have "struct ether_addr" if we include<netinet/if_ether.h>?
        #
        check_struct_has_member("struct ether_addr" octet "sys/types.h;sys/socket.h;net/if.h;netinet/in.h;netinet/if_ether.h" HAVE_STRUCT_ETHER_ADDR)
    endif()
endif()
cmake_pop_check_state()

check_function_exists(dnet_htoa STDLIBS_HAVE_DNET_HTOA)
if(STDLIBS_HAVE_DNET_HTOA)
    set(HAVE_DNET_HTOA TRUE)
else(STDLIBS_HAVE_DNET_HTOA)
    check_library_exists(dnet dnet_htoa "" HAVE_DNET_HTOA)
    if(HAVE_DNET_HTOA)
        set(TCPDUMP_LINK_LIBRARIES dnet ${TCPDUMP_LINK_LIBRARIES})
    endif(HAVE_DNET_HTOA)
endif(STDLIBS_HAVE_DNET_HTOA)
if(HAVE_DNET_HTOA)
    #
    # OK, we have dnet_htoa().  Do we have netdnet/dnetdb.h?
    #
    check_include_files("sys/types.h;netdnet/dnetdb.h" HAVE_NETDNET_DNETDB_H)
    if(HAVE_NETDNET_DNETDB_H)
        #
        # Yes.  Does it declare dnet_htoa()?
        #
        cmake_push_check_state()
        set(CMAKE_REQUIRED_LIBRARIES dnet)
        check_symbol_exists(dnet_htoa "sys/types.h;netdnet/dnetdb.h" NETDNET_DNETDB_H_DECLARES_DNET_HTOA)
        cmake_pop_check_state()
    endif(HAVE_NETDNET_DNETDB_H)

    #
    # Do we have netdnet/dn.h?
    #
    check_include_file(netdnet/dn.h HAVE_NETDNET_DN_H)
    if(HAVE_NETDNET_DN_H)
        #
        # Yes.  Does it declare struct dn_naddr?
        #
        cmake_push_check_state()
        set(CMAKE_EXTRA_INCLUDE_FILES netdnet/dn.h)
        check_type_size("struct dn_naddr" STRUCT_DN_NADDR)
        cmake_pop_check_state()
    endif(HAVE_NETDNET_DN_H)
endif(HAVE_DNET_HTOA)

#
# Data types.
#
# XXX - there's no check_struct() macro that's like check_struct_has_member()
# except that it only checks for the existence of the structure type,
# so we use check_struct_has_member() and look for ss_family.
#

#
# Check for IPv6 support.
# We just check for AF_INET6 and struct in6_addr.
#
cmake_push_check_state()

set(CMAKE_EXTRA_INCLUDE_FILES sys/types.h sys/socket.h netinet/in.h)
check_symbol_exists(AF_INET6 "sys/types.h;sys/socket.h;netinet/in.h" HAVE_AF_INET6)

check_type_size("struct in6_addr" HAVE_STRUCT_IN6_ADDR)
cmake_pop_check_state()
if(HAVE_AF_INET6 AND HAVE_STRUCT_IN6_ADDR)
    set(HAVE_OS_IPV6_SUPPORT TRUE)
endif(HAVE_AF_INET6 AND HAVE_STRUCT_IN6_ADDR)

######################################
# External dependencies
######################################

set(TCPDUMP_LINK_LIBRARIES ${TCPDUMP_LINK_LIBRARIES})

set(NETDISSECT_SOURCE_LIST_C
    addrtoname.c
    addrtostr.c
    af.c
    ascii_strcasecmp.c
    checksum.c
    cpack.c
    gmpls.c
    gmt2local.c
    in_cksum.c
    ipproto.c
    l2vpn.c
    machdep.c
    nlpid.c
    oui.c
    parsenfsfh.c
    print.c
    print-802_11.c
    print-802_15_4.c
    print-ah.c
    print-ahcp.c
    print-aodv.c
    print-aoe.c
    print-ap1394.c
    print-arcnet.c
    print-arp.c
    print-ascii.c
    print-atalk.c
    print-atm.c
    print-babel.c
    print-beep.c
    print-bfd.c
    print-bgp.c
    print-bootp.c
    print-bt.c
    print-calm-fast.c
    print-carp.c
    print-cdp.c
    print-cfm.c
    print-chdlc.c
    print-cip.c
    print-cnfp.c
    print-dccp.c
    print-decnet.c
    print-dhcp6.c
    print-domain.c
    print-dtp.c
    print-dvmrp.c
    print-eap.c
    print-egp.c
    print-eigrp.c
    print-enc.c
    print-esp.c
    print-ether.c
    print-fddi.c
    print-forces.c
    print-fr.c
    print-frag6.c
    print-ftp.c
    print-geneve.c
    print-geonet.c
    print-gre.c
    print-hncp.c
    print-hsrp.c
    print-http.c
    print-icmp.c
    print-icmp6.c
    print-igmp.c
    print-igrp.c
    print-ip.c
    print-ip6.c
    print-ip6opts.c
    print-ipcomp.c
    print-ipfc.c
    print-ipnet.c
    print-ipx.c
    print-isakmp.c
    print-isoclns.c
    print-juniper.c
    print-krb.c
    print-l2tp.c
    print-lane.c
    print-ldp.c
    print-lisp.c
    print-llc.c
    print-lldp.c
    print-lmp.c
    print-loopback.c
    print-lspping.c
    print-lwapp.c
    print-lwres.c
    print-m3ua.c
    print-medsa.c
    print-mobile.c
    print-mobility.c
    print-mpcp.c
    print-mpls.c
    print-mptcp.c
    print-msdp.c
    print-msnlb.c
    print-nflog.c
    print-nfs.c
    print-nsh.c
    print-ntp.c
    print-null.c
    print-olsr.c
    print-openflow-1.0.c
    print-openflow.c
    print-ospf.c
    print-ospf6.c
    print-otv.c
    print-pgm.c
    print-pim.c
    print-pktap.c
    print-ppi.c
    print-ppp.c
    print-pppoe.c
    print-pptp.c
    print-radius.c
    print-raw.c
    print-resp.c
    print-rip.c
    print-ripng.c
    print-rpki-rtr.c
    print-rrcp.c
    print-rsvp.c
    print-rt6.c
    print-rtsp.c
    print-rx.c
    print-sctp.c
    print-sflow.c
    print-sip.c
    print-sl.c
    print-sll.c
    print-slow.c
    print-smtp.c
    print-snmp.c
    print-stp.c
    print-sunatm.c
    print-sunrpc.c
    print-symantec.c
    print-syslog.c
    print-tcp.c
    print-telnet.c
    print-tftp.c
    print-timed.c
    print-tipc.c
    print-token.c
    print-udld.c
    print-udp.c
    print-usb.c
    print-vjc.c
    print-vqp.c
    print-vrrp.c
    print-vtp.c
    print-vxlan.c
    print-vxlan-gpe.c
    print-wb.c
    print-zephyr.c
    print-zeromq.c
    ${LOCALSRC}
    netdissect.c
    signature.c
    strtoaddr.c
    util-print.c
)

#
# Replace missing functions
#
foreach(FUNC strlcat strlcpy strdup strsep getservent getopt_long)
    string(TOUPPER ${FUNC} FUNC_UPPERCASE)
    set(HAVE_FUNC_UPPERCASE HAVE_${FUNC_UPPERCASE})
    if(NOT ${HAVE_FUNC_UPPERCASE})
        set(NETDISSECT_SOURCE_LIST_C ${NETDISSECT_SOURCE_LIST_C} missing/${FUNC}.c)
    endif()
endforeach()
if(NOT WIN32)
    if(NOT HAVE_VSNPRINTF OR NOT HAVE_SNPRINTF)
        set(NETDISSECT_SOURCE_LIST_C ${NETDISSECT_SOURCE_LIST_C} missing/snprintf.c)
    endif(NOT HAVE_VSNPRINTF OR NOT HAVE_SNPRINTF)
endif(NOT WIN32)

#add_library(netdissect STATIC
#    ${NETDISSECT_SOURCE_LIST_C}
#)

add_library(netdissect OBJECT
    ${NETDISSECT_SOURCE_LIST_C}
)

file(GLOB PROJECT_SOURCE_LIST_H
    *.h
)

source_group("Source Files" FILES ${PROJECT_SOURCE_LIST_C})
source_group("Header Files" FILES ${PROJECT_SOURCE_LIST_H})

######################################
# Register targets
######################################

#add_executable(helloworld helloworld.c)
#target_link_libraries(helloworld netdissect ${TCPDUMP_LINK_LIBRARIES})

######################################
# Write out the config.h file
######################################

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmakeconfig.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)

######################################
# Custom installation for iovnet
######################################

set(NETDISSECT_HEADERS
    print.h
    netdissect.h
    netdissect-stdinc.h
    compiler-tests.h
    varattrs.h
    funcattrs.h
    ip.h
    ip6.h)

install(FILES ${NETDISSECT_HEADERS} DESTINATION include/iovnet/tools/netdissect)