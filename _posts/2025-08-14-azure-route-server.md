---
layout: default
title: "Azure Route Server: the simple way to make Azure routing behave"
date: 2025-08-14 10:00:00 +0100
categories: azure networking
permalink: /azure-route-server/
---

<div class="page-content">

<p align="center">
  <img src="/Azure-Blog/assets/images/DC-Networking.webp" width="500" alt="Happy engineer with Microsoft hoodie in a data hall with cabling" />
</p>

### A quick hello

I am really enjoying writing this series. It forces me to go deeper, test ideas properly, and explain them in a way that helps the next person. Personally, I love Azure Route Server. It is one of the best tools in the box, often misunderstood and definitely under‑utilised. By the end of this post, I hope you will see why, it is in my professional view is that there is very little reason not to deploy it in any hub that exchanges routes with NVAs or gateways.

---

## 1. From UDR pain to Route Server possibilities

We left off last time wading through the swamp of User Defined Routes, the quiet little workhorses that keep Azure traffic flowing where you want it to go. They are the digital equivalent of hand painted road signs. Reliable when placed correctly. Disastrous when one gets spun the wrong way in the wind. One wrong entry and secure traffic goes on a world tour, a hybrid link gets lost, or production falls over because a route decided to take the day off.

The worst part is that Azure gives you no native way to centrally administer all those route tables. Each one sits in its own silo. It is like being caretaker of a hundred remote lighthouses. You visit by hand, change the bulb, and hope you did not miss one.

Wouldn’t it be nice to keep the precision of UDRs without the lighthouse tour? If the network could update itself when topology changes? That is where **Azure Route Server** enters. It does not bulldoze your design. It takes over the grind and brings proper dynamic routing to the cloud world you assumed was already dynamic.

---

## 2. The black box of Azure’s fabric

Azure’s fabric is a huge software defined network. Your next hop might live in another rack or even another zone. There is no console cable to follow and no chassis LEDs to stare at. You inject intent, and the fabric delivers.

Security, resilience, and scale still need your intent. That is why we place deliberate signs with UDRs and why dynamic routing helps. You define inspection points, isolation boundaries, and failover paths. The platform handles the heavy lifting.

---

## 3. What Route Server is and what it does

The biggest misconception is that Route Server sits in the data path. It does not. **Route Server is control plane only.** No packets flow through it.

What it actually does:

- Establishes **BGP** sessions with your peers: NVAs, VPN Gateways, ExpressRoute Gateways.  
- **Learns** routes from them and **injects** those routes into the VNet routing fabric.  
- **Advertises** Azure‑known routes back to those peers so everyone shares the same map.  
- Withdraws routes when a peer fails, which shifts traffic without you touching UDRs.

A very useful side effect is that many designs can retire load balancers that previously sat in front of NVAs only to force failover. With BGP and, where supported, ECMP, the platform can prefer or share across NVAs naturally.

<p align="center">
  <img src="/Azure-Blog/assets/images/route-injection.png" width="780" alt="Route Server with eBGP to NVA influencing private traffic for spokes" />
</p>

---

## 4. BGP, the heart of Route Server

Let us first be clear about BGP. It is the routing protocol that keeps the internet working. A short while ago, someone told me, “BGP is a fad and will never catch on.” That comment said a lot about how widely BGP is misunderstood.

BGP does not ship packets. It tells devices what **paths** exist to reach networks, with attributes like AS Path, Local Preference, and MED. Route Server uses BGP to keep NVAs and gateways in sync with Azure’s view of the world. If a path disappears, the route is withdrawn and traffic shifts to another path.

What BGP is not: a firewall, encryption, or a magic fix for a poor design. You still need a clean IP plan, security policy, and a sensible topology.

---

## 5. Design patterns and integration scenarios

**Hub‑and‑spoke with NVAs in the hub**  
NVAs peer with Route Server. They advertise the protected prefixes. Spokes learn the path through the hub without manually editing UDRs. Failover is driven by route withdrawal.

<p align="center">
  <img src="/Azure-Blog/assets/images/route-injection-vpn.png" width="780" alt="Adding site-to-site/IPsec to the hub and Route Server design" />
</p>

**Hybrid connectivity with ExpressRoute and VPN**  
Prefer ExpressRoute while it is up. If it drops, the VPN path activates without you touching the spokes. Steering is policy, not manual table edits.

**Central firewall service‑chaining**  
Keep inspection consistent. NVAs or Azure Firewall can advertise themselves as the next hop for targeted prefixes. Route adds or removals update the chain automatically.

<p align="center">
  <img src="/Azure-Blog/assets/images/route-injection-vpn-expressroute-firewall.png" width="720" alt="Firewall steering with ER and VPN, controlled by Route Server" />
</p>

**Multi‑region**  
Deploy **one Route Server per hub**. Peer it locally with that hub’s NVAs and gateways. Exchange only the prefixes that must be global.

<p align="center">
  <img src="/Azure-Blog/assets/images/multiregion-with-expressroute.png" width="900" alt="Two‑region hub and spoke with Global VNet Peering and Route Server" />
</p>

<p align="center">
  <img src="/Azure-Blog/assets/images/ARS-multiregion.png" width="900" alt="Per‑region Route Server pattern" />
</p>

> There will be a separate deep dive on Azure Virtual WAN. Much of the good behaviour described here is built into vWAN hubs already.

---

## 6. Limits you should actually care about

Per current Microsoft documentation:

- Max **BGP peers per Route Server**: **8**
- Max **routes a single peer can advertise**: **1,000** (can be raised to **4,000** via a support case)
- Max **VMs across the connected VNets**: **4,000**
- Max **supported VNets**: **500**
- Max **total on‑prem and VNet prefixes** supported: **10,000**

If an NVA advertises more routes than the limit, the BGP session drops. Plan your summarisation and filtering accordingly.

---

## 7. FAQ items that matter in the real world

**Why does Route Server need public IPs with open ports**  
Those endpoints are for Azure’s SDN and management plane to reach the service. Authentication is certificate based and audited. Route Server does not expose your VNet.

**Can Route Server filter routes from NVAs**  
It supports the **NO_ADVERTISE** BGP community. If an NVA tags a route with NO_ADVERTISE, Route Server will not pass it on to other peers, including the ExpressRoute gateway. This helps keep route volume under control.

**If two NVAs advertise the same route, what happens**  
If the AS Path length is the same, Route Server programs multiple copies with different next hops into the VM hosts and the hosts use **ECMP**. If one NVA has a shorter AS Path, that one wins.

**Do I need BGP enabled on the VPN Gateway**  
No. Route Server can still peer with NVAs. You do not have to enable BGP on the VPN gateway to use Route Server.

**Can I associate a UDR with the RouteServerSubnet**  
No. The Route Server subnet is not for data traffic. It is control plane only.

---

## 8. Security and operational best practice

- **Zero Trust mindset** for routing. Authenticate every BGP session with MD5. Store keys in Key Vault.  
- **Prefix filtering both ways.** Only accept and advertise what you intend. Be very careful with 0.0.0.0/0.  
- **Monitoring.** Alert on session drops, sudden route count changes, and the appearance of default routes from unexpected peers.  
- **Blast radius.** Keep peering **per hub** and **per trust zone.** Do not build a single Route Server that peers with everything everywhere.  
- **Runbooks.** Document your expected baseline and test failovers.

A fictional financial services customer once restored an NVA from an old backup that started advertising a default route with a better path. Traffic swung to the wrong box within seconds. Proper MD5 auth and prefix filters would have rejected the session and prevented the incident.

---

## 9. Why it is under‑utilised

- Teams are comfortable with static UDRs until scale makes them brittle.  
- Misconception that Route Server is in the data path. It is not.  
- Uncertainty about multi‑region placement. The simple rule is a Route Server **in each hub that exchanges routes**. Keep BGP neighbours close.

**Example**  
Take a **(fictional)** global customer headquartered in UK South with peering to hubs in APAC and the Americas. Each hub has its own NVAs, its own Route Server, and its own gateway. Prefixes that need to be global are shared across regions. Everything else stays local. Convergence is fast, failover is clean, and there are far fewer moving parts than a large UDR estate with load balancers.

---

## 10. Future outlook

Microsoft does not publish exact dates and this is a **personal opinion** as I do not have visibility to the internal roadmap, but likely directions include:

- More policy control on the Route Server resource itself  
- Better multi‑hub orchestration and analytics  
- Tighter integration with Azure Firewall Policy and Network Manager  
- Continued IPv6 parity improvements

Design with that trajectory in mind and you will be ready to adopt new capabilities as they land.

---

## 11. Summary and decision guide

**Use Route Server if** you have NVAs or gateways that need dynamic route exchange, want to reduce UDR sprawl, remove load balancers that only exist for failover, or need predictable convergence across hubs and regions.

**Skip it if** the environment is small and static, where the operational overhead outweighs the benefit.

---

## Closing thought

In a Zero Trust, multi‑region, hybrid‑connected world, the value of Route Server is not just convenience. It is resilience. It lets you spend more time designing where the network should go and less time fixing how it gets there.

If you have followed from the days of hand‑editing UDRs into the world of BGP and dynamic exchange, you have seen how Route Server turns routing from a scatter of isolated notes into a steady rhythm section. It simplifies designs, strengthens failover, sharpens security, and opens the door to architectures that would otherwise be fragile.

Think back to the lighthouse keeper changing bulbs on lonely rocks. Route Server is the moment the lights stay in sync on their own. You focus on the bigger picture with confidence that the network will adapt when life happens.

</div>
