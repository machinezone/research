*************************************************
Comparison of Networking Solutions for Kubernetes
*************************************************

Kubernetes requires that each container in a cluster has a unique, routable IP. Kubernetes doesn't assign IPs itself, leaving the task to third-party solutions.

In this study, our goal was to find the solution with the lowest latency, highest throughput, and the lowest setup cost. Since our load is latency-sensitive, our intent is to measure high percentile latencies at relatively high network utilization. We particularly focused on the performance under 30–50% of the maximum load, because we think this best represents the most common use cases of a non-overloaded system.


.. toctree::
    :maxdepth: 3
    :hidden:

    self


.. todo:: What are acceptable values for us?

.. todo::

    The testing scenarios or the general outline does not suggest that we're latency sensitive. I'd add that, saying something that since our load is latency-sensitive, our intent is to measure high percentile latencies at relatively high network utilization. This might provide better context to understand our load testing intent.


Competitors
===========


.. _reference:

Docker with ``--net=host``
--------------------------

This was our reference setup. All other competitors are compared against this setup.

The `--net=host <https://docs.docker.com/engine/reference/run/#network-settings>`__ option means that containers inherit the IPs of their host machines, i.e. no network containerization is involved.

A priori, no network containerization performs better than any network containerization; this is why we used this setup as a reference.


.. _flannel:

Flannel
-------

`Flannel <https://github.com/coreos/flannel>`__  is a virtual network solution maintained by the `CoreOS <https://coreos.com>`__ project. It's a well-tested, production ready solution, so it has the lowest setup cost.

When you add a machine with flannel to the cluster, flannel does three things:

#.  Allocates a subnet for the new machine using `etcd <https://github.com/coreos/etcd>`__
#.  Creates a virtual `bridge interface <http://www.linuxfoundation.org/collaborate/workgroups/networking/bridge>`__ on the machine (called *docker0 bridge*)
#.  Sets up a packet forwarding `backend <https://github.com/coreos/flannel#backends>`__:

    ``aws-vpc``
        Register the machine subnet in the Amazon AWS instance table. The number of records in this table is limited by 50, i.e. you can't have more than 50 machines in a cluster if you use flannel with ``aws-vpc``. Also, this backend works only with Amazon's AWS.

    ``host-gw``
        Create IP routes to subnets via remote machine IPs. Requires direct layer2 connectivity between hosts running flannel.

    ``vxlan``
        Create a virtual `VXLAN interface <https://en.wikipedia.org/wiki/Virtual_Extensible_LAN>`__.

Because flannel uses a bridge interface to forward packets, each packet goes through two network stacks when travelling from one container to another.

.. todo:: Put a picture here.


.. _ipvlan:

IPvlan
------
`IPvlan <https://www.kernel.org/doc/Documentation/networking/ipvlan.txt>`__ is driver in the Linux kernel that lets you create virtual interfaces with unique IPs without having to use a bridge interface.

To assign an IP to a container with IPvlan you have to:

#.  Create a container without a network interface at all
#.  Create an ipvlan interface in the default network namespace
#.  Move the interface to the container's network namespace

IPvlan is a relatively new solution, so there are no ready-to-use tools to automate this process. This makes it difficult to deploy IPvlan with many machines and containers, i.e. the setup cost is high.

However, IPvlan doesn't require a bridge interface and forwards packets directly from the NIC to the virtual interface, so we expected it to perform better than flannel.

.. todo:: Put a picture here.


Load Testing Scenario
=====================

For each competitor we run these steps:

#.  :ref:`Set up networking on two physical machines <environment>`
#.  Run `tcpkali <https://github.com/MachineZone/tcpkali>`_ in a container on one machine, let is send requests at a constant rate
#.  Run Nginx in a container on the other machine, let it respond with a fixed-size file
#.  Capture system metrics and tcpkali results

We ran the benchmark with the request rate varying from 50,000 to 450,000 requests per second (RPS).

On each request, Nginx responded with a static file of a fixed size: 350 B (100 B content, 250 B headers) or 4 KB.


Results
=======

#.  IPvlan shows the lowest latency and the highest maximum throughput. Flannel with ``host-gw`` and ``aws-vpc`` follows closely behind, however ``host-gw`` shows better results under maximum load.

#.  Flannel with ``vxlan`` shows the worst results in all tests. However, we suspect that its exceptionally poor 99.999 percentile is due to a bug.

#.  The results for a 4 KB response are similar to those for 350 B response, with two noticeable differences:

    -   the maximum RPS point is much lower, because with 4 KB responses it takes only ≈270k RPS to fully load a 10 Gbps NIC
    -   IPvlan is much closer to ``--net=host`` near the throughput limit

Our current choice is flannel with ``host-gw``. It doesn't have many dependencies (e.g. no AWS or new Linux version required), it's easy to set up compared to IPvlan, and it has sufficient performance characteristics. IPvlan is our backup solution. If at some point flannel adds IPvlan support, we'll switch to it.

Even though ``aws-vpc`` performed slightly better than ``host-gw``, its 50 machine limitation and the fact that it's hardwired to Amazon's AWS are a dealbreaker for us.


50,000 RPS, 350 B
-----------------

.. image:: images/100/latency-50000rps.png
    :width: 60%
    :align: center

At 50,000 requests per second, all candidates show acceptable performance. You can already see the main trend: IPVlan performs the best, ``host-gw`` and ``aws-vpc`` follow closely behind, ``vxlan`` is the worst.


150,000 RPS, 350 B
------------------

.. image:: images/100/latency-150000rps.png
    :width: 60%
    :align: center

.. table:: Latency percentiles at 150,000 RPS (≈30% of maximum RPS), ms

    .. list-table::
        :header-rows: 1
        :stub-columns: 1

        -   -   Setup
            -   95 %ile
            -   99 %ile
            -   99.5 %ile
            -   99.99 %ile
            -   99.999 %ile
            -   Max Latency
        -   -   IPvlan
            -   :textcolor:`<#008000> 0.7`
            -   :textcolor:`<#008000> 0.9`
            -   :textcolor:`<#008000> 1`
            -   :textcolor:`<#FF0000> 6.7`
            -   :textcolor:`<#FFD700> 9.9`
            -   :textcolor:`<#9ACD32> 18`
        -   -   ``aws-vpc``
            -   :textcolor:`<#9ACD32> 0.9`
            -   :textcolor:`<#9ACD32> 1.1`
            -   :textcolor:`<#9ACD32> 1.2`
            -   :textcolor:`<#9ACD32> 6.5`
            -   :textcolor:`<#9ACD32> 9.8`
            -   :textcolor:`<#008000> 15.7`
        -   -    ``host-gw``
            -   :textcolor:`<#9ACD32> 0.9`
            -   :textcolor:`<#9ACD32> 1.1`
            -   :textcolor:`<#9ACD32> 1.2`
            -   :textcolor:`<#008000> 5.9`
            -   :textcolor:`<#008000> 9.6`
            -   :textcolor:`<#FFD700> 24.3`
        -   -    ``vxlan``
            -   :textcolor:`<#FF0000> 1.2`
            -   :textcolor:`<#FF0000> 1.5`
            -   :textcolor:`<#FF0000> 1.6`
            -   :textcolor:`<#FFD700> 6.6`
            -   :textcolor:`<#FF0000> 201.9`
            -   :textcolor:`<#FF0000> 405.3`
        -   -   ``--net=host``
            -   0.5
            -   0.6
            -   0.6
            -   4.8
            -   8.9
            -   11.8

IPvlan is slightly better than ``host-gw`` and ``aws-vpc``, but it has the worst 99.99 percentile. ``host-gw`` performs slightly better than ``aws-vpc``.


.. _rps250-350b:

250,000 RPS, 350 B
------------------

.. image:: images/100/latency-250000rps.png
    :width: 60%
    :align: center

This load is also expected to be common on production, so these results are particularly important.

.. table:: Latency percentiles at 250,000 RPS (≈50% of maximum RPS), ms

    .. list-table::
        :header-rows: 1

        -   -   Setup
            -   95 %ile
            -   99 %ile
            -   99.5 %ile
            -   99.99 %ile
            -   99.999 %ile
            -   Max Latency
        -   -   IPvlan
            -   :textcolor:`<#008000> 1`
            -   :textcolor:`<#008000> 1.2`
            -   :textcolor:`<#008000> 1.4`
            -   :textcolor:`<#9ACD32> 6.3`
            -   :textcolor:`<#9ACD32> 10.1`
            -   :textcolor:`<#008000> 24.3`
        -   -   ``aws-vpc``
            -   :textcolor:`<#FFD700> 1.2`
            -   :textcolor:`<#FFD700> 1.5`
            -   :textcolor:`<#9ACD32> 1.6`
            -   :textcolor:`<#008000> 5.6`
            -   :textcolor:`<#008000> 9.4`
            -   :textcolor:`<#9ACD32> 27.3`
        -   -    ``host-gw``
            -   :textcolor:`<#9ACD32> 1.1`
            -   :textcolor:`<#9ACD32> 1.4`
            -   :textcolor:`<#9ACD32> 1.6`
            -   :textcolor:`<#FFD700> 8.6`
            -   :textcolor:`<#FFD700> 11.2`
            -   :textcolor:`<#FFD700> 40.1`
        -   -    ``vxlan``
            -   :textcolor:`<#FF0000> 1.5`
            -   :textcolor:`<#FF0000> 1.9`
            -   :textcolor:`<#FF0000> 2.1`
            -   :textcolor:`<#FF0000> 16.6`
            -   :textcolor:`<#FF0000> 202.4`
            -   :textcolor:`<#FF0000> 245.5`
        -   -   ``--net=host``
            -   0.7
            -   0.8
            -   0.9
            -   3.7
            -   7.7
            -   16.8

IPvlan again shows the best performance, but ``aws-vpc`` has the best 99.99 and 99.999 percentiles. ``host-gw`` outperforms ``aws-vpc`` in 95 and 99 percentiles.


350,000 RPS, 350 B
------------------

.. image:: images/100/latency-350000rps-no-vxlan.png
    :width: 60%
    :align: center

In most cases, the latency is close to the :ref:`250,000 RPS, 350 B <rps250-350b>` case, but it's rapidly growing after 99.5 percentile, which means that we are getting close to the maximum RPS.


.. _rps450-350b:

450,000 RPS, 350 B
------------------

This is the maximum RPS that produced sensible results.

IPvlan leads again with latency ≈30% worse than that of ``--net-host``:

.. image:: images/100/latency---net=host-450000rps.png
    :width: 60%
    :align: center

.. image:: images/100/latency-ipvlan-450000rps.png
    :width: 60%
    :align: center

Interestingly,  ``host-gw`` performs much better than ``aws-vpc``:

.. image:: images/100/latency-450000rps.png
    :width: 60%
    :align: center


.. _rps500-350b:

500,000 RPS, 350 B
------------------

Under 500,000 RPS, only IPvlan still works and even outperforms ``--net=host``, but the latency is so high that we think it would be of no use to latency-sensitive applications.

.. image:: images/100/latency-500000rps-no-host-gw.png
    :width: 60%
    :align: center


50k RPS, 4 KB
-------------

.. image:: images/4k/latency-50000rps.png
    :width: 60%
    :align: center

Bigger response results in higher network usage, but the leaderboard looks pretty much the same as with the smaller response:

.. table:: Latency percentiles at 50k RPS (≈20% of maximum RPS), ms

    .. list-table::
        :header-rows: 1

        -   -   Setup
            -   95 %ile
            -   99 %ile
            -   99.5 %ile
            -   99.99 %ile
            -   99.999 %ile
            -   Max Latency
        -   -   IPvlan
            -   :textcolor:`<#008000> 0.6`
            -   :textcolor:`<#008000> 0.8`
            -   :textcolor:`<#008000> 0.9`
            -   :textcolor:`<#9ACD32> 5.7`
            -   :textcolor:`<#008000> 9.6`
            -   :textcolor:`<#008000> 15.8`
        -   -   ``aws-vpc``
            -   :textcolor:`<#9ACD32> 0.7`
            -   :textcolor:`<#9ACD32> 0.9`
            -   :textcolor:`<#9ACD32> 1`
            -   :textcolor:`<#008000> 5.6`
            -   :textcolor:`<#9ACD32> 9.8`
            -   :textcolor:`<#FF0000> 403.1`
        -   -    ``host-gw``
            -   :textcolor:`<#9ACD32> 0.7`
            -   :textcolor:`<#9ACD32> 0.9`
            -   :textcolor:`<#9ACD32> 1`
            -   :textcolor:`<#FF0000> 7.4`
            -   :textcolor:`<#FFD700> 12`
            -   :textcolor:`<#9ACD32> 202.5`
        -   -    ``vxlan``
            -   :textcolor:`<#FF0000> 0.8`
            -   :textcolor:`<#FF0000> 1.1`
            -   :textcolor:`<#FF0000> 1.2`
            -   :textcolor:`<#9ACD32> 5.7`
            -   :textcolor:`<#FF0000> 201.5`
            -   :textcolor:`<#FFD700> 402.5`
        -   -   ``--net=host``
            -   0.5
            -   0.7
            -   0.7
            -   6.4
            -   9.9
            -   14.8


150k RPS, 4 KB
--------------

.. image:: images/4k/latency-150000rps.png
    :width: 60%
    :align: center

``Host-gw`` has a surprisingly poor 99.999 percentile, but it still shows good results for lower percentiles.

.. table:: Latency percentiles at 150k RPS (≈60% of maximum RPS), ms

    .. list-table::
        :header-rows: 1

        -   -   Setup
            -   95 %ile
            -   99 %ile
            -   99.5 %ile
            -   99.99 %ile
            -   99.999 %ile
            -   Max Latency
        -   -   IPvlan
            -   :textcolor:`<#008000> 1`
            -   :textcolor:`<#008000> 1.3`
            -   :textcolor:`<#008000> 1.5`
            -   :textcolor:`<#008000> 5.3`
            -   :textcolor:`<#9ACD32> 201.3`
            -   :textcolor:`<#FFD700> 405.7`
        -   -   ``aws-vpc``
            -   :textcolor:`<#9ACD32> 1.2`
            -   :textcolor:`<#9ACD32> 1.5`
            -   :textcolor:`<#9ACD32> 1.7`
            -   :textcolor:`<#9ACD32> 6`
            -   :textcolor:`<#008000> 11.1`
            -   :textcolor:`<#008000> 405.1`
        -   -    ``host-gw``
            -   :textcolor:`<#9ACD32> 1.2`
            -   :textcolor:`<#9ACD32> 1.5`
            -   :textcolor:`<#9ACD32> 1.7`
            -   :textcolor:`<#FF0000> 7`
            -   :textcolor:`<#FF0000> 211`
            -   :textcolor:`<#9ACD32> 405.3`
        -   -    ``vxlan``
            -   :textcolor:`<#FF0000> 1.4`
            -   :textcolor:`<#FF0000> 1.7`
            -   :textcolor:`<#FF0000> 1.9`
            -   :textcolor:`<#9ACD32> 6`
            -   :textcolor:`<#FFD700> 202.51`
            -   :textcolor:`<#FF0000> 406`
        -   -   ``--net=host``
            -   0.9
            -   1.2
            -   1.3
            -   4.2
            -   9.5
            -   404.7

250k RPS, 4 KB
--------------

.. image:: images/4k/latency-250000rps-no-vxlan.png
    :width: 60%
    :align: center

This is the maximum RPS with big response. ``aws-vpc`` performs much better than  ``host-gw``, unlike the :ref:`small response case <rps450-350b>`.

``Vxlan`` was excluded from the graph once again.


.. _environment:

Test Environment
================

Background
----------

To understand this article and reproduce our test environment, you should be familiar with the basics of high performance.

These articles provide useful insights on the topic:

-   `How to receive a million packets per second`__ *by CloudFlare*
-   `How to achieve low latency with 10Gbps Ethernet`__ *by CloudFlare*
-   `Scaling in the Linux Networking Stack`__ *from the Linux kernel documentation*

__ https://blog.cloudflare.com/how-to-receive-a-million-packets/
__ https://blog.cloudflare.com/how-to-achieve-low-latency/
__ https://www.kernel.org/doc/Documentation/networking/scaling.txt


Machines
--------

-   We use two `c4.8xlarge instances <https://aws.amazon.com/ec2/instance-types/#c4>`__ by Amazon AWS EC2 with CentOS 7.
-   Both machines have `enhanced networking <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking.html>`__ enabled.
-   Each machine is `NUMA <https://en.wikipedia.org/wiki/Non-uniform_memory_access>`__  with 2 processors; each processor has 9 cores, each core has 2 hyperthreads, which effectively allows to run 36 threads on each machine.
-   Each machine has a 10Gbps network interface card (NIC) and 60 GB memory.
-   To support enhanced networking and IPvlan, we've installed `Linux kernel 4.3.0`__ with Intel's ixgbevf driver.

__ http://elrepo.org/linux/kernel/el7/x86_64/RPMS/


Setup
-----

Modern NICs offer `Receive Side Scaling (RSS) <https://msdn.microsoft.com/en-us/library/windows/hardware/ff556942(v=vs.85).aspx>`__ via multiple `interrupt request (IRQ)  <https://en.wikipedia.org/wiki/Interrupt_request_(PC_architecture)>`__ lines. EC2 provides only two interrupt lines in a virtualized environment, so we tested several RSS and `Receive Packet Steering (RPS) Receive Packet Steering (RPS) <https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Performance_Tuning_Guide/network-rps.html>`__ configurations and ended up with following configuration, partly suggested by the Linux kernel documentation:

.. todo:: Put a picture here.

IRQ
    The first core on each of the two NUMA nodes is configured to receive interrupts from NIC.

    To match a CPU to a NUMA node, use ``lscpu``::

        $ lscpu | grep NUMA
        NUMA node(s):          2
        NUMA node0 CPU(s):     0-8,18-26
        NUMA node1 CPU(s):     9-17,27-35


    This is done by writing 0 and 9 to */proc/irq/<num>/smp_affinity_list*, where IRQ numbers are obtained with ``grep eth0 /proc/interrupts``::

        $ echo 0 > /proc/irq/265/smp_affinity_list
        $ echo 9 > /proc/irq/266/smp_affinity_list

RPS
    Several combinations for RPS have been tested. To improve latency, we offloaded the IRQ handling processors by using only CPUs 1–8 and 10–17. Unlike IRQ's *smp_affinity*, the *rps_cpus* sysfs file entry doesn't have a *_list* counterpart, so we use bitmasks to list the CPUs to which RPS can forward traffic\ [#]_::

        $ echo "00000000,0003fdfe" > /sys/class/net/eth0/queues/rx-0/rps_cpus
        $ echo "00000000,0003fdfe" > /sys/class/net/eth0/queues/rx-1/rps_cpus

Transmit Packet Steering (XPS)
    All NUMA 0 processors (including HyperThreading, i.e. CPUs 0-8, 18-26) were set to tx-0 and NUMA 1 (CPUs 9-17, 27-37) to tx-1\ [#]_::

        $ echo "00000000,07fc01ff" > /sys/class/net/eth0/queues/tx-0/xps_cpus
        $ echo "0000000f,f803fe00" > /sys/class/net/eth0/queues/tx-1/xps_cpus

Receive Flow Steering (RFS)
    We're planning to use 60k permanent connections, official documentation suggests to round up it to the nearest power of two::

    $ echo 65536 > /proc/sys/net/core/rps_sock_flow_entries
    $ echo 32768 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
    $ echo 32768 > /sys/class/net/eth0/queues/rx-1/rps_flow_cnt

Nginx
    Nginx uses 18 workers, each worker has it's own CPU (0-17). This is set by the `worker_cpu_affinity <http://nginx.org/en/docs/ngx_core_module.html#worker_cpu_affinity>`__ option:

    .. code-block:: nginx

        workers 18;
        worker_cpu_affinity 1 10 100 1000 10000 ...;

`Tcpkali <https://github.com/MachineZone/tcpkali>`_
    Tcpkali doesn't have built-in CPU affinity support. In order to make use of RFS, we run tcpkali in a taskset and tune the scheduler to make thread migrations happen rarely::

    $ echo 10000000 > /proc/sys/kernel/sched_migration_cost_ns
    $ taskset -ac 0-17 tcpkali --threads 18 ...

This setup allows us to spread interrupt load across the CPU cores more uniformly and achieve better throughput with the same latency compared to the other setups we have tried.

Cores 0 and 9 deal exclusively with NIC interrupts and don't serve packets, but they still are the most busy ones:

.. image:: images/cpu-load-tuned.png
   :width: 500
   :align: center

RedHat's `tuned <https://fedorahosted.org/tuned/>`__ was also used with the network-latency profile on.

To minimize the influence of nf_conntrack, `NOTRACK rules were added <http://serverfault.com/a/536303/99164>`__.

Sysctls was tuned to support large number of tcp connections:

.. code-block:: cfg

   fs.file-max = 1024000
   net.ipv4.ip_local_port_range = "2000 65535"
   net.ipv4.tcp_max_tw_buckets = 2000000
   net.ipv4.tcp_tw_reuse = 1
   net.ipv4.tcp_tw_recycle = 1
   net.ipv4.tcp_fin_timeout = 10
   net.ipv4.tcp_slow_start_after_idle = 0
   net.ipv4.tcp_low_latency = 1


.. rubric:: Footnotes

.. [#] `Linux kernel documentation: RPS Configuration <https://www.kernel.org/doc/Documentation/networking/scaling.txt>`__
.. [#] `Linux kernel documentation: XPS Configuration <https://www.kernel.org/doc/Documentation/networking/scaling.txt>`__

.. disqus::
