..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Dynamic Routing for External Networks
=====================================

`Blueprint Link
<https://blueprints.launchpad.net/neutron/+spec/bgp-dynamic-routing>`_

The goal of this blueprint is to add a dynamic routing capability to OpenStack
deployment. This feature would allow Neutron to exchange routes with an external
router using standard routing protocols.

This document aims to define a foundation supporting different
route discovery/advertisement protocols. This blueprint will concentrate on BGP
implementation.


Problem description
===================
This a new feature and this section will describe use cases where dynamic
routing may be useful.


**Multi-homed OpenStack cloud**

Allow Neutron to dynamically announce and learn routes from external uplink
routers.

Sample Topology::

       +--------+         +--------+
       | Uplink |         | Uplink |
       | Router |         | Router |
       +--------+         +--------+
                 \       /
                   -----
                     |
                   _( )_
                 _(     )_
                ( External)
                 -( Net )-
   +--------+      -(_)-      +--------+
   | Tenant |_____/  |  \_____| Tenant |
   | Router |        |        | Router |
   +--------+    +--------+   +--------+
                 | Tenant |
                 | Router |
                 +--------+


A multi-homed OpenStack cloud would run a routing protocol (for example, BGP)
against at least one router in each uplink network provider. By announcing
external network hosting floating IP prefixes to those peers, the Neutron
network would be reachable by the rest of the internet via both paths. If the
link to an uplink provider broke, the failure information would propagate to
routers further up the stream, keeping the cloud reachable through the remaining
healthy link. Likewise, in such a case, Neutron would eliminate the routes
learned through the faulty link from its forwarding table, redirecting all
cloud-originated traffic through the healthy link.

In this scenario each external network's address range will be advertised as a
route. Individually allocated floating IP addresses will not be exported as host
routes to reduce the workload of the upstream router(s).

**Routed Model for Floating IPs on External Network**

This use case will allow an external network with a large public IP space to
possibly span more than one L2 network. This could allow for improved scale and
isolation between AZs while maintaining the appearance of a large external
network where floating IPs and networks can float freely.

This use case eliminates one reason for assigning a permanent IP from the
floating IP pool for each router. The router will be assigned an IP address on
a private routing network. There are still some reasons remaining to assign such
a permanent public IP. Default SNAT, DNS forwarding, and proxied metadata
requests are among other reasons. Some of these could be modified to also use a
private network but that is beyond the scope of this use case.

In the case of DVR, this model eliminates the need to allocate a public IP out
of the floating ip pool for each compute node.

This use case may also include announcing public networks behind Neutron routers
to an upstream router. It does not require learning routes from the upstream
router.

This use case is covered in detail in `Pluggable External Net Blueprint
<https://blueprints.launchpad.net/neutron/+spec/pluggable-ext-net>`_

Proposed change
===============


Overview
--------


Peer Configuration
++++++++++++++++++
A system that supports dynamic routing must be able to both advertise its own
routes to its peers as well as discover peers’ routes. It order to achieve this
the system must know the list of its peers and be able to trust the information
that it receives from them.

For Border Gateway Protocol (BGP), a peer address and session password are
required to establish a TCP connection to discover and share routing
information. This session may also be established without a password, however
routing information may not be properly authenticated.

A system may have multiple BGP peers. Typically each BGP peer will be connected
using a dedicated network interface. This interface must have IP connectivity
to its corresponding BGP peer. The BGP sessions are TCP connections and will be
established over the corresponding network interfaces. The routes advertised
by a BGP peer will have next hop port assigned to the network interface
associated with that peer.


Route Advertisement
+++++++++++++++++++

When enabled, the system will automatically advertise all external networks to
configured dynamic routing peers. Routes added manually by administrator will
have an option to be advertised to external dynamic routing peers.


Route Discovery
+++++++++++++++

Routes advertised by remote dynamic routing peers will be added to the routing
tables that manage forwarding from external networks defined in OpenStack to the
uplinks. The route information will show that it was installed dynamically and
will contain the ID of the dynamic routing peer that has advertised it.

The dynamically installed routes will also contain destination virtual port
information and relative weight. This additional configuration is useful when
the system has multiple uplinks to the same destination network. In such
scenario there may be 2 or more identical routes that may be differentiated by
the destination port and relative weight fields. The actual L3 agent or 3rd
party router implementation will be able to select the correct port to forward
packets to.


Dynamic Routing System
++++++++++++++++++++++

In default implementation a single system will be used to manage dynamic routing
information at the edge of OpenStack deployment. Dynamic peering will not be
performed by each Neutron router due to scaling concerns, as such approach may
create a high number of peering relationships.

This system would be responsible for advertising public OpenStack network ranges
to the upstream routers. It will also receive routes from the upstream routers.
This system doesn’t have to actually forward packets (this decision is left to
the plugin developers). In open source implementation, dynamic routing exchange
will be performed using an instance of Quagga (http://www.nongnu.org/quagga/).

Vendor specific implementation doesn't have to be limited to this model. For
example, a third party plugin may choose to support dynamic routing distributed
over 2 or more physical uplink servers for additional network redundancy.


IPv6 Considerations
+++++++++++++++++++

The implementation must be able to exchange IPv4 and IPv6 routes.


Alternatives
------------

Dynamic routing may be implemented using protocols other than BGP. This document
aims to create the framework to make it easier to add more dynamic routing
protocols in the future.


Data model impact
-----------------

This document proposes modifying data objects and schema in the following way.

Data Object Changes
+++++++++++++++++++

Two new data model classes will be added: ``db.l3_db.BgpPeer`` and
``db.l3_db.GatewaySpec``.

New ``db.l3_db.BgpPeer`` class will contain the following attributes:

* ``id``: UUID
* ``port``: UUID
* ``peer``: String
* ``password``: String
* ``local-as``: Integer
* ``remote-as``: Integer
* ``weight``: Integer

New ``db.l3_db.GatewaySpec`` class will contain the following attributes:

* ``id``: UUID
* ``networks``: List of network UUIDs
* ``routes``: List of ``db.models_v2.Route`` resources
* ``bgpenable``: Boolean
* ``bgpid``: String
* ``bgppeers``: List of ``db.l3_db.BgpPeer`` resources

In current Neutron implementation the route data object is defined in
``db.models_v2.Route`` class. The following attributes will be added to this
class:

* ``source``, String
* ``weight``, Integer
* ``action``, String
* ``port``, UUID
* ``dynamic``, Boolean
* ``advertise``, Boolean
* ``origin_type``, String
* ``origin_id``, UUID


Schema changes
++++++++++++++
A new resource type will be defined for storing route configuration. A
collection of these objects may be used to define a routing table in other
resources. This resource will be called ``Route`` and contain the following
attributes:

+------------+--------+-----+-----+---------+-----------+----------------------+
|Attribute   |Type    |Req  |CRUD |Default  |Validation |Notes                 |
|            |        |     |     |Value    |Constraints|                      |
+============+========+=====+=====+=========+===========+======================+
|id          |uuid-str|n/a  |R    |generated|n/a        |Unique Identifier for |
|            |        |     |     |         |           |route configration    |
+------------+--------+-----+-----+---------+-----------+----------------------+
|source      |CIDR    |N    |CRU  |0.0.0.0/0|n/a        |Value to compare with |
|            |        |     |     |         |           |the source IP address |
|            |        |     |     |         |           |of the flow being     |
|            |        |     |     |         |           |forwarded             |
+------------+--------+-----+-----+---------+-----------+----------------------+
|destination |CIDR    |N    |CRU  |0.0.0.0/0|n/a        |Value to compare with |
|            |        |     |     |         |           |the destination IP    |
|            |        |     |     |         |           |address of the flow   |
|            |        |     |     |         |           |being forwarded       |
+------------+--------+-----+-----+---------+-----------+----------------------+
|type        |string  |Y    |CRU  |forward  |forward,   |Action to apply to the|
|            |        |     |     |         |reject,    |flow when this route  |
|            |        |     |     |         |blackhole  |matches it            |
+------------+--------+-----+-----+---------+-----------+----------------------+
|weight      |integer |N    |CRU  |0        |Positive   |If multiple routes    |
|            |        |     |     |         |integer    |match a flow, use     |
|            |        |     |     |         |           |weight to             |
|            |        |     |     |         |           |preferentially select |
|            |        |     |     |         |           |the route to apply    |
+------------+--------+-----+-----+---------+-----------+----------------------+
|nexthop     |IP      |N    |CRUD |None     |Routable IP|IP address of the next|
|            |address |     |     |         |address    |hop                   |
+------------+--------+-----+-----+---------+-----------+----------------------+
|nexthop-port|uuid    |N    |CRUD |None     |Valid port |UUID of the next hop  |
|            |        |     |     |         |UUID       |port                  |
+------------+--------+-----+-----+---------+-----------+----------------------+
|advertise   |boolean |N    |CRU  |false    |n/a        |True if the route     |
|            |        |     |     |         |           |should be advertised  |
|            |        |     |     |         |           |when dynamic routing  |
|            |        |     |     |         |           |is enabled            |
+------------+--------+-----+-----+---------+-----------+----------------------+
|dynamic     |boolean |N    |CRU  |false    |n/a        |True if the route was |
|            |        |     |     |         |           |installed             |
|            |        |     |     |         |           |automatically by      |
|            |        |     |     |         |           |dynamic routing       |
+------------+--------+-----+-----+---------+-----------+----------------------+
|origin-type |string  |N    |CRUD |none     |none,      |Name of the system (if|
|            |        |     |     |         |bgppeer    |any) that created this|
|            |        |     |     |         |           |route                 |
+------------+--------+-----+-----+---------+-----------+----------------------+
|origin-id   |uuid-str|N    |CRUD |none     |UUID of    |If the route was      |
|            |        |     |     |         |dynamic    |installed dynamically,|
|            |        |     |     |         |routing    |store the identifier  |
|            |        |     |     |         |object     |of the entity that has|
|            |        |     |     |         |           |offered this          |
|            |        |     |     |         |           |route. For example,   |
|            |        |     |     |         |           |``id`` of BgpPeer     |
|            |        |     |     |         |           |resource              |
+------------+--------+-----+-----+---------+-----------+----------------------+


A new resource type will be defined for BGP configuration. It will be called
``BgpPeer`` and contain the following attributes:

+---------+---------+-----+-----+---------+----------------+-------------------+
|Attribute|Type     |Req  |CRUD |Default  |Validation      |Notes              |
|         |         |     |     |Value    |Constraints     |                   |
+=========+=========+=====+=====+=========+================+===================+
|id       |uuid-str |n/a  |R    |generated|n/a             |Unique identifier  |
|         |         |     |     |         |                |for BGP Peer       |
|         |         |     |     |         |                |configuration      |
+---------+---------+-----+-----+---------+----------------+-------------------+
|port     |uuid-str |Y    |CRU  |n/a      |Port with       |Routes discovered  |
|         |         |     |     |         |specified UUID  |using this BGP peer|
|         |         |     |     |         |must exist      |will be applied to |
|         |         |     |     |         |                |this port          |
+---------+---------+-----+-----+---------+----------------+-------------------+
|peer     |IP       |Y    |CRY  |n/a      |Valid IP address|BGP peer to        |
|         |address  |     |     |         |reachable using |exchange routes    |
|         |         |     |     |         |the port        |with               |
|         |         |     |     |         |associated with |                   |
|         |         |     |     |         |this resource   |                   |
+---------+---------+-----+-----+---------+----------------+-------------------+
|password |string   |N    |CUD  |n/a      |                |Password string    |
|         |         |     |     |         |                |used to            |
|         |         |     |     |         |                |authenticate with  |
|         |         |     |     |         |                |the remote peer    |
+---------+---------+-----+-----+---------+----------------+-------------------+
|local-as |integer  |Y    |CRU  |n/a      |0-65535         |Local Anonymous    |
|         |         |     |     |         |                |System number      |
+---------+---------+-----+-----+---------+----------------+-------------------+
|remote-as|integer  |Y    |CRU  |n/a      |0-65535         |Autonomous System  |
|         |         |     |     |         |                |number of the      |
|         |         |     |     |         |                |remote peer        |
+---------+---------+-----+-----+---------+----------------+-------------------+
|weight   |integer  |N    |CRUD |0        |n/a             |Weight to assign to|
|         |         |     |     |         |                |routes received    |
|         |         |     |     |         |                |from the peer      |
+---------+---------+-----+-----+---------+----------------+-------------------+


A new resource will be defined to describe how to forward traffic from external
Neutron networks to upstream provider routers. It will be called GatewaySpec and
contain the following attributes:

+---------+--------+-----+-----+---------+-----------+-------------------------+
|Attribute|Type    |Req  |CRUD |Default  |Validation |Notes                    |
|         |        |     |     |Value    |Constraints|                         |
+=========+========+=====+=====+=========+===========+=========================+
|id       |uuid-str|n/a  |R    |generated|n/a        |Unique identifier for    |
|         |        |     |     |         |           |external network gateway |
|         |        |     |     |         |           |configuration            |
+---------+--------+-----+-----+---------+-----------+-------------------------+
|networks |list of |N    |CRUD |none     |Valid      |List of networks that are|
|         |uuid-str|     |     |         |network    |connected to this        |
|         |        |     |     |         |identifiers|resource. When dynamic   |
|         |        |     |     |         |           |routing is enabled,      |
|         |        |     |     |         |           |connected networks will  |
|         |        |     |     |         |           |be advertised to remote  |
|         |        |     |     |         |           |peers                    |
+---------+--------+-----+-----+---------+-----------+-------------------------+
|routes   |list of |N    |CRUD |none     |n/a        |Routing table for this   |
|         |Route   |     |     |         |           |gateway specification.   |
|         |        |     |     |         |           |Route resources may be   |
|         |        |     |     |         |           |added/removed manually or|
|         |        |     |     |         |           |by dynamic routing system|
+---------+--------+-----+-----+---------+-----------+-------------------------+
|bgpenable|boolean |N    |RU   |False    |           |Enable or disable BGP    |
|         |        |     |     |         |           |dynamic routing. When    |
|         |        |     |     |         |           |this attribute is        |
|         |        |     |     |         |           |"False", BGP peer        |
|         |        |     |     |         |           |configuration is ignored |
+---------+--------+-----+-----+---------+-----------+-------------------------+
|bgpid    |IP      |N    |RU   |0.0.0.0  |Must be    |Unique BGP identifier of |
|         |address |     |     |         |IPv4       |this virtual router. Note|
|         |        |     |     |         |address    |that this field is not an|
|         |        |     |     |         |           |actual IP address used in|
|         |        |     |     |         |           |packet forwarding        |
+---------+--------+-----+-----+---------+-----------+-------------------------+
|bgppeers |list of |N    |RU   |None     |n/a        |List of BGP peers        |
|         |BgpPeer |     |     |         |           |                         |
+---------+--------+-----+-----+---------+-----------+-------------------------+


REST API impact
---------------


Route Settings
++++++++++++++
Route resources are used to define a routing table for L3 devices in Neutron
network. Therefore, it is assumed that there will always be a device that owns a
route resource. For example, a routing table may be present in external gateway
specification. All API calls in Route section shall contain a device prefix.

Example of device prefix for gateway specification resource: ::

  /gateway/{gateway_id}/

The routing table configuration for this gateway specification may be accessed
at: ::

  /gateway/{gateway_id}/routes

The following Neutron API changes will allow an administrator to configure
routing table.


List Routes, Show Routes
************************

+-----+----------------------------+-------------------------------------------+
|Verb |URI                         |Description                                |
+=====+============================+===========================================+
|GET  |/{device_prefix}/routes     |Get the list of routes                     |
+-----+----------------------------+-------------------------------------------+
|GET  |/{device_prefix}/routes/{id}|Show the configuration for the specified   |
|     |                            |route                                      |
+-----+----------------------------+-------------------------------------------+

Response Codes:

* 200: Normal
* 401: Unauthorized
* 403: Forbidden
* 404: Not Found

On success a response will contain one or more route objects (JSON format used
in the example). On failure, the response will contain empty body. ::

 {
  “routes”:
  [{
     “id”: “cb29d21a-f334-4a79-a2a4-a08fc65672fb”,
     “source”: “0.0.0.0/0”,
     “destination”: “0.0.0.0/0”,
     “weight”: 1,
     “type”: “ACCEPT”,
     “nexthop”: “10.1.1.3”,
     “nexthop-port”: “a629ad80-87ff-4e1e-b060-ceb7425dd1cd”,
     “advertise”: “false”,
     “dynamic”: “true”,
     “origin-type”: “bgppeer”,
     “origin-id”: “c1bcd3d8-2f02-4d05-8283-ff87ae962223”
   },
   {
     “id”: “cb29d21a-f334-4a79-a2a4-a08fc65672fb”,
     “source”: “0.0.0.0/0”,
     “destination”: “192.168.128.0/24”,
     “weight”: 1,
     “type”: “ACCEPT”,
     “nexthop”: “10.1.7.3”,
     “nexthop-port”: “d957523d-ba9a-4d6f-914d-c2206d6dec55”,
     “advertise”: “false”,
     “dynamic”: “false”,
     “origin-type”: “”,
     “origin-id”: “”
   }]
 }


Create Route
************

+-----+----------------------------+-------------------------------------------+
|Verb |URI                         |Description                                |
+=====+============================+===========================================+
|POST |/{device_prefix}/routes     |Create a new route                         |
+-----+----------------------------+-------------------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden
* 404: Not Found

This operation requires a request body and returns a response body. Both contain
Route object. A JSON-encoded example is provided:

Request: ::

 {
  “route”:
  {
    “source”: “192.168.64.0/24”,
    “destination”: “0.0.0.0/0”,
    “weight”: 1,
    “type”: “REJECT”,
  }
 }

Response: ::

 {
  “route”:
  {
    “id”: “6b4d97b9-6fd3-4d3f-9ec4-44ce6503360d”,
    “source”: “0.0.0.0/0”,
    “source”: “192.168.64.0/24”,
    “destination”: “0.0.0.0/0”,
    “weight”: 1,
    “type”: “REJECT”,
    “nexthop-port”: “”,
    “advertise”: “false”,
    “dynamic”: “false”,
    “origin-type”: “”,
    “origin-id”: “”
  }
 }


Update Route
************

+-----+----------------------------+-------------------------------------------+
|Verb |URI                         |Description                                |
+=====+============================+===========================================+
|PUT  |/{device_prefix}/routes/{id}|Update route                               |
+-----+----------------------------+-------------------------------------------+

Response Codes:

* 200: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden
* 404: Not Found

This operation requires a request body and returns a response body. Both contain
Route object. A JSON-encoded example is provided:

Request: ::

 {
  “route”:
  {
    “id”: “6b4d97b9-6fd3-4d3f-9ec4-44ce6503360d”,
    “type”: “BLACKHOLE”
  }
 }

Response: ::

 {
  “route”:
  {
    “id”: “6b4d97b9-6fd3-4d3f-9ec4-44ce6503360d”,
    “source”: “0.0.0.0/0”,
    “source”: “192.168.64.0/24”,
    “destination”: “0.0.0.0/0”,
    “weight”: 1,
    “type”: “BLACKHOLE”,
    “nexthop-port”: “”,
    “advertise”: “false”,
    “dynamic”: “false”,
    “origin-type”: “”,
    “origin-id”: “”
  }
 }


Delete Route
************

+-------+-----------------------------+----------------------------------------+
|Verb   |URI                          |Description                             |
+=======+=============================+========================================+
|DELETE |/{device_prefix}/routes/{id} |Delete route                            |
+-------+-----------------------------+----------------------------------------+

Response Codes:

* 204: Normal
* 401: Unauthorized
* 404: Not Found

This operation does not require request body and does not provide response body.


BGP Peer Settings
+++++++++++++++++

The following Neutron API changes will allow an administrator to configure BGP
peers.


List BGP Peers, Show BGP Peer
*****************************

+-----+------------------------------------+-----------------------------------+
|Verb |URI                                 |Description                        |
+=====+====================================+===================================+
|GET  |/gateways/{gateway_id}/bgppeers     |Get the list of all BGP peers      |
|     |                                    |associated with the specified      |
|     |                                    |gateway configuration              |
+-----+------------------------------------+-----------------------------------+
|GET  |/gateways/{gateway_id}/bgppeers/{id}|Show the confiugration for BGP peer|
|     |                                    |with the spcified id               |
+-----+------------------------------------+-----------------------------------+

Response Codes:

* 200: Normal
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator tries to access
  configuration)
* 404: Not Found (for example, when the BGP peer with the specified ID doesn’t
  exist)

On success a response will contain one or more BGP peer objects (JSON format
used in the example). On failure, the response will contain empty body. Note
that “password” field is not included in the output. The API may only be used to
create/update/delete password field, not to display it. Password will be
provided to the plugin code in the clear text. ::

 {
  “bgppeers”:
  [{
     “id”: “88d2dbf0-35e5-11e3-aa6e-0800200c9a66”,
     “port”: “dcb9172a-2966-4a15-bb9b-d669c54dc39b”,
     “peer”: “10.32.0.2”,
     “local-as”: “65000”,
     “remote-as”: “65001”,
     “weight”: “1”,
   },
   {
     “id”: “40edaac2-881c-457b-9b4f-05bcd8510d28”,
     “port”: “033656e4-cca5-41d2-9101-0f308c063d29”,
     “peer”: “10.32.0.3”,
     “local-as”: “65000”,
     “remote-as”: “65002”,
     “weight”: “2”
   }]
 }


Create BGP Peer
***************

+-----+-------------------------------+----------------------------------------+
|Verb |Uri                            |Description                             |
+=====+===============================+========================================+
|POST |/gateways/{gateway_id}/bgppeers|Create a new BGP peer configuration for |
|     |                               |the specified gateway configuration     |
+-----+-------------------------------+----------------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator user tries to create BGP
  configuration)
* 404: Not Found (for example, when the router with the specified id does not
  exist)

This operation requires a request body and returns a response body. Both contain
BgpPeer object. A JSON-encoded example is provided.

Request: ::

 {
  “bgppeer”:
  {
    “port”: “a629ad80-87ff-4e1e-b060-ceb7425dd1cd”,
    “peer”: “10.32.0.17”,
    “password”: “secret”,
    “local-as”: “65000”,
    “remote-as”: “65005”,
  }
 }

Response: ::

 {
  “bgppeer”:
  {
    “id”: “c1bcd3d8-2f02-4d05-8283-ff87ae962223”,
    “port”: “a629ad80-87ff-4e1e-b060-ceb7425dd1cd”,
    “peer”: “10.32.0.17”,
    “local-as”: “65000”,
    “remote-as”: “65005”,
    “weight”: “2147483647”,
  }
 }


Update BGP Peer
***************

+-----+------------------------------------+-----------------------------------+
|Verb |Uri                                 |Description                        |
+=====+====================================+===================================+
|PUT  |/gateways/{gateway_id}/bgppeers/{id}|Update BGP peer configuration      |
+-----+------------------------------------+-----------------------------------+

Response Codes:

* 200: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator tries to update BGP
  configuration)
* 404: Not Found (for example, when the BGP peer with the specified id does not
  exist)

This operation requires a request body and returns a response body. Both contain
BgpPeer object. A JSON-encoded example is provided.

Request: ::

 {
  “bgppeer”:
  {
    “id”: “c1bcd3d8-2f02-4d05-8283-ff87ae962223”,
    “remote-as”: “65006”
  }
 }

Response: ::

 {
  “bgppeer”:
  {
    “id”: “c1bcd3d8-2f02-4d05-8283-ff87ae962223”,
    “peer”: “10.32.0.17”,
    “port”: “a629ad80-87ff-4e1e-b060-ceb7425dd1cd”,
    “local-as”: “65000”,
    “remote-as”: “65006”,
    “weight”: “2147483647”
  }
 }


Delete BGP Peer
***************

+-------+------------------------------------+---------------------------------+
|Verb   |Uri                                 |Description                      |
+=======+====================================+=================================+
|DELETE |/gateways/{gateway_id}/bgppeers/{id}|Delete BGP peer configuration    |
+-------+------------------------------------+---------------------------------+

Response Codes:

* 204: Normal
* 401: Unauthorized
* 404: Not Found (for example, if the BGP peer with the specified id does not
  exist)

This operation does not require request body and does not provide response body.

GatewaySpec Settings
++++++++++++++++++++

The following Neutron API changes will allow an administrator to configure
gateway specification to connect OpenStack deployment to outside network.

List GatewaySpec, Show GatewaySpec
**********************************

+-----+--------------+---------------------------------------------------------+
|Verb |URI           |Description                                              |
+=====+==============+=========================================================+
|GET  |/gateways     |Get the list of all gateway specifications               |
+-----+--------------+---------------------------------------------------------+
|GET  |/gateways/{id}|Show gateway specification                               |
+-----+--------------+---------------------------------------------------------+

Response Codes:

* 200: Normal
* 401: Unauthorized
* 403: Forbidden (for example, when non-administrator tries to access
  configuration)
* 404: Not Found

On success a response will contain one or more GatewaySpec objects (JSON format
used in the example). On failure, the response will contain empty body. ::

 {
  “gateways”:
  [{
     “id”: “821e2ba8-b7da-43f5-8dc8-5f70ded64cca”
     “networks”: [“e9b07a5b-5456-4202-857e-5c58f03c8833”,
                  “1eea5739-51b6-49ae-bca4-21cd5150d9b3”],
     “routes”: [],
     “bgpenable”: “true”,
     “bgpid”: “1.1.1.1”,
     “bgppeers”: []
   }]
 }

Create GatewaySpec
******************

+-----+---------+--------------------------------------------------------------+
|Verb |URI      |Description                                                   |
+=====+=========+==============================================================+
|POST |/gateways|Create a new gateway specification                            |
+-----+---------+--------------------------------------------------------------+

Response Codes:

* 201: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden

This operation requires a request body and returns a response body. Both contain
GatewaySpec object. A JSON-encoded example is provided.

Request: ::

 {
  “gateway”:
  {
    “networks”: [“92fcf0c6-c6da-416b-8d45-34e9dae9e608”],
    “bgpenable”: “true”
    “bgpid”: “1.1.1.2”
  }
 }

Response: ::

 {
  “gateway”:
  {
    “id”: “d1804e12-3a05-46bc-a73a-3875473ddc43”
    “networks”: [“e9b07a5b-5456-4202-857e-5c58f03c8833”]
    “routes”: [],
    “bgpenable”: “true”,
    “bgpid”: “1.1.1.2”,
    “bgppeers”: []
  }
 }


Update GatewaySpec
******************

+-----+--------------+---------------------------------------------------------+
|Verb |URI           |Description                                              |
+=====+==============+=========================================================+
|PUT  |/gateways/{id}|Update GatewaySpec configuration                         |
+-----+--------------+---------------------------------------------------------+

Response Codes:

* 200: Normal
* 400: Bad Request (for example, invalid request format)
* 401: Unauthorized
* 403: Forbidden
* 404: Not Found

This operation requires a request body and returns a response body. Both contain
GatewaySpec object. A JSON-encoded example is provided.

Request: ::

 {
  “gateway”:
  {
    “id”: “d1804e12-3a05-46bc-a73a-3875473ddc43”,
    “bgpid”: “1.1.1.3”
  }
 }

Response: ::

 {
  “gateway”:
  {
    “id”: “d1804e12-3a05-46bc-a73a-3875473ddc43”
    “networks”: [“e9b07a5b-5456-4202-857e-5c58f03c8833”]
    “routes”: [],
    “bgpenable”: “true”,
    “bgpid”: “1.1.1.3”,
    “bgppeers”: []
  }
 }


Delete BGP Peer
***************

+------+--------------+---------------------------------------------------------+
|Verb  |URI           |Description                                              |
+======+==============+=========================================================+
|DELETE|/gateways/{id}|Delete GatewaySpec configuration                         |
+------+--------------+---------------------------------------------------------+

Response Codes:

* 204: Normal
* 401: Unauthorized
* 404: Not Found

This operation does not require request body and does not provide response body.


Security impact
---------------

This feature will allow an external system to manipulate routing information
within Neutron network. The external system should be trusted and may be
authenticated using a shared secret.

Dynamic routing may only be configured by the system administrator.


Notifications impact
--------------------

A notification should be provided when connectivity of control channel over
which routes are exchanged is interrupted


Other end user impact
---------------------

The following CLI commands will be added to manage routing table:

* *route-list(gateway)*: List configured routes for the specified gateway
* *route-show(gateway, id)*: Show details route configuration for the specified
  route
* *route-create(gateway, [source], [destination], [weight], [type], [nexthop],
  [nexthop-port], [advertise], [dynamic])*: Create new route
* *route-update(gateway, id, [source], [destination], [weight], [type],
  [nexthop], [nexthop-port], [advertise], [dynamic])*: Update route
  configuration
* *route-delete(gateway, id)*: Delete route

The following CLI commands will be added to manage BGP peer configuration:

* *bgp-peer-list(gateway)*: List configured BGP peers for the specified gateway
* *bgp-peer-show(gateway, id)*: Show detailed BGP configuration for the
  specified peer
* *bgp-peer-create(gateway, peer, [password], local-as, remote-as, port,
  [weight])*: Create a new BGP peer configuration
* *bgp-peer-update(id, [gateway], [peer], [password], [local-as], [remote-as],
  [port], [weight])*: Update existing BGP peer configuration
* *bgp-peer-delete(gateway, id)*: Delete BGP configuration for the specified
  peer

The following CLI commands will be added to manage Gateway specification for
connecting OpenStack to outside networks:

* *gateway-list*: List configured gateways
* *gateway-show(id)*: Show detailed gateway configuration
* *gateway-create([networks], [routes], [bgpenable], [bgpid], [bgppeers])*:
  Create new gateway specification
* *gateway-update(id, [networks], [routes], [bgpenable], [bgpid], [bgppeers])*:
  Update gateway specification
* *gateway-delete(id)*: Delete gateway specification

The CLI commands to create/update/delete Neutron networks will have an extra
parameter. This parameter is called “gateway-spec”. This parameter may only be
specified for external networks and by admin user. This parameter will take a
UUID identifying corresponding GatewaySpec object.


Horizon Requirements
++++++++++++++++++++

A new screen will be added to configure gateway configuration for connecting
OpenStack to outside networks. This screen will allow routes and BGP
configuration to be added to gateway configuration.

An external network will have an option to be linked to gateway configuration.


Usage Example
+++++++++++++
Configure 2 uplinks for the gateway specification serving an external network.
When one uplink fails the traffic is automatically re-routed to the second
uplink. This example assumes “admin-router” connected to public network exists
and has 2 uplink ports.

Sample configuration using Neutron CLI commands: ::

  neutron gateway-create --name ext-gateway --bgpenable true \
    --bgpid 1.1.1.1

  neutron bgp-peer-create --gateway ext-gateway --peer 10.10.0.15 \
    --password secretsession --local-as 65010 --remote-as 65001 \
    --port [UUID of uplink port 1] --weight 1

  neutron bgp-peer-create --gateway ext-gateway --peer 10.10.0.16 \
    --password secretsession --local-as 65010 --remote-as 65002 \
    --port [UUID of uplink port 2] --weight 2

  neutron net-update --name external --gateway ext-gateway


Performance Impact
------------------

This feature describes an out of band mechanism to negotiate routing
configuration. This feature should not have a performance impact on Neutron
network.


Other deployer impact
---------------------

This feature would have to explicitly enabled and configured before it will take
effect. There are no changes to configuration files.


Developer impact
----------------

The plugin will have access to classes described in the “Data model changes”
section. It may choose to ignore data stored in the new objects if BGP dynamic
routing is not supported. When BGP dynamic routing is supported, the new data
objects will contain BGP configuration. The plugin will have an option to create
and delete routes automatically and mark them as dynamic routes.

No changes are required from the plugins if BGP dynamic routing is not
supported. When a plugin vendor would like to implement dynamic routing using
BGP they will take advantage of the new data model to obtain the configuration
necessary to establish BGP sessions. The actual BGP implementation is vendor
specific. Open source BGP implementations are available and may be adopted.


Implementation
==============


Assignee(s)
-----------

This is a pre-liminary contributor list

Primary assignee:
  nextone92

Other contributors:
  devvesa


Work Items
----------


Dependencies
============

This feature will depend on a BGP daemon to exchange route information. This
daemon could be Quagga (http://www.nongnu.org/quagga/).


Testing
=======

Dynamic routing testing may be performed in an isolated environment. An external
autonomous system may be simulated with an instance of BGP capable software
router (for example, quagga).

The following dynamic routing scenarios could be tested:

Verify that when BGP is enabled on the gateway and one peer is configured the
gateway establishes BGP session with the peer, receives a list of routes, and
submits advertised routes to the peer.

Verify that when BGP is disabled on the gateway and one peer is configured the
gateway establishes no BGP sessions

Verify that when BGP is enabled on the gateway and 3 BGP peers are configured,
the gateway establishes 3 BGP sessions, one to each of the configured peers.

When 2 or more peers are configured, verify that BGP implementation is able to
detect when the BGP session is interrupted the routes received from that BGP
session are automatically removed from the routing table


Documentation Impact
====================

What is the impact on the docs team of this change? Some changes might require
donating resources to the docs team to have the documentation updated. Don't
repeat details discussed above, but please reference them here.


References
==========

* `Neutron Dynamic Routing Use Cases
  <https://wiki.openstack.org/wiki/Neutron/DynamicRoutingUseCases>`_
* `Pluggable External Net Blueprint
  <https://blueprints.launchpad.net/neutron/+spec/pluggable-ext-net>`_
* `Border Gateway Protocol
  <http://en.wikipedia.org/wiki/Border_Gateway_Protocol>`_
* `Quagga <http://www.nongnu.org/quagga/>`_
