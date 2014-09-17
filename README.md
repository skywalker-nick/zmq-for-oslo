OpenStack ZeroMQ Driver for oslo.messaging Deployment Guide
============

0MQ (also known as ZeroMQ, 0MQ, or zmq) looks like an embeddable networking library but acts like a concurrency framework. It gives you sockets that carry atomic messages across various transports like in-process, inter-process, TCP, and multicast. You can connect sockets N-to-N with patterns like fan-out, pub-sub, task distribution, and request-reply. It's fast enough to be the fabric for clustered products. Its asynchronous I/O model gives you scalable multi-core applications, built as asynchronous message-processing tasks. It has a score of language APIs and runs on most operating systems. 0MQ is from iMatix and is LGPLv3 open source.

Originally the zero in 0MQ was meant as "zero broker" and (as close to) "zero latency" (as possible). Since then, it has come to encompass different goals: zero administration, zero cost, and zero waste. More generally, "zero" refers to the culture of minimalism that permeates the project. We add power by removing complexity rather than by exposing new functionality.

Abstract
============

In Folsom, OpenStack introduces an optional messaging system using ZeroMQ.  For some deployments, especially for those super-large scenarios, it may be desirable to use a broker-less RPC mechanism to scale out. 

Currently (in Juno cycle), ZeroMQ is one of the RPC backend drivers in oslo.messaging. In the coming Juno release, as almost all the core projects in OpenStack have switched to oslo.messaging, ZeroMQ can be the only RPC driver across the OpenStack cluster. This document provides deployment information for this driver in oslo.messaging.

Other than AMQP-based drivers, like RabbitMQ or Qpid, ZeroMQ doesn¡¯t have any central brokers in oslo.messaging, instead, each host (running OpenStack services) is both ZeroMQ client and server. As a result, each host needs to listen to a certain TCP port for incoming connections and directly connect to other hosts simultaneously.

Topics are used to identify the destination for a ZeroMQ RPC call. There are two types of topics, bare topics and directed topics. Bare topics look like 'compute', while directed topics look like 'compute.machine1'.

Configuration
==============

Enabling (mandatory)
--------------------

To enable the driver, in the section [DEFAULT] of the conf file of each OpenStack project, the 'rpc_backend' flag must be set to 'zmq' and the 'rpc_zmq_host' flag must be set to the hostname of the current node.

        [DEFAULT]
        rpc_backend = zmq
        rpc_zmq_host = host1


Name Resolution (mandatory)
---------------------------

All TCP connections are made to hostnames. To specify, these are hostnames, not FQDNs. Host resolution is mandatory. The straightforward method is to set all the hostnames in /etc/hosts of each OpenStack node (where OpenStack services are running).

The best practice is to deploy viable nameservers which can resolve all hostnames in the cloud under a single domain. Machines should then have DNS search domain to resolve the bare hostnames.

i.e.
        
        dns zone for lab.example.com on 192.168.1.1
        zmq_hostname IN A 192.168.2.1

        /etc/resolv.conf:
        search lab.example.com
        nameserver 192.168.1.1


Match Making (mandatory)
-------------------------

The ZeroMQ driver implements a matching capability to discover hosts available for communication when sending to a bare topic. This allows broker-less communications.

The MatchMaker is pluggable and it provides two different MatchMaker classes in the current release.

MatchMakerRing: loads a static hash table from a JSON file, sends messages to a certain host via directed topics or cycles hosts per bare topic and supports broker-less fanout messaging. On fanout messages returns an array of directed topics (messages are sent to all destinations).

MatchMakerRedis: loads the hash table from a remote Redis server, supports dynamic host/topic registrations, host expiration, and hooks for consuming applications to acknowledge or neg-acknowledge topic.host service availability.

To set the MatchMaker class, use option 'rpc_zmq_matchmaker' in [DEFAULT] of each project.

        rpc_zmq_matchmaker = oslo.messaging._drivers.matchmaker_ring.MatchMakerRing
        or
        rpc_zmq_matchmaker = oslo.messaging._drivers.matchmaker_ring.MatchMakerRedis

To specify the ring file for MatchMakerRing, use option 'ringfile' in [matchmaker_ring] of each project.

An example for Nova:

        [matchmaker_ring]
        ringfile = /etc/nova/nova_matchmaker_ring.json

To specify the Redis server for MatchMakerRedis, use options in [matchmaker_redis] of each project.

        [matchmaker_redis]
        host = 127.0.0.1
        port = 6379
        password = None


MatchMaker Data Source (mandatory)
-----------------------------------

MatchMaker data source is stored in files or Redis server discussed in the previous section. How to make up the database is the key issue for making ZeroMQ driver work.

If deploying the MatchMakerRing, a ring file is required. The format of the ring file should contain a hash where each key is a base topic and the values are hostname arrays to be sent to.

i.e.

        /etc/nova/nova_matchmaker_ring.json
        {
            'scheduler': ['host1', 'host2'],
            'conductor': ['host1', 'host2'],
        }

If deploying the MatchMakerRedis, a Redis server is required. Each (K, V) pair stored in Redis is that the key is a base topic and the corresponding values are hostname arrays to be sent to.

Those AMQP-based methods like RabbitMQ and Qpid doesn't require any knowledge about the source and destination of any topic. However, ZeroMQ driver does. The challenging task is that you should learn and get all the (K, V) pairs from each OpenStack project to make up the matchmaker data source.

The matchmaker data source file can be downloaded from the repo:
https://github.com/li-ma/zmq-for-oslo


Message Receivers (mandatory)
-------------------------------

Each machine running OpenStack services, or sending RPC messages, must run the 'oslo-messaging-zmq-receiver' daemon. This receives replies to call requests and routes responses via IPC to blocked callers.

The IPC runtime directory needs to be fully owned by project owner. For example, in most Linux distribution, neutron services are owned by user neutron. As a result, the IPC runtime directory which should be '/var/run/neutron' also needs to be owned by user neutron. This can be accomplished via 'chown -R neutron:neutron /var/run/neutron'.

The IPC runtime directory, 'rpc_zmq_ipc_dir', can be set in [DEFAULT] section of each project conf file.

Currently, the script for running zmq-receiver doesn't include in oslo.messaging. You can download it from the repo:
https://github.com/li-ma/zmq-for-oslo

The parameters for the script oslo-messaging-zmq-receiver should be:

        oslo-messaging-zmq-receiver
            --config-file /etc/neutron/neutron.conf
            --log-file /var/log/neutron/zmq-receiver.log
        
if the project is neutron.


Listening Ports (mandatory)
---------------------------

The ZeroMQ driver uses TCP to communicate. The port is configured with 'rpc_zmq_port' in [DEFAULT] section of each project, which defaults to 9500.

Suggest changing it for different project, like nova uses 9501, neutron uses 9502, etc.

For example:

        rpc_zmq_port = 9501

Thread Pool (optional)
-----------------------

Each service will launch threads for incoming requests. These threads are maintained via a pool, the maximum number of threads is limited by rpc_thread_pool_size. The default value is 1024. (This is a common RPC configuration variable, also applicable to Kombu and Qpid)

This configuration can be set in [DEFAULT] section of each project conf file.

For example:

        rpc_thread_pool_size = 1024


Listening Address (optional)
------------------------------

All services bind to an IP address or Ethernet adapter. By default, all services bind to '*', effectively binding to 0.0.0.0. This may be changed with the option 'rpc_zmq_bind_address' which accepts a wildcard, IP address, or Ethernet adapter.

This configuration can be set in [DEFAULT] section of each project conf file.

For example:

        rpc_zmq_bind_address = *

References
-----------

ZeroMQ Guide: http://zguide.zeromq.org/page:all

ZeroMQ deployment guide for Nova: http://ewindisch.github.io/nova/
(not applicable for oslo.messaging)
