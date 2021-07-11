<script type="text/javascript" src="https://cdn.jsdelivr.net/gh/hellt/drawio-js@main/embed2.js" async></script>
As promised in the [introduction](intro.md), we are about to start configuring the EVPN service in our fabric to let the servers talk to each other as if they were on the same Layer 2 network. But prior to configuring EVPN based overlay, a routing protocol needs to be configured in the fabric to dynamically discover and advertise the reachability of all the leaf VXLAN Termination End Point (VTEP) addresses throughout the IP fabric.

With SR Linux, the following routing protocols can be used in the underlay: 

* ISIS
* OSPF
* EBGP

We will use a BGP based fabric design as described in [RFC7938](https://tools.ietf.org/html/rfc7938) due to its simplicity, scalability, and ease of multi-vendor interoperability.

## Leaf-Spine interfaces
Let's start with configuring the IP interfaces on the inter-switch links to ensure L3 connectivity is established. According to our lab topology configuration, and using the `192.168.xx.0/30` network to address the links, we will move towards the following underlay addressing design:

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;margin:0 auto; display:block;" data-mxgraph="{&quot;page&quot;:2,&quot;zoom&quot;:2,&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;check-visible-state&quot;:true,&quot;resize&quot;:true,&quot;url&quot;:&quot;https://raw.githubusercontent.com/learn-srlinux/site/diagrams/quickstart.drawio&quot;}"></div>

On each leaf and spine we will bring up the relevant [interface](ifaces.md) and address its routed [subinterface](ifaces.md#subinterfaces) to achieve L3 connectivity.

We begin with connecting to the CLI of our nodes via SSH[^1]:

```bash
# connecting to leaf1
ssh admin@clab-quickstart-leaf1
```

Then on each node we enter into [candidate configuration mode](cfgmgmt.md#configuration-modes) and proceed with the relevant interfaces configuration.

Let's see the step by step process of an interface configuration on leaf1 with providing the paste-ables snippets for the rest of the nodes

1. Enter the candidate configuration mode to make edits
    ```
    Welcome to the srlinux CLI.
    Type 'help' (and press <ENTER>) if you need any help using this.


    --{ running }--[  ]--
    A:leaf1# enter candidate
    ```
2. The prompt will indicate the changed active mode
    ```
    --{ candidate shared default }--[  ]--                                                                     
    A:leaf1#                                                                                                   
    ```
3. Enter into the interface configuration context
    ```
    --{ candidate shared default }--[  ]--                                                                     
    A:leaf1# interface ethernet-1/49                                                                           
    ```
4. Create a subinterface under the parent interface to configure IPv4 address on it
    ```
    --{ * candidate shared default }--[ interface ethernet-1/49 ]--                                            
    A:leaf1# subinterface 0                                                                                    
    --{ * candidate shared default }--[ interface ethernet-1/49 subinterface 0 ]--                             
    A:leaf1# ipv4 address 192.168.11.1/30                                                                      
    ```
5. Now apply the changes by issuing a `commit now` command
    ```
    --{ * candidate shared default }--[ interface ethernet-1/49 subinterface 0 ipv4 address 192.168.11.1/30 ]--
    A:leaf1# commit now                                                                                        
    All changes have been committed. Leaving candidate mode.
    ```

Below you will find the relevant configuration snippets[^2] for leafs and spine of our fabric which you can paste in the terminal while being in candidate mode.

=== "leaf1"
    ```
    interface ethernet-1/49 {
        subinterface 0 {
            ipv4 {
                address 192.168.11.1/30 {
                }
            }
        }
    }
    ```

=== "leaf2"
    ```
    interface ethernet-1/49 {
        subinterface 0 {
            ipv4 {
                address 192.168.12.1/30 {
                }
            }
        }
    }
    ```
=== "spine1"
    ```
    interface ethernet-1/1 {
        subinterface 0 {
            ipv4 {
                address 192.168.11.2/30 {
                }
            }
        }
    }
    interface ethernet-1/2 {
        subinterface 0 {
            ipv4 {
                address 192.168.12.2/30 {
                }
            }
        }
    }
    ```

Once those snippets are committed to the running configuration with `commit now` command, we can ensure that the changes have been applied:

```
--{ + running }--[  ]--                             
A:spine1# show interface ethernet-1/1               
====================================================
ethernet-1/1 is up, speed 10G, type None
  ethernet-1/1.0 is up
    Network-instance: 
    Encapsulation   : null
    Type            : routed
    IPv4 addr    : 192.168.11.2/30 (static, None)
----------------------------------------------------
====================================================
```

At this moment, the configured interfaces can not be used as they are not yet associated with any [network instance](netwinstance.md). Below we are placing the interfaces to the network-instance `default` that is created automatically created by SR Linux.

=== "leaf1 & leaf2"
    ```
    --{ + candidate shared default }--[  ]--                                                   
    A:leaf1# network-instance default interface ethernet-1/49.0                                
    
    --{ +* candidate shared default }--[ network-instance default interface ethernet-1/49.0 ]--
    A:leaf1# commit now                                                                        
    All changes have been committed. Leaving candidate mode.
    ```

=== "spine1"
    ```
    --{ + candidate shared default }--[  ]--                                                   
    A:spine1# network-instance default interface ethernet-1/1.0                               
    
    --{ +* candidate shared default }--[ network-instance default interface ethernet-1/1.0 ]--
    A:spine1# /network-instance default interface ethernet-1/2.0                               
    
    --{ +* candidate shared default }--[ network-instance default interface ethernet-1/2.0 ]--
    A:spine2# commit now                                                                       
    All changes have been committed. Leaving candidate mode.
    ```

When interfaces are owned by the network-instance default, we can ensure that the basic IP connectivity is working by issuing a ping between the pair of interfaces. For example from `spine1` to `leaf2`:

```
--{ + running }--[  ]--                                     
A:spine1# ping 192.168.12.1 network-instance default        
Using network instance default
PING 192.168.12.1 (192.168.12.1) 56(84) bytes of data.
64 bytes from 192.168.12.1: icmp_seq=1 ttl=64 time=31.4 ms
64 bytes from 192.168.12.1: icmp_seq=2 ttl=64 time=10.0 ms
64 bytes from 192.168.12.1: icmp_seq=3 ttl=64 time=13.1 ms
64 bytes from 192.168.12.1: icmp_seq=4 ttl=64 time=16.5 ms
^C
--- 192.168.12.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 10.034/17.786/31.409/8.199 ms
```

## EBGP
Since in this exercise the design decision was to use BGP in the data center, we need to configure EBGP peering for the leaf-spine pairs. The EBGP will make sure of advertising the VTEPs across the fabric. The VTEPs will be configured later, in this step we will take care of adding the eBGP peering.

Using the following diagram with ASN/Router ID allocation let's turn this into working configuration:

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;margin:0 auto; display:block;" data-mxgraph="{&quot;page&quot;:3,&quot;zoom&quot;:2,&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;check-visible-state&quot;:true,&quot;resize&quot;:true,&quot;url&quot;:&quot;https://raw.githubusercontent.com/learn-srlinux/site/diagrams/quickstart.drawio&quot;}"></div>

Here is a breakdown of the steps that are needed to configure eBGP on `leaf1` towards `spine1`:

1. **Add BGP protocol to network-instance**  
    Routing protocols are configured under a network-instance context. By adding BGP protocol to the default network-instance we implicitly enable this protocol.  
    ```
    --{ + candidate shared default }--[  ]--       
    A:leaf1# network-instance default protocols bgp
    ```

1. **Assign ASN**  
    The ASN reported to peers when this network-instance opens a BGP session toward another router (unless it is overridden by a local-as configuration).  
    According to the diagram above, `leaf1` has ASN 101.  
    ```
    --{ +* candidate shared default }--[ network-instance default protocols bgp ]--
    A:leaf1# autonomous-system 101
    ```

1. **Assign Router ID**  
    This is the BGP identifier reported to peers when this network-instance opens a BGP session toward another router. This overrides the router-id configuration at the network-instance level.  
    Leaf1 has a router-id of 10.0.0.1.
    ```
    --{ +* candidate shared default }--[ network-instance default protocols bgp ]--
    A:leaf1# router-id 10.0.0.1
    ```

1. **Enable AF**  
    Enable all address families that should be enabled globally as a default for all peers of the BGP instance.  
    When you later configure individual neighbors or groups, you can override the enabled families at those levels.
    ```
    --{ +* candidate shared default }--[ network-instance default protocols bgp ]--
    A:leaf1# ipv4-unicast admin-state enable
    ```

1. **Create export/import policies**  
    The export/import policy is required for a eBGP peer to advertise and install routes.  
    The policy named `all` that we create below will be used both as an import and export policy, effectively allowing all routes to be advertised and received.  

    The routing policies are configured at `/routing-policy` context, so first, we switch to it from the current `bgp` context:
    ```
    --{ * candidate shared default }--[ network-instance default protocols bgp ]--
    A:leaf1# /routing-policy                                                      

    --{ * candidate shared default }--[ routing-policy ]--                        
    A:leaf1#
    ```
    Now that we are in the right context, we can copy paste the policy definition:
    ```
    --{ +* candidate shared default }--[ routing-policy ]--
    A:leaf1# info
        policy all {
            default-action {
                accept {
                }
            }
        }
    ```

1. **Create peer-group config**  
    A peer group should include sessions that have a similar or almost identical configuration.  
    In this example, the peer group is named `eBGP-underlay` since it will be used to enable underlay routing between the leafs and spines.  
    New groups are administratively enabled by default.

    First, we come back to our bgp from the routing-policy context:
    ```
    --{ * candidate shared default }--[ routing-policy ]--
    A:leaf1# /network-instance default protocols bgp      

    --{ * candidate shared default }--[ network-instance default protocols bgp ]--
    A:leaf1#
    ```
    Now create the peer group. The common group configuration includes the `peer-as` and `export-policy` statements.
    ```
    --{ +* candidate shared default }--[ network-instance default protocols bgp ]--
    A:leaf1# group eBGP-underlay

    --{ +* candidate shared default }--[ network-instance default protocols bgp group eBGP-underlay ]--
    A:leaf1# peer-as 201                                                                               

    --{ +* candidate shared default }--[ network-instance default protocols bgp group eBGP-underlay ]--
    A:leaf1# export-policy all

    --{ +* candidate shared default }--[ network-instance default protocols bgp group eBGP-underlay ]--
    A:leaf1# import-policy all
    ```

1. **Configure neighbor**  
    Configure the BGP session with `spine1`. In this example, `spine1` is reachable through the `ethernet-1/49.0` subinterface. On this subnet, `spine1` has the IPv4 address `192.168.11.2`.  
    In this minimal configuration example, the only required configuration for the neighbor is its association with the group `eBGP-underlay` that was previously created.  
    New neighbors are administratively enabled by default.
    ```
    --{ +* candidate shared default }--[ network-instance default protocols bgp ]--
    A:leaf1# neighbor 192.168.11.2 peer-group eBGP-underlay
    ```

2. **Commit configuration**  
    It seems like the EBGP config has been sorted out. Let's see what we have in our candidate datastore so far.  
    Regardless of which context you are currently in, you can see the diff against the baseline config by doing `diff /`
    ```
    --{ * candidate shared default }--[ network-instance default protocols bgp group eBGP-underlay ]--
    A:leaf1# diff /                                                                                   
        network-instance default {
            protocols {
    +             bgp {
    +                 autonomous-system 101
    +                 router-id 10.0.0.1
    +                 group eBGP-underlay {
    +                     export-policy all
    +                     import-policy all
    +                     peer-as 201
    +                 }
    +                 ipv4-unicast {
    +                     admin-state enable
    +                 }
    +                 neighbor 192.168.11.2 {
    +                     peer-group eBGP-underlay
    +                 }
    +             }
            }
        }
    +     routing-policy {
    +         policy all {
    +             default-action {
    +                 accept {
    +                 }
    +             }
    +         }
    +     }
    ```
    That is what we've added in all those steps above, everything looks OK, so we are good to commit the configuration.
    ```
    --{ +* candidate shared default }--[ network-instance default protocols bgp ]--
    A:leaf1# commit now
    ```

EBGP configuration on `leaf2` and `spine1` is almost a twin from the one we did for `leaf1`. Here is a copy-paste-able[^3] config snippets for all of the nodes:

=== "leaf1"
    ```
    network-instance default {
        protocols {
            bgp {
                autonomous-system 101
                router-id 10.0.0.1
                group eBGP-underlay {
                    export-policy all
                    import-policy all
                    peer-as 201
                }
                ipv4-unicast {
                    admin-state enable
                }
                neighbor 192.168.11.2 {
                    peer-group eBGP-underlay
                }
            }
        }
    }
    routing-policy {
        policy pass-all-bgp {
            default-action {
                accept {
                }
            }
        }
    }
    ```
=== "leaf2"
    ```
    network-instance default {
        protocols {
            bgp {
                autonomous-system 102
                router-id 10.0.0.2
                group eBGP-underlay {
                    export-policy all
                    import-policy all
                    peer-as 201
                }
                ipv4-unicast {
                    admin-state enable
                }
                neighbor 192.168.12.2 {
                    peer-group eBGP-underlay
                }
            }
        }
    }
    routing-policy {
        policy pass-all-bgp {
            default-action {
                accept {
                }
            }
        }
    }
    ```
=== "spine1"
    Spine configuration is a bit different, in a way that `peer-as` is specified under the neighbor context, and not the group one.
    ```
    network-instance default {
        protocols {
            bgp {
                autonomous-system 201
                router-id 10.0.1.1
                group eBGP-underlay {
                    export-policy all
                    import-policy all
                }
                ipv4-unicast {
                    admin-state enable
                }
                neighbor 192.168.11.1 {
                    peer-group eBGP-underlay
                    peer-as 101
                }
                neighbor 192.168.12.1 {
                    peer-group eBGP-underlay
                    peer-as 102
                }
            }
        }
    }
    routing-policy {
        policy pass-all-bgp {
            default-action {
                accept {
                }
            }
        }
    }
    ```

## Loopbacks
As we will create a BGP based EVPN control plane at a later stage, we need to configure loopback addresses for our leaf devices so that they can build an iBGP peering over those interfaces.

In the context of the VXLAN data plane, a special kind of a loopback needs to be created - [`system0`](ifaces.md#system) interface.

!!!info
    The `system0.0` interface hosts the loopback address used to originate and typically
    terminate VXLAN packets. This address is also used by default as the next-hop of all
    EVPN routes.

Configuration of the `system0` interface is exactly the same as for the regular interfaces. The IPv4 addresses we assign to `system0` interfaces will match the Router-ID of a given BGP speaker.

=== "leaf1"
    ```
    /interface system0 {
        admin-state enable
        subinterface 0 {
            ipv4 {
                address 10.0.0.1/32 {
                }
            }
        }
    }
    /network-instance default {
        interface system0.0 {
        }
    }
    ```
=== "leaf2"
    ```
    /interface system0 {
        admin-state enable
        subinterface 0 {
            ipv4 {
                address 10.0.0.2/32 {
                }
            }
        }
    }
    /network-instance default {
        interface system0.0 {
        }
    }
    ```
=== "spine1"
    ```
    /interface system0 {
        admin-state enable
        subinterface 0 {
            ipv4 {
                address 10.0.1.1/32 {
                }
            }
        }
    }
    /network-instance default {
        interface system0.0 {
        }
    }
    ```

## Verification
As stated in the beginning of this section, the VXLAN VTEPs need to be advertised throughout the DC fabric. The `system0` interfaces we just configured are the VTEPs and they should be advertised via EBGP peering established before. The following verification commands can help ensure that.

### BGP status
The first thing worth verifying is that BGP protocol is enabled and operational on all devices. Below is an example of a summary command issued on `leaf1`:

``` linenums="1"
--{ + running }--[  ]--
A:leaf1# show network-instance default protocols bgp summary
-------------------------------------------------------------
BGP is enabled and up in network-instance "default"
Global AS number  : 101
BGP identifier    : 10.0.0.1
-------------------------------------------------------------
  Total paths               : 3
  Received routes           : 3
  Received and active routes: None
  Total UP peers            : 1
  Configured peers          : 1, 0 are disabled
  Dynamic peers             : None
-------------------------------------------------------------
Default preferences
  BGP Local Preference attribute: 100
  EBGP route-table preference   : 170
  IBGP route-table preference   : 170
-------------------------------------------------------------
Wait for FIB install to advertise: True
Send rapid withdrawals           : disabled
-------------------------------------------------------------
Ipv4-unicast AFI/SAFI
    Received routes               : 3
    Received and active routes    : None
    Max number of multipaths      : 1, 1
    Multipath can transit multi AS: True
-------------------------------------------------------------
Ipv6-unicast AFI/SAFI
    Received routes               : None
    Received and active routes    : None
    Max number of multipaths      : 1,1
    Multipath can transit multi AS: True
-------------------------------------------------------------
EVPN-unicast AFI/SAFI
    Received routes               : None
    Received and active routes    : None
    Max number of multipaths      : N/A
    Multipath can transit multi AS: N/A
-------------------------------------------------------------
```

### BGP neighbor status
Equally important is the neighbor summary status that we can observe like that:

```
--{ + running }--[  ]--
A:spine1# show network-instance default protocols bgp neighbor
----------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
----------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------
+----------------+-----------------------+----------------+------+---------+-------------+-------------+-----------+-----------------------+
|    Net-Inst    |         Peer          |     Group      | Flag | Peer-AS |    State    |   Uptime    | AFI/SAFI  |    [Rx/Active/Tx]     |
|                |                       |                |  s   |         |             |             |           |                       |
+================+=======================+================+======+=========+=============+=============+===========+=======================+
| default        | 192.168.11.1          | eBGP-underlay  | S    | 101     | established | 0d:18h:20m: | ipv4-unic | [2/1/4]               |
|                |                       |                |      |         |             | 49s         | ast       |                       |
| default        | 192.168.12.1          | eBGP-underlay  | S    | 102     | established | 0d:18h:20m: | ipv4-unic | [2/1/4]               |
|                |                       |                |      |         |             | 9s          | ast       |                       |
+----------------+-----------------------+----------------+------+---------+-------------+-------------+-----------+-----------------------+
----------------------------------------------------------------------------------------------------------------------------------------------
Summary:
2 configured neighbors, 2 configured sessions are established,0 disabled peers
0 dynamic peers
```

### Received/Advertised routes
The reason we configured eBGP in the fabric's the underlay is to advertise the VXLAN tunnel endpoints - `system0` interfaces. To check that our leafs advertise their system interfaces and receive routes with them:

In the example below we verify that `leaf1` advertise the prefix of `system0` (`10.0.0.1/32`) interface towards its EBGP `spine1` peer:
```
--{ + running }--[  ]--
A:leaf1# show network-instance default protocols bgp neighbor 192.168.11.2 advertised-rou
tes ipv4
-----------------------------------------------------------------------------------------
Peer        : 192.168.11.2, remote AS: 201, local AS: 101
Type        : static
Description : None
Group       : eBGP-underlay
-----------------------------------------------------------------------------------------
Origin codes: i=IGP, e=EGP, ?=incomplete
+-------------------------------------------------------------------------------------+
|    Network        Next Hop       MED     LocPref           AsPath           Origin  |
+=====================================================================================+
| 10.0.0.1/32      192.168.11.      -        100     [101]                       i    |
|                  1                                                                  |
| 192.168.11.0/3   192.168.11.      -        100     [101]                       i    |
| 0                1                                                                  |
+-------------------------------------------------------------------------------------+
-----------------------------------------------------------------------------------------
2 advertised BGP routes
-----------------------------------------------------------------------------------------
```

On the far end of the fabric, `leaf2` receives both the `leaf1` and `spine1` system interface prefixes:

```
--{ + running }--[  ]--
A:leaf2# show network-instance default protocols bgp neighbor 192.168.12.2 received-route
s ipv4
-----------------------------------------------------------------------------------------
Peer        : 192.168.12.2, remote AS: 201, local AS: 102
Type        : static
Description : None
Group       : eBGP-underlay
-----------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale
Origin codes: i=IGP, e=EGP, ?=incomplete
+-----------------------------------------------------------------------------------+
|  Status      Network    Next Hop       MED       LocPref     AsPath      Origin   |
+===================================================================================+
|    u*>      10.0.0.1/   192.168.1       -          100      [201,           i     |
|             32          2.2                                 101]                  |
|    u*>      10.0.1.1/   192.168.1       -          100      [201]           i     |
|             32          2.2                                                       |
|    u*>      192.168.1   192.168.1       -          100      [201]           i     |
|             1.0/30      2.2                                                       |
|     *       192.168.1   192.168.1       -          100      [201]           i     |
|             2.0/30      2.2                                                       |
+-----------------------------------------------------------------------------------+
-----------------------------------------------------------------------------------------
4 received BGP routes : 3 used 4 valid
-----------------------------------------------------------------------------------------
```

### Route table
The last stop in the control plane verification ride would be to check if the remote loopback prefixes were installed in the `default` network-instance where we expect them to be:

``` linenums="1" hl_lines="12 15"
--{ running }--[  ]--
A:leaf1# show network-instance default route-table ipv4-unicast summary
-----------------------------------------------------------------------------------------------------------------------------------
IPv4 Unicast route table of network instance default
-----------------------------------------------------------------------------------------------------------------------------------
+-----------------+-------+------------+----------------------+----------------------+----------+---------+-----------+-----------+
|     Prefix      |  ID   | Route Type |     Route Owner      |      Best/Fib-       |  Metric  |  Pref   | Next-hop  | Next-hop  |
|                 |       |            |                      |     status(slot)     |          |         |  (Type)   | Interface |
+=================+=======+============+======================+======================+==========+=========+===========+===========+
| 10.0.0.1/32     | 3     | host       | net_inst_mgr         | True/success         | 0        | 0       | None      | None      |
|                 |       |            |                      |                      |          |         | (extract) |           |
| 10.0.0.2/32     | 0     | bgp        | bgp_mgr              | True/success         | 0        | 170     | 192.168.1 | None      |
|                 |       |            |                      |                      |          |         | 1.2 (indi |           |
|                 |       |            |                      |                      |          |         | rect)     |           |
| 10.0.1.1/32     | 0     | bgp        | bgp_mgr              | True/success         | 0        | 170     | 192.168.1 | None      |
|                 |       |            |                      |                      |          |         | 1.2 (indi |           |
|                 |       |            |                      |                      |          |         | rect)     |           |
| 192.168.11.0/30 | 1     | local      | net_inst_mgr         | True/success         | 0        | 0       | 192.168.1 | ethernet- |
|                 |       |            |                      |                      |          |         | 1.1       | 1/49.0    |
|                 |       |            |                      |                      |          |         | (direct)  |           |
| 192.168.11.1/32 | 1     | host       | net_inst_mgr         | True/success         | 0        | 0       | None      | None      |
|                 |       |            |                      |                      |          |         | (extract) |           |
| 192.168.11.3/32 | 1     | host       | net_inst_mgr         | True/success         | 0        | 0       | None (bro | None      |
|                 |       |            |                      |                      |          |         | adcast)   |           |
| 192.168.12.0/30 | 0     | bgp        | bgp_mgr              | True/success         | 0        | 170     | 192.168.1 | None      |
|                 |       |            |                      |                      |          |         | 1.2 (indi |           |
|                 |       |            |                      |                      |          |         | rect)     |           |
+-----------------+-------+------------+----------------------+----------------------+----------+---------+-----------+-----------+
-----------------------------------------------------------------------------------------------------------------------------------
7 IPv4 routes total
7 IPv4 prefixes with active routes
0 IPv4 prefixes with active ECMP routes
-----------------------------------------------------------------------------------------------------------------------------------
```

Both `leaf2` and `spine1` prefixes are found in the route table of network-instance `default` and the `bgp_mgr` is the owner of those prefixes, which means that they have been added to the route-table by the BGP app.

### Dataplane
To finish the verification process let's ensure that the datapath is indeed working, and the VTEPs on both leafs can reach each other via the routed fabric underlay.

For that we will leverage the `ping` command with src/dst pair to use loopback addresses:

```
--{ running }--[  ]--
A:leaf1# ping -I 10.0.0.1 network-instance default 10.0.0.2
Using network instance default
PING 10.0.0.2 (10.0.0.2) from 10.0.0.1 : 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=63 time=17.5 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=63 time=12.2 ms
```

Now the fabric underlay is properly configured, and we can proceed with EVPN service configuration!

[^1]: default SR Linux credentials are `admin:admin`.
[^2]: the snippets were extracted with `info interface ethernet-1/x` command issued in running mode.
[^3]: you can paste those snippets right after you do `enter candidate`