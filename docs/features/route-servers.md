# Route Servers

> **Prerequisite Reading:** Ensure you first familiarize yourself with [the generic documentation on managing and generating router configurations here](routers.md).

Normally on a peering exchange, all connected parties will establish bilateral peering relationships with each other customer connected to the exchange. As the number of connected parties increases, it becomes increasingly more difficult to manage peering relationships with customers of the exchange. A typical peering exchange full-mesh eBGP configuration might look something similar to the diagram on the left hand side.

![Route Servers Diagram](img/rs-diagram.png)

The full-mesh BGP session relationship scenario requires that each BGP speaker configure and manage BGP sessions to every other BGP speaker on the exchange. In this example, a full-mesh setup requires 7 BGP sessions per member router, and this increases every time a new member connects to the exchange.

However, by using a route servers for peering relationships, the number of BGP sessions per router stays at two: one for each route server (assuming a resilient set up). Clearly this is a more sustainable way of maintaining IXP peering relationships with a large number of participants.

## Configuration Generation Features

Please [review the generic router documentation](routers.md) to learn how to automatically generate route server configurations. This section goes into a bit more specific detail on INEX's route server configuration (as shipped with IXP Manager) and why it's safe to use.

You should also look at the following resources:

*  watch the *IXP Manager Update & Route Server Configuration* presentation from the 32nd Euro-IX Forum held in Galway, Ireland, April 2018: [[pdf](https://www.ixpmanager.org/media/2018/20180417-euroix-ixpmanager.pdf)] [[video](https://www.ixpmanager.org/videos/201804-euroix.mp4)].
* to fully understand RPKI with IXP Manager, [watch our presentation](https://www.inex.ie/inex-news/shiny-new-route-servers/) from APRICOT 2019.
*  [read this article]( https://youtu.be/cqhJwuBaxxQ?t=1549) on INEX's website.

The features of the route server configurations that IXP Manager generates include:

* [RPKI support](rpki.md) when using the Bird v2 templates;
* full prefix filtering based on IRRDB entries (can be disabled on a per member basis if required) - see [the IRRDB documentation here](irrdb.md);
* full origin ASN filtering based on IRRDB entries (can be disabled on a per member basis if required);
* in all cases, prefix filtering for IPv4 and v6 based on the IANA special purpose registries (also known as bogon lists);
* ensuring next hop is the neighbor address to ensure no next hop hijacking;
* max prefix limits;
* multiple VLAN interfaces for a single member supported;
* large BGP communities supported;
* over a decade of production use and experience.

With Bird v2 support in IXP Manager v5, we provide better looking glass integration and other tooling to show members which prefixes are filtered and why. The Bird v2 filtering mechanism is as follows:

1. Filter small prefixes (default is > /24 / /48 for ipv4 / ipv6).
2. Filter martians / bogons prefixes ([see this template](https://github.com/inex/IXP-Manager/blob/master/resources/views/api/v4/router/server/bird2/header.foil.php)).
3. Sanity check - filter prefixes with no AS path or > 64 ASNs in AS path.
4. Sanity check to ensure peer AS is the same as first AS in the prefix’s AS path.
5. Prevent next-hop hijacking. This occurs when a participant advertises a prefix with a next hop IP other than their own. An exception exists to allow participants with multiple connections advertise their other router (next-hop within the same AS).
6. Filter known transit networks - see FIXME
7. IRRDB filtering: ensure origin AS is in set of ASNs from member AS-SET.
8. RPKI:
    * Valid -> accept
    * Invalid -> drop
9. RPKI Unknown -> revert to standard IRRDB prefix filtering.


## Setting Up

You first need to add your route servers to the **IXP Manager** routers database. See [this page on how to do that](routers.md).

Typically an IXP's route server service will have a dedicated ASN that is different to the IXP's own management / route collector ASN. As such, you need to add a new *internal* customer to IXP Manager.

**You are strongly advised to use / request a 16-bit ASN from your RIR for route server use. If you do not, you will be unable to offer your members standard community based filtering.**

Here's an example from INEX for our *route server #1*:

![Route Servers INEX Customer](img/rs-inex-customer.png)

You then need to create an interface for this route server on each peering LAN where the service will be offered. Here again is INEX's example from our peering LAN1 in Dublin:

![Route Servers INEX VLAN Interface Entry](img/rs-inex-vli.png)

There's a couple things to note in the above:

1. *AS112 Client* is checked which means (so long as *Route Server Client* is checked on the [AS112 service](as112.md)) the AS112 service will peer with the route servers.
2. *Apply IRRDB Filtering* has no meaning here as this is the route server rather than the route server client.


## Per ASN Import / Export Filters

There are occasions where you may need to override the default filtering mechanism for some members. IXP Manager allows you to create custom Bird2 checks at the **start** of the standard import / export filters when using **Bird2** (not supported on the older Bird v1 configuration).

To do this, you must [create skinned files](skinning.md) named after the ASN. For example, let's assume your skin name is `example` and the ASN of the member you want to apply custom filtering to is `65500`; then you would an export and/or import filter in files named:

* `$IXPROOT/resources/skins/example/api/v4/router/server/bird2/f_import_as65500.foil.php`
* `$IXPROOT/resources/skins/example/api/v4/router/server/bird2/f_export_as65500.foil.php`

You'll see [real examples from INEX here](https://github.com/inex/IXP-Manager/tree/master/resources/skins/inex/api/v4/router/server/bird2). Remember that these are placed at the *beginning of the standard filters* allowing you to explicitly `accept` or `reject` the prefix. However, remember that INEX `accepts` prefixes on import always but tags prefixes for filtering with large community `routerasn:1101:x` - please see the resources referenced above for details on this.


## Displaying Filtered Prefixes

Using Bird v2 and internal large communities, we have completely overhauled how we show end users what prefixes are filtered by the route servers.

If you are running route servers using the Bird v2 configuration and if you have installed [the looking glass](looking-glass.md) then you should set the following in your `.env` file:

```
IXP_FE_FRONTEND_DISABLED_FILTERED_PREFIXES=false
```

![Route Servers Filtered Prefixes](img/rs-filtered-prefixes.png)

This is a live view gathered from each Bird v2 route server with a looking glass.


### Legacy Prefix Analysis Tool

The older but deprecated means of viewing filtered prefixes was the *Route Server Prefix Analysis tool* which allows your members to examine what routes they are advertising to the route servers, which are being accepted and which are being rejected.

#### Limits / Caveats

Implemented as a Perl script which accesses the database directly. The script can also only be used on one LAN and one route server. Thus, pick you most popular LAN and route server.

#### Setting Up


* Download [this script](https://github.com/inex/IXP-Manager/blob/master/tools/runtime/route-servers/compare-route-server-prefixes.pl) to your route server;
* Ensure you install and configure the IXP Manager Perl library on this route server also;
* Edit the `compare-route-server-prefixes.pl` script and set the Bird configuration file and socket (you can find the position in the file by searching for `XXX-SET-ME`);
* See also [this diff](https://www.inex.ie/pipermail/ixpmanager/2015-May/000558.html) for further hints;
* Execute the `compare-route-server-prefixes.pl` script a few times a day as you think appropriate;
* In your `.env` file, ensure the following is set:

```ini
IXP_FE_FRONTEND_DISABLED_RS_PREFIXES=false
```

Once you make the last change above, the prefix analysis tool will be available to administrators and members on IXP Manager.
