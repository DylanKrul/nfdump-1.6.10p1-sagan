Modified Nfdump for use with Sagan
----------------------------------

This is a modified version of nfdump utilities to support the Sagan 
log analysis engine (https://github.com/beave/sagan). 

When compiling, build with the "--enable-sagan" flag.  This adds a
new command line option for nfcapd (-F).  For example:

nfcapd -l /var/log/netflow-archive -p 2056 -F /var/run/sagan-netflow.fifo

This tells nfcapd to record data,  as normal,  to the /var/log/netflow-archive
directory.  nfcapd should listen on port 2056 for netflow data. 
That data should be written the /var/run/saggan-netflow.fifo.  The FIFO
written to in the Sagan format.  That is,  no additional formating is
necessary.

A Sagan + Netflow HOWTO will be released soon.


Original README continues below
-------------------------------

Stable Release v1.6.10

See the Changelog file for all changes in release 1.6.10

Notes on NSEL/ASA support
-------------------------

nfdump-1.6.9 includes a new written from scratch implemented NSEL/ASA
module. It's based on the CISCO ASA Spec 8.4: 
"Implementation Note for NetFlow Collectors, Version 8.4"
Due to this new implementation, nfdump-1.6.9 is not compatible with old 
nfdump-1.5.8-2-NSEL.
To build nfdump, add --enable-nsel to the configure command. By enabling 
the ASA/NSEL option, nfdump processes normal flows as well ASA/NSEL records 
likewise. nfcapd adds by default all required NSEL extesions equivalent 
to '-Tnsel'

Note on NEL support
-------------------

nfdump-1.6.9 includes a new module for decoding the CISCO NEL ( NAT event
logging ) records. It's considered to be experimantal, as no official 
documentation can be found. Let me know otherwise.
To build nfdump, add --enable-nel to the configure command. By enabling 
the NEL option, nfdump processes normal flows as well NEL records 
likewise. nfcapd adds by default all required NEL extesions equivalent 
to '-Tnel'

Although it's possibel to enable NSEL und NEL likewise, users could get 
confused by nfdump output, as NSEL output format overwrites NEL format.
In that case you need explicitly to define -o nel.

Notes on IPFIX
---------------

nfdump contains an IPFIX module for decoding IPFIX data. It
is considered not yet to be complete and does not yet support full IPFIX.
o Supports basically same feature set of elements as netflow_v9 module
o Only UDP traffic is accepted no SCTP so far
o No sampling support.
o Still more test data needed. If you would like to see more IPFIX 
  support, please contact me. 


General README
--------------

This is a small description, what the nfdump tools do and how they work.
Nfdump is distributed under the BSD license - see BSD-license.txt

The nfdump tools collect and process netflow data on the command line. 
They are part of the NFSEN project, which is explained more detailed at 
http://www.terena.nl/tech/task-forces/tf-csirt/meeting12/nfsen-Haag.pdf

The Web interface mentioned is not part of nfdump and is available at
http://nfsen.sourceforge.net

nfdump tools overview:
----------------------

nfcapd - netflow collector daemon. 
Reads the netflow data from the network and stores the flow records 
into files.  Automatically rotates files every n minutes. ( typically 
every 5 min ) nfcapd reads netflow versions v1, v5, v7 and v9 flows 
as well as IPFIX flows transparently. Several netflow streams can be 
sent to a single or collector.

nfdump - netflow dump.
Reads the netflow data from the files stored by nfcapd. It's filter 
syntax is similar to tcpdump ( pcap like ) but for netflow adapted. 
If you like tcpdump you will like nfdump. nfdump displays netflow 
data and/or creates top N statistics of flows, bytes, packets. nfdump 
has a powerful and flexible flow aggregation including bi-directional 
flows. The output format is user selectable and also includes a simple 
csv format for post processing.

nfreplay - netflow replay
Reads the netflow data from the files stored by nfcapd and sends it
over the network to another host.

nfexpire - expire old netflow data
Manages data expiration. Sets appropriate limits.

Optional binaries:

nfprofile - netflow profiler. Required by NfSen
Reads the netflow data from the files stored by nfcapd. Filters the 
netflow data according to the specified filter sets ( profiles ) and
stores the filtered data into files for later use. 

nftrack - Port tracking decoder for NfSen plugin PortTracker.

ft2nfdump - read flow-tools format - Optional tool
ft2nfdump acts as a pipe converter for flow-tools data. It allows
to read any flow-tools data and process and save it in nfdump format.

sfcapd - sflow collector daemon
scfapd collects sflow data and stores it into nfcapd comaptible files.
"sfcapd includes sFlow(TM), freely available from http://www.inmon.com/".

nfreader - Framework for programmers
nfreader is a framework to read nfdump files for any other purpose.
Own C code can be added to process flows. nfreader is not installed

parse_csv.pl - Simple reader, written in Perl.
parse_csv.pl reads nfdump csv output and print the flows to stdout.
This program is intended to be a framework for post processing flows
for any other purpose.

Note for sflow users:
sfcapd and nfcapd can be used concurrently to collect netflow and sflow
data at the same time. Generic command line options apply to both 
collectors likewise. Due to lack of availability of sflow devices,
I could not test the correct output of IPv6 records. Users are requested 
to send feedback to the list or directly to me. sfcapd's sflow decoding 
module is based on InMon's sflowtool code and supports similar fields as 
nfcapd does for netflow v9, which is a subset of all available sflow 
fields in an sflow record. More fields may be integrated in future 
versions of sfcapd.


Compression
-----------
Binary data files can optionally be compressed using the fast LZO1X-1 
compression. For more details on this algorithm see, 
http://www.oberhumer.com/opensource/lzo. LZO1X-1 is very fast, so
that compression can be used in real time by the collector. LZO1X-1
reduces the file size around 50%. You can check the compression speed
for your system by doing ./nftest <path/to/an/existing/netflow/file>. 


Principle of Operation:
-----------------------
The goal of the design is to able to analyze netflow data from
the past as well as to track interesting traffic patterns 
continuously. The amount of time back in the past is limited only
by the disk storage available for all the netflow data. The tools
are optimized for speed for efficient filtering. The filter rules
should look familiar to the syntax of tcpdump ( pcap compatible ).

All data is stored to disk, before it gets analyzed. This separates
the process of storing and analyzing the data. 

The data is organized in a time-based fashion. Every n minutes
- typically 5 min - nfcapd rotates and renames the output file
with the timestamp nfcapd.YYYYMMddhhmm of the interval e.g. 
nfcapd.200907110845 contains data from July 11th 2009 08:45 onward.
Based on a 5min time interval, this results in 288 files per day.

Analyzing the data can be done for a single file, or by concatenating
several files for a single output. The output is either ASCII text
or binary data, when saved into a file, ready to be processed again
with the same tools.

You may have several netflow sources - let's say 'router1' 'router2'
and so on. The data is organized as follows:

/flow_base_dir/router1
/flow_base_dir/router2

which means router1 and router2 are subdirs of the flow_base_dir.

Although several flow sources can be sent to a single collector,
It's recommended to have multiple collector on busy networks for 
each source.
Example: Start two collectors on different ports:

nfcapd -w -D -S 2 -B 1024000 -l /flow_base_dir/router1 -p 23456
nfcapd -w -D -S 2 -B 1024000 -l /flow_base_dir/router2 -p 23457

nfcapd can handle multiple flow sources.
All sources can go into a single file or can be split:

All into the same file:
nfcapd -w -D -S 2 -l /flow_base_dir/routers -p 23456

Collected on one port and split per source:
nfcapd -w -D -S 2 -n router1,172.16.17.18,/flow_base_dir/router1 \
  -n router2,172.16.17.20,/flow_base_dir/router2 -p 23456

See nfcapd(1) for a detailed explanation of all options.

Security: none of the tools requires root privileges, unless you have
a port < 1024. However, there is no access control mechanism in nfcapd.
It is assumed, that host level security is in place to filter the 
proper IP addresses.

See the manual pages or use the -h switch for details on using each of 
the programs. For any questions send email to phaag@users.sourceforge.net

Configure your router to export netflow. See the relevant documentation
for your model. 

A generic Cisco sample configuration enabling NetFlow on an interface:

    ip address 192.168.92.162 255.255.255.224
	 interface fastethernet 0/0
	 ip route-cache flow

To tell the router where to send the NetFlow data, enter the following 
global configuration command:

	ip flow-export 192.168.92.218 9995
	ip flow-export version 5 

	ip flow-cache timeout active 5

This breaks up long-lived flows into 5-minute segments. You can choose 
any number of minutes between 1 and 60;


Netflow v9 full export example of a cisco 7200 with sampling enabled:

    interface Ethernet1/0
     ip address 192.168.92.162 255.255.255.224
     duplex half
     flow-sampler my-map
    !
    !
    flow-sampler-map my-map
     mode random one-out-of 5
    !
    ip flow-cache timeout inactive 60
    ip flow-cache timeout active 1
    ip flow-capture fragment-offset
    ip flow-capture packet-length
    ip flow-capture ttl
    ip flow-capture vlan-id
    ip flow-capture icmp
    ip flow-capture ip-id
    ip flow-capture mac-addresses
    ip flow-export version 9
    ip flow-export template options export-stats
    ip flow-export template options sampler
    ip flow-export template options timeout-rate 1
    ip flow-export template timeout-rate 1
    ip flow-export destination 192.168.92.218 9995


See the relevant documentation for a full description of netflow commands

Note: Netflow version v5 and v7 have 32 bit counter values. The number of
packets or bytes may overflow this value, within the flow-cache timeout
on very busy routers. To prevent overflow, you may consider to reduce the 
flow-cache timeout to lower values. All nfdump tools use 64 bit counters 
internally, which means, all aggregated values are correctly reported.

The binary format of the data files is netflow version independent.
For speed reasons the binary format is machine architecture dependent, and 
as such can not be exchanged between little and big endian systems.
Internally nfdump does all processing IP protocol independent, which means
everything works for IPv4 as well as IPv6 addresses.
See the nfdump(1) man page for details. 

netflow version 9:
nfcapd supports a large range of netflow v9 tags. Version 1.6 nfdump 
supports the following fields. This list can be found in netflow_v9.h

(See orginal README for more information)


32 and 64 bit counters are supported for any counters. However, internally
nfdump stores packets and bytes counters always as 64bit counters. 
16 and 32 bit AS numbers are supported.

Extensions: nfcapd supports a large number of v9 tags. In order to optimise
disk space and performance, v9 tags are grouped into a number of extensions
which may or may not be stored into the data file. Therefore the v9 templates
configured on the exporter may be tuned with the collector. Only the tags 
common to both are stored into the data files. Extensions can be switch
on/off by using the -T option.

Sampling: By default, the sampling rate is set to 1 (unsampled) or to 
any given value specified by the -s cmd line option. If sampling information 
is found in the netflow stream, it overwrites the default value. Sampling 
is automatically recognised when announced in v9 option templates 
(tags #48, #49, #50 ) or in the unofficial v5 header hack. Note: Not all 
platforms (or IOS versions) support exporting sampling information in 
netflow data, even if sampling is configured. The number of bytes/packets 
in each netflow record is automatically multiplied by the sampling rate. 
The total number of flows is not changed as this is not accurate enough. 
(Small flows versus large flows)

nfcapd can listen on IPv6 or IPv4. Furthermore multicast is supported.

Flow-tools compatibility
------------------------
When building with configure option --enable-ftconv, the flow-tools converter
is compiled. Using this converter, any flow-tools created data can be read
and processed and stored by nfdump.

Example:

	flow-cat [options] | ft2nfdump | nfdump [options]


See the INSTALL file for installation details.
