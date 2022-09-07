What makes NFS hard to learn?
-----------------------------
It's totally different from every other network service I've
dealt with. (``sshd``, ``nginx``, ``vsftpd``)

Userspace vs kernelspace server implementation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Instead of having a userspace program that serves requests, you
have both a userspace program and a kernel component. The
userspace program communicates with its corresponding kernel
component, and the kernel component does most of the work.

One vs many daemons
^^^^^^^^^^^^^^^^^^^
Instead of a single daemon program, there are several daemons,
each responsible for separate duties related to the exports.
Each connection gets a new thread of the daemon process.

Systemd services
^^^^^^^^^^^^^^^^
Instead of one systemd service unit, you have several, but only
the ``nfs-server`` service is essential. The ``nfs-client.target``
unit should be enabled for clients, but it's not strictly needed.
If you want to export quota service, you'll need ``rpc-quota.service``.

One vs many network protocols
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
For NFSv3, instead of a single protocol, you have many! At least
NFS, NSM, and NLM, but possibly more. These extra protocols
handle file locking, status information, and quota services.
NFSv4, a revamped version of the NFS protocol, folds all of those
facilities into the NFS protocol itself.

One fixed port vs dynamically assigned ports
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Instead of a static port assignment, you have network requests
via RPC that get assigned to appropriate ports dynamically by the
``portmap`` daemon. Actually, maybe some ports are static, I
don't really know. This is point of confusion for me.

Lots of supporting infrastructure
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If you need authentication services, Kerebros and LDAP get
involved. Those network services need supporting infrastructure,
too -- a DNS and NTP server. This probably only make sense for
remote connections if you're connecting to a VPN. So, pile that
on the heap. What a lot of stuff!


Questions
---------

.. Topic:: Six honest serving-men

   | I keep six honest serving-men
   |   (They taught me all I knew);
   | Their names are What and Why and When
   |   And How and Where and Who.

   ~ Rudyark Kipling [1865-1936]

* Who

  * Who created NFS?
  * Who is responsible for upstream development of the Linux NFS
    server implementation -- both userspace and in the kernel?
  * Who are the interested parties in NFS continued existence?

* What

  * What are the major components of autofs?

    * ``autofs`` - kernel resident filesystem driver
    * ``automountd`` and autounmountd
    * ``automount`` - an administrative utility

  * What is the ``nfsd`` pseudo filesystem?
  * What systemd service units are associated with NFS shares?

    ::

      $ set nfs_packages nfs-utils nfs4-acl-tools portreserve quota-rpc
      $ for package in $nfs_packages; echo $package\n; rpm -ql $package | grep systemd; echo; end

      nfs-utils
      /usr/lib/systemd/system-generators/nfs-server-generator
                                         rpc-pipefs-generator
      /usr/lib/systemd/system/#### Services
                              auth-rpcgss-module.service
                              nfs-blkmap.service
                              nfs-convert.service
                              nfsdcld.service
                              nfs-idmapd.service
                              nfs-mountd.service
                              nfs-server.service # required for server
                              nfs-utils.service
                              rpc-gssd.service
                              rpc-statd-notify.service
                              rpc-statd.service
                              #### Targets
                              nfs-client.target # reccomended for client
                              rpc_pipefs.target
                              #### Mounts
                              proc-fs-nfsd.mount
                              var-lib-nfs-rpc_pipefs.mount
      /usr/share/man/man7/nfs.systemd.7.gz

      portreserve
      /usr/lib/systemd/system/portreserve.service

      quota-rpc
      /usr/lib/systemd/system/rpc-rquotad.service # enable this to provide quota services to NFSv3 clients

    The systemd man page for nfs says that the server needs ``nfs-server.service`` enabled, and that
    the client should have ``nfs-client.target`` enabled. I don't understand what ``nfs-client.target``
    actually does, though. On a different test setup, I was able to get the client working without
    enabling this target.

  * What processes are associated with NFS shares?


    * ``nfsd`` - nfs server kernel module

    * ``rpcbind`` - advertises rpc services and assigns them to ports

    * ``rpc.mountd`` - responds to MOUNT requests from NFSv3 clients

    * ``rpc.nfsd`` - The user-level part of the nfs server, started by
       nfs-server.service. Advertises explicit NFS protocol versions.

    * ``lockd`` - implements Network Lock Manager protocol, started whenever NFS server is run.

    * ``rpc.statd`` - Implements NSM RPC protocol. Notifies when sever crashes.

    * ``rpc.quotad`` - Process provides quota information for remote users.
       The rpc-quotad service, which is provided by the quota-rpc
       package, has to be started manually by the user when the nfs-server
       is started.

    * ``rpc.idmapd`` - Provides NFSv4 client and server upcalls, which map between
                    on-the-wire NFSv4 names (user@domain) and local UIDs and GIDs.

  * What ports are associated with NFS shares?

    ::

      nfs        2049/{tcp,udp}
      nfs4       2049/tcp
      mountd    20048/{tcp,udp}
      rpc-bind    111/{tcp,udp}
      rquotad     875/{tcp,udp}

    There are also *dynamically assigned ports*, set up by
    ``portmapper``. I don't know how that works.

  * What network protocols are associated with NFS shares?

    Here's what I've discovered so far

    * NFS (Network File System)
    * pNFS (parallel NFS)
    * NLM (Network Lock Manager)
    * NSM (Network State Manager)

  * What is RPC?
  * What does the ``portmap`` process do?
  * What existing SELinux policies are there for NFS?
  * What happens when the server goes down during a file operation?

* Where

  * Where is the code repository for upstream development?
  * Where is the documentation?
  * Where is the issue tracker?
  * Where can I find help?
  * Where do the developers chat online?

* When

  * When does it make sense to use NFSv3 vs NFSv4?

* Why

  * Why do many of the processes involved with NFS shares have a kernel component?
  * Why are multiple protocols involved, instead of just one?
  * Why did Sun decide to use RPC for the design of NFS?
  * Why hasn't NFS been replaced with something else?

* How

  * How do I set up a share on the server, and make it available to clients?
  * How do I configure the firewall to allow clients to connect?
  * How do I secure the share so that clients can't get access to the rest of the system?

    * Maybe this is a good use case for a ``podman`` container, run as a systemd unit?
    * Maybe there are SELinux settings related to NFS?

  * How do I set up authentication in a secure way?
  * How do I determine what component is failing when troubleshooting?



ULSAH Chapter 21: NFS
*********************
21.1 Meet network file services

* The competition

  SMB, SAN systems - block level access over a network, distributed filesystems: GlusterFS and Ceph.

* Issues of state

  * Stateful servers keep track of all open files across the network.
  * Statefulness lets clients maintain more control over files and facilitates the robust management
    of files open in r/w mode.
  * On a stateless server, each request is independent of the requests that have preceded it.
  * NFSv3 is stateless, NFSv4 is stateful.

* Performance concerns

  * NFS tries to minimize the number of network requests. Read-ahead caching.

* Security

  * You need a central authentication service. Hello Krb&LDAP, my dear old friends.
  * For privacy of traffic over a network, you need some kind of encrypted transport like a VPN, or SSH.
  * Many sites on a trusted LAN choose to forgo crypto entirely since it's a pain in the ass to implement.

21.2 The NFS approach

* Protocol versions and history

  * There are three generations of the NFS protocol v2, v3, and v4.
  * Use NFSv4 whenever possible.
  * The default configuration on RHEL is to use the latest version of v4, and then degrade to v3 if it isn't available.
  * When using NFSv3, the cooperation of other protocols is required to keep track of state information.

    * locking NLM
    * mounting NFS
    * status information NSM
    * quotas (???)

* Remote procedure calls

  * Operations that read and write files, mount filesystems, access file metadata, and check file
    permissions are all implemented as RPCs.
  * The NFS protocol spec is written generically, so a distinct RPC layer isn't needed, but all
    implementations seem to follow this architecture.

* Transport protocols

  * NFSv3 uses either TCP or UDP or both.
  * NFSv4 uses only TCP.

* State

  * NFSv3 is stateless. The server simply discloses a secret "cookie" at the conclusion of a
    successful mount negotiation, which identifies the mounted directories to the NFS server and so
    opens a way to access its contents.
  * NFSv4 is stateful, and does not need a cookie. The protocol keeps track of open file handles and
    other tracking information.

* Filesystem exports

  * In NFSv2 and NFSv3 each export is treated as an independent entity that is exported separately.
  * In NFSv4 all exports are incorporated into a single hierarchical psuedo-filesystem.

* File locking

  * Syscalls: ``flock``, ``lockf``, ``fcntl``
  * File locking is generally unreliable.
  * File locking is implemented as a separate protocol for NFSv3. The ``lockd`` and ``statd``
    daemons do this, using the NLM and NSM network protocols, respectively.
  * NFSv4 includes locking as part of the protocol. There are no separate daemon programs for it.

* Security concerns

  * AUTH_NONE, AUTH_SYS - unix style permissions and authentication using passwd or sssd, PRCSEC_GSS
    - an API over RPC that enables different back-ends to handle access control and authentication
      decisions. Facilitates Krb&LDAP. This configuration requires both the client and server to
      participate in a krb realm.

* Identity mapping in version 4

  * Don't use AUTH_SYS, it's a really bad idea.
  * Identity mapping plays no role in authentication or access control, it only translates username
    strings like user@nfs-domain into UID/GIDs that the client see.

* Root access and the nobody account

  * By default the NFS server intercepts incoming requests mad on behalf of UID 0 and changes them
    to look as if they cam from some other user.
  * A placeholder account named "nobody" (UID 65534) is used, instead.

* Performance consideration in version 4

21.3 Server-side NFS

  * In NFSv3 the process used by clients to mount a filesystem is separate from the process used to
    access files. Uses separate daemons and network protocols. ``rpc.mountd`` for mount discovery and
    requests, and ``rpc.nfsd`` for actual file service.
  * NFSv4 does not use ``rpc.mountd`` at all.
  * Active mounts are maintained in a binary files named ``/etc/xtab``.
  * ``/etc/exports`` is a plaintext configuration file that stores details about which directories
    should be exported over the network. It's contents can be listed using ``exportfs -a``.

* Linux exports

  * ``exports`` basic format: ``export_dir client_hostname_or_addr(fs_options)``. There can be
    multiple hosts-name/option pairs.
  * Filesystems listed in the ``exports`` file without a specific set of hosts are usually mountable
    by all machines.
  * This same security hole can be created by a misplaced space: ``/home *.users.admin.com (rw)``
    gives r/w access to *all hosts, except for those that match* ``*.user.admin.com``.
  * Maybe it makes sense to automate editing the ``exports`` file in such a way that this mistake is
    impossible to make -- perhaps using a template which is filled in based on some kind of wizard
    program.

21.4 Client-side NFS
21.5 Identity mapping for NFS version 4
21.6 ``nfsstat``: dump NFS statistics
21.7 Dedicated NFS file servers
21.8 Automatic mounting

  Three different kinds of configuration files, referred to as "maps":

  * master maps - lists the direct and indirect maps that
    automount should pay attention to
    
    * Require a HUP signal to be sent to the daemon to refresh their contents.
    * ``mount-point [map-type[,format]:]map [options]``

  * direct maps

    * Require a HUP signal to be sent to the daemon to refresh
      their contents.

  * indirect maps

    * Can be changed on the fly and the automounter will
      recognize those changes on the next operation it performs
      on a map.

  ``auomount -v`` useful for debugging
  ``auomount -L`` dry-run

21.9 Recommended reading
