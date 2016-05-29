Python Core Concepts
====================

.. contents:: :local:

Bootstrapping a switch
----------------------

There are a number of ways to bootstrap the necessary components on to a switch, and automatically load the minimal, initial configuration.  We strongly suggest `ZTP Server`_ to automate the steps from initial power-on to contacting the Puppet master.

Sample minimal configuration on a switch includes basic IP connectivity, hostname and domain-name which are used to generate the switch's SSL certificate, a name-server or host entry for "puppet", the default master name unless otherwise specified, and enabling eAPI (management api http-commands):

.. code-block:: console

  !
  hostname my-switch
  ip domain-name example.com
  !
  ip name-server vrf default 8.8.8.8
  ! OR
  ip host puppet 192.2.2.5
  !
  interface Management1
     ip address 192.2.2.101/24
     no shutdown
  !
  ip route 0.0.0.0/0 192.2.2.1
  !
