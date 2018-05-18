===============================================
SwiftHLM (Swift Hight-Latency Media) middleware
===============================================

# (C) Copyright 2016 IBM Corp.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
# implied.
# See the License for the specific language governing permissions and
# limitations under the License.

### Version: 0.2.2

### Authors:
Slavisa Sarafijanovic (sla@zurich.ibm.com)
Harald Seipp (seipp@de.ibm.com)

### Content:
1. Description (Function Overview)
2. Requirements
3. Install
4. Configure
5. Activate
6. HLM Backend
7. External Interface and Usage Examples 
8. Desing/internals Overview
9. References

1. Description (Function Overview)
===============================================

SwiftHLM is useful for running OpenStack Swift on top of high
latency media (HLM) storage, such as tape or optical disk archive based
backends, allowing to store cheaply and access efficiently large amounts of
infrequently used object data.

SwiftHLM can be added to OpenStack Swift (without modifying Swift itself) to
extend Swift's interface and thus allow to explicitly control and query the
state (on disk or on HLM) of Swift objects data, including efficient prefetch
of bulk of objects from HLM to disk when those objects need to be accessed.
This function previously missing in Swift can be seen similar to Amazon Glacier
[1], either through the Glacier API or the Amazon S3 Lifecycle Management API
[2].

BDT Tape Library Connector (open source) [3] and IBM Spectrum Archive [4] are
examples of HLM backends that provide important and complex functions to manage
HLM resources (tape mounts/unmounts to drives, serialization of requests for
tape media and tape drives resources) and can use SwiftHLM functions for a
proper integration with Swift.

Access to data stored on HLM could be done transparently, without using
SwiftHLM, but that does not work well in practice for many important use cases
and for various reasons as discussed in [5]. In [5] it is also explained how
SwiftHLM function can be orthogonal and complementary to Swift (ring to ring)
tiering [6].

SwiftHLM version 0.2.2 provides the following basic HLM functions on the external
Swift interface:

- MIGRATE (container or an object from disk to HLM)
- RECALL (i.e. prefetch a container or an object from HLM to disk)
- STATUS (get status for a container or an object)
- REQUESTS (get status of migration and recall requests previously submitted
  for a contaner or an object).

MIGRATE and RECALL are asynchronous operations, meaning that the request from
user is queued and user's call is responded immediately, then the request is
processed as a background task. Requests are currently processed in a FIFO
manner (scheduling optimizations are future work).  REQUESTS and STATUS are
synchronous operations that block the user's call until the queried information
is collected and returned.

Detailed (still exemplary and not standardized) syntax and usage examples are 
provided below in section "7. External Interface and Usage Examples".

For each of these functions, SwiftHLM Middleware invokes additional SwiftHLM
components to perform the task, which includes calls to HLM storage backend,
for which a generic backend interface is defined below in section "6. HLM
Backend". Description of other components is provided in the header of
the implementation file for each component. 

2. Requirements
=============================================== 

- OpenStack Swift Juno, Kilo, or Liberty (tested) or a later release (not
  tested)
- HLM backend that supports SwiftHLM functions, see HLM Backend section below
  for details
- Python 2.7+


3. Install
===============================================

    Get code, e.g.: 
    git clone https://github.com/ibm-research/swifthlm

    Install:
    cd swifthlm
    python setup.py install

4. Configure
===============================================

4.1. Configure SwiftHLM middleware to work with Swift

  a) If Swift is installed from source
    Modify Swift's configuration file /etc/swift/proxy-server.conf to include hlm middleware.
    In section [pipeline:main] add hlm keyword into pipeline, e.g.:

      pipeline = catch_errors gatekeeper healthcheck proxy-logging cache bulk tempurl formpost slo dlo ratelimit tempauth hlm staticweb container-quotas account-quotas proxy-logging proxy-server

    Add a new section:
      # High latency media (hlm) middleware
      [filter:hlm]
      use = egg:swifthlm#swifthlm
      set log_level = INFO
      #set log_level = DEBUG

  b) If Swift installed as part of Spectrum Scale 4.2.1 and later:

    # mmces service stop OBJ --all

    Retrieve your current Swift middleware pipeline setting:
    # mmobj config list --ccrfile proxy-server.conf --section pipeline:main --property pipeline

    Example output for the previous command:
    pipeline = healthcheck cache formpost tempurl swift3 s3token authtoken keystoneauth container-quotas account-quotas staticweb bulk slo dlo proxy-server

    Edit content of swifthlm/config/proxy-server.conf.merge to match the
    previous pipeline configuration and add 'hlm' before 'proxy-server'. E.g.
    for the above listed previous pipeline configuration, after editing the
    line that configures the pipeline should look like:

    [pipeline:main]
    pipeline =  healthcheck cache formpost tempurl swift3 s3token authtoken keystoneauth container-quotas account-quotas staticweb bulk slo dlo hlm proxy-server

    To write back the configuration and register the swifthlm middleware, run:
    # mmobj config change --ccrfile proxy-server.conf --merge-file /tmp/proxy-server.conf.merge

4.2 Configure passwordless ssh from Swift Proxy and SwiftHLM Dispatcher nodes to
    Swift storage nodes, for the user used to run Swift and SwiftHLM processes,
    steps 4.2.1 - 4.2.2. This is typically swift user, but can also be stack 
    user (in devstack deployments) or any other user. Normally this user is not
    a privileged user and cannot execute privileged operations. 

4.2.1 Make sure the user from 4.2 is allowed ssh login. 
    
    E.g. if this user is swift, and if swift user entry in /etc/passwd is:
    swift:x:160:160:OpenStack Swift Daemons:/var/lib/swift:/sbin/nologin
    ... modify it to:
    swift:x:160:160:OpenStack Swift Daemons:/var/lib/swift:/bin/bash

    (Note: a possible future improvement is to move SwiftHLM Handler function
    from the remotely invokable pythyon module into a remotely accesible service
    running on a proxy node, and thus avoid need to allow swift user ssh login.)

4.2.2 Setup key-based ssh for the user from 4.2:
    - Use ssh-keygen to generate RSA keys on Proxy and Dispatcher nodes
    - cp content of /home/swift/.ssh/id_rsa.pub from Proxy and Dispatcher nodes
    into /home/swift/.ssh/authorized_keys on storage nodes

4.3 Register SwiftHLM Dispatcher with systemd:

    cp swifthlm/config/swifthlm.dispatcher.service /etc/systemd/system/swifthlm.dispatcher.service
    
    If the user from 4.2 is not swift user:
    - edit /etc/systemd/system/swifthlm.dispatcher.service, line "User=swift",
    and replace swift with the user from 4.2 
    Note: if you further change this file upon the dispatcher service is 
    is started, run 'systemctl daemon-reload' for the change to take effect

4.4 Configure SwiftHLM to use a specific connector/backend:

    More information about integrating and configuring HLM backends is provided
    in Section 6. HLM Backend. 
    
    If SwiftHLM is not configured to use a specific connector/backend, a dummy
    connector/backend provided and installed as part of SwiftHLM will be used
    as the default one.

    Connector for LTFS DM backend is installed as part of SwiftHLM
    installation, but is not enabled by default (see Section 6. for how to do
    that). In order to enable and use LTFS DM connector, LTFS DM backend needs
    to be installed from https://github.com/ibm-research/LTFS-Data-Management
    LTFS DM is an open source software that adds tape storage to a standard
    disk based Linux filesystem - it keeps the original namespace of the disk
    file system exposed to users and applications (via standard POSIX
    interface) but allows migrating file data to and from attached tape
    storage. 

5. Activate
===============================================

To activate SwiftHLM middleware, restart Swift services:

  a) Swift installed from source:
    # swift-init main reload

  b) Spectrum Scale 4.1.1 or later:
    # mmces service start OBJ --all

To start SwiftHLM Dispatcher service (one one node, e.g. a proxy node):
    Start:
      # systemctl start swifthlm.dispatcher
    Check status:
      systemctl status swifthlm.dispatcher -l
    If/when needed, dispatcher can be stopped using:
      # systemctl stop swifthlm.dispatcher

To use SwiftHLM, see instruction in Section 7.

Note: If SwiftHLM is not configured to use a specific connector/backend
(Section 6), a dummy connector/backend provided and installed as part of
SwiftHLM will be used as the default one. 

6. HLM Backend
===============================================

An HLM backend that supports SwiftHLM functions (MIGRATE, RECALL, STATUS) is
exposed to Swift in the same way as if SwiftHLM is not used (via a file system
interface and a Swift ring definition), plus it needs to additionally support
processing and responding requests from SwiftHLM middleware for performing
SwiftHLM functions.

SwiftHLM Handler is the component of SwiftHLM  that invokes backend HLM
operations via SwiftHLM generic backend interface (GBI). For each backend a
Connector needs to be implemented that maps GBI requests to the backend HLM
operations. 

A backend specific connector can be installed as a standard python module, or
simply stored as a .py file at arbirary location to which the swift user has
access. Then SwiftHLM should be configured to use that specific
connector/backend, by appending the content of
swifthlm/object-server.conf.merge file to /etc/swift/object-server.conf, and
edditing the corresponding configuration values to match the specific
connector/backend. 

Here is the example of the content of swifthlm/object-server.conf.merge edited
for use with LTFS Data Management backend (open sourced at
https://github.com/ibm-research/LTFS-Data-Management) after it is appended to
/etc/swift/object-server.conf:

#
### High latency media (hlm) configuration on storage node
[hlm]
## You can override the default log level here:
# set log_level = INFO
set log_level = DEBUG
## Declare SwiftHLM Connector (and consequently the Backend) to be used:
#
# Dummy Connector/Backend - used by default if no other connector is
# provided/declared
#swifthlm_connector_module = swifthlm.dummy_connector
#
# IBM Spectrum Archive and IBM Spectrum Protect (proprietary) Connector/Backend
#swifthlm_connector_module = swifthlmibmsa.ibmsa_swifthlm_connector
#
# LTFS DM (open source) Connector/Backend
# Note: this connector is installed by default as part of SwiftHLM install, but
# it is not activated (declared to use) by default. LTFS DM backend needs to be
# installed separately (https://github.com/ibm-research/LTFS-Data-Management)
swifthlm_connector_module = swifthlm.ltfsdm_connector
#
# Your own Connector/Backend
# EITHER define the connector python module name (if installed as a python
# module), e.g.:
# swifthlm_connector_module = swifthlmxxx.opt_disc_connector
# OR specify the connector directory path and filename:
#swifthlm_connector_dir = /opt/xxx/swifthlmconnector
#swifthlm_connector_filename = connector.py
#
## Connector/Backend specific settings
#
## LTFS DM Connector settings
[ltfsdm]
# Path to ltfsdm binary. If not set, /usr/local/bin/ltfsdm is used by default.
ltfsdm_path = /usr/local/bin/ltfsdm
# Path to the directory to use for temporary files
connector_tmp_dir = /tmp/swifthlm
# Tape storage pool to use for storing objects data
tape_storage_pool = swiftpool
#
# IBM Spectrum Archive/Protect Connector settings
[ibmsasp]
# IBM Spectrum Archive/Protect Connector configuration
connector_tmp_dir = /tmp/swifthlm
# IBM Spectrum Archive/Protect or LTFS DM Backend configuration
gpfs_filesystem_or_fileset = /mnt/gpfs
library = library0
tape_storage_pool = swiftpool@library0



To use SwiftHLM with IBM Spectrum Archive or IBM Spectrum Protect connector and
backend (proprietary, see
http://www.redbooks.ibm.com/abstracts/redp5430.html?Open for how to get,
install and configure it), in the above shown content appended to
/etc/swift/object-server.conf, change the following 2 lines:

#swifthlm_connector_module = swifthlmibmsa.ibmsa_swifthlm_connector
swifthlm_connector_module = swifthlm.ltfsdm_connector

into:

swifthlm_connector_module = swifthlmibmsa.ibmsa_swifthlm_connector
#swifthlm_connector_module = swifthlm.ltfsdm_connector


7. External Interface and Usage Examples 
===============================================

* Syntax for using SwiftHLM enabled Swift via a standard (unmodified) curl Swift client:

curl -H "X-Auth-Token: $TOKEN" -X POST "http://zagreb.zurich.ibm.com:8080/hlm/v1/migrate/AUTH_test/cont1
curl -H "X-Auth-Token: $TOKEN" -X POST "http://zagreb.zurich.ibm.com:8080/hlm/v1/recall/AUTH_test/cont1
curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb.zurich.ibm.com:8080/hlm/v1/status/AUTH_test/cont1
curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb.zurich.ibm.com:8080/hlm/v1/requests/AUTH_test/cont1


* Examples of outputs for the above commands:

##### Get status of Object cont3/obj00:

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/status/AUTH_test/cont3/obj00" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    38  100    38    0     0     46      0 --:--:-- --:--:-- --:--:--    46
{
    "/AUTH_test/cont3/obj00": "resident"
}

real    0m0.831s
user    0m0.017s
sys     0m0.008s
[root@belgrade ~]#

##### Get status of all Objects of Container cont3:

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/status/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   114  100   114    0     0    128      0 --:--:-- --:--:-- --:--:--   128
{
    "/AUTH_test/cont3/obj00": "resident",
    "/AUTH_test/cont3/obj01": "resident",
    "/AUTH_test/cont3/obj02": "resident"
}

real    0m0.892s
user    0m0.016s
sys     0m0.007s
[root@belgrade ~]#

##### Migrate Object cont3/obj00:

[root@belgrade ~]#
[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X POST "http://zagreb:8080/hlm/v1/migrate/AUTH_test/cont3/obj00"
Accepted migrate request.

real    0m0.058s
user    0m0.001s
sys     0m0.003s
[root@belgrade ~]#

##### Check if request to migrate Object cont3/obj00 is completed:

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/requests/AUTH_test/cont3/obj00" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    68  100    68    0     0   1910      0 --:--:-- --:--:-- --:--:--  1942
[
    "20170303034043.566--migrate--AUTH_test--cont3--0--obj00--pending"
]

real    0m0.041s
user    0m0.017s
sys     0m0.007s
[root@belgrade ~]#

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/requests/AUTH_test/cont3/obj00" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    53  100    53    0     0   1566      0 --:--:-- --:--:-- --:--:--  1606
[
    "There are no pending or failed SwiftHLM requests."
]

real    0m0.039s
user    0m0.013s
sys     0m0.010s
[root@belgrade ~]#

##### Get status of all Objects of Container cont3:

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/status/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   114  100   114    0     0    115      0 --:--:-- --:--:-- --:--:--   115
{
    "/AUTH_test/cont3/obj00": "migrated",
    "/AUTH_test/cont3/obj01": "resident",
    "/AUTH_test/cont3/obj02": "resident"
}

real    0m0.991s
user    0m0.013s
sys     0m0.010s
[root@belgrade ~]#

##### Migrate entire container cont3 (but make tape backend unavailable on one of the storage nodes):

ot@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X POST "http://zagreb:8080/hlm/v1/migrate/AUTH_test/cont3"
Accepted migrate request.

real    0m0.062s
user    0m0.003s
sys     0m0.003s
[root@belgrade ~]#

##### Check requests for Container cont3:

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/requests/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    61  100    61    0     0   1923      0 --:--:-- --:--:-- --:--:--  1967
[
    "20170303034800.465--migrate--AUTH_test--cont3--0--pending"
]

real    0m0.039s
user    0m0.015s
sys     0m0.007s
[root@belgrade ~]#

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/requests/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    60  100    60    0     0   1673      0 --:--:-- --:--:-- --:--:--  1714
[
    "20170303034800.465--migrate--AUTH_test--cont3--0--failed"
]

real    0m0.041s
user    0m0.015s
sys     0m0.008s
[root@belgrade ~]#

##### Get status of all Objects of Container cont3 (tape backend is fixed and again available):

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/status/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   113  100   113    0     0    103      0  0:00:01  0:00:01 --:--:--   103
{
    "/AUTH_test/cont3/obj00": "migrated",
    "/AUTH_test/cont3/obj01": "unknown",
    "/AUTH_test/cont3/obj02": "migrated"
}

real    0m1.103s
user    0m0.014s
sys     0m0.009s
[root@belgrade ~]#

##### Resubmit migration for Container cont3:

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X POST "http://zagreb:8080/hlm/v1/migrate/AUTH_test/cont3"
Accepted migrate request.

real    0m0.057s
user    0m0.000s
sys     0m0.004s
[root@belgrade ~]#
[root@belgrade ~]#
[root@belgrade ~]#

##### Check requests for Container cont3:

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/requests/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   121  100   121    0     0   3275      0 --:--:-- --:--:-- --:--:--  3361
[
    "20170303035302.857--migrate--AUTH_test--cont3--0--pending",
    "20170303034800.465--migrate--AUTH_test--cont3--0--failed"
]

real    0m0.043s
user    0m0.015s
sys     0m0.009s
[root@belgrade ~]#

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/requests/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    53  100    53    0     0   1780      0 --:--:-- --:--:-- --:--:--  1827
[
    "There are no pending or failed SwiftHLM requests."
]

real    0m0.035s
user    0m0.011s
sys     0m0.012s
[root@belgrade ~]#

##### Get status of Objects of Container cont3:

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/status/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   114  100   114    0     0    104      0  0:00:01  0:00:01 --:--:--   105
{
    "/AUTH_test/cont3/obj00": "migrated",
    "/AUTH_test/cont3/obj01": "migrated",
    "/AUTH_test/cont3/obj02": "migrated"
}

real    0m1.092s
user    0m0.017s
sys     0m0.006s
[root@belgrade ~]#

##### Recall all Objects of Container cont3:

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X POST "http://zagreb:8080/hlm/v1/recall/AUTH_test/cont3"
Accepted recall request.

real    0m0.064s
user    0m0.000s
sys     0m0.005s
[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/requests/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    53  100    53    0     0   1821      0 --:--:-- --:--:-- --:--:--  1827
[
    "There are no pending or failed SwiftHLM requests."
]

real    0m0.035s
user    0m0.011s
sys     0m0.012s
[root@belgrade ~]#

##### Get status of Objects of Container cont3 (now on disk and tape, thus "premigrated"):

[root@belgrade ~]# time curl -H "X-Auth-Token: $TOKEN" -X GET "http://zagreb:8080/hlm/v1/status/AUTH_test/cont3" | python -mjson.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   123  100   123    0     0    125      0 --:--:-- --:--:-- --:--:--   125
{
    "/AUTH_test/cont3/obj00": "premigrated",
    "/AUTH_test/cont3/obj01": "premigrated",
    "/AUTH_test/cont3/obj02": "premigrated"
}

real    0m0.986s
user    0m0.015s
sys     0m0.007s
[root@belgrade ~]#

##### 


8. Design/Internals Overview

This section provides overview of SwiftHLM components and their join operation
for providing the function described in section 1.

SwiftHLM workflow for processing MIGRATION requests (and it is same for RECALL
requests) is as follows.  SwiftHLM middleware on proxy server intrcepts
SwiftHLM migration requests and queues them inside Swift, by storing them into
a special HLM-dedicated container as zero size objects.  After SwiftHLM request
is queued, 202 code is returned to the application.

Another SwiftHLM process, called SwiftHLM Dispatcher, is processing the queued
requests asynchronously with respect to user/application that submitted them.
It picks a request from the queue, in FIFO or a more advance manner, and groups
the requests into one list per involved storage node. 

For each storage node/list Dispatcher invokes remotely a SwiftHLM program on
that storage node (the name of that program is SwiftHLM Handler), and provides
it with the list. Handler could also be a long running process listening for
and processing submissions from Dispatcher. Either way, the function performed
by Handler is to map the objects to files (or to HLM backend objects) and
submits the file list and the migration requests to HLM backend, if the backend
already provide the function to move data between LLM (low latency media) and
HLM (hight latency media). Examples of backends with such function are IBM
Spectrum Archive and BDT Tape Library Connector. 

In order to support different backends, a Generic Backend Interface is defined
and used by Handler to submit the request to HLM backend, via the backend
specific Connector that maps the request to the backend specific API. If HLM
backend does not support moving data and managing object state, the backend
Connector needs to implement that function as well.

Once the backend completes the operation the result (succes or failure) is
propagated back to the dispatcher. In case of success, the request is removed
from the queue, otherwise it is marked as failed and kept in the queue for some
period (to be able to answer the request status queries). One could also
consider implementing request retries. 

Querying object status (STATUS) is processed by SwiftHLM middleware
synchrounously, by groupping the queries per storage nodes and invoking the
Handler (same as Dispatcher does for migration and recall), but for status the
SwiftHLM middleware also merges the statuses reportd by the backend and
provides the merged result to the Swift application.

Querying requests status (REQUESTS) Query for requests status for an object or
a container are processed by SwiftHLM middleware, by reading listing of the
special HLM-dedicated container(s). If there are not pending (incompleted or
failed) requests for a container, the previously submitted operations for that
container may be considered completed. This is more efficient than to query
state for each object of a container.

9. References
===============================================
[1] Amazon Glacier API, http://docs.aws.amazon.com/amazonglacier/latest/dev/amazon-glacier-api.html  
[2] Amazon S3 integration with Glacier, https://aws.amazon.com/blogs/aws/archive-s3-to-glacier   
[3] Tape Library Connector, https://github.com/BDT-GER/SWIFT-TLC
[4] IBM Spectrum Archive, http://www-03.ibm.com/systems/storage/tape/ltfs/
[5] SwiftHLM design discussion,  https://wiki.openstack.org/wiki/Swift/HighLatencyMedia
[6] Swift ring to ring tiering, https://review.openstack.org/#/c/151335/3/specs/in_progress/tiering.rst
