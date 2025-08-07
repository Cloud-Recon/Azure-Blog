---
layout: post
title: "2025-08-07 - Life Before Azure Route Server"
---

Before we get into Azure Route Server, let’s talk about something that throws nearly every on-premises network engineer off balance, especially if, like me, you come from a Cisco background and the first time you step into Azure: **the never-ending tangle of route tables**, a world that seems a backward step in networking.

In a traditional world, routes were simple. You had a router. It had interfaces. You knew exactly which side of the network a packet came from and where it needed to go. Routing was often automatic, mostly physical, and rarely touched unless something broke or a new site came online. It had its quirks, it *made sense*.

Then you land in Azure and discover you need to build **User Defined Routes (UDRs)** for what feels like… everything.

1. If you want traffic from a spoke to reach your firewall = UDR.
2. You want to steer traffic through a NVA = UDR.
3. You want to override a default internet route = UDR.
4. You want to peer a VNet and keep it secure = UDR.
5. If you’re using Virtual WAN or ExpressRoute or Private Link, guess what… UDR.

At first glance, it feels like we’ve gone backwards. All this software-defined magic and we’re still **hand-crafting route tables like it’s 2005**. But there are good reasons for it, and understanding them is the key to designing Azure networks properly.

So why does Azure need UDRs? Why is this “manual routing” still a thing?

---

## 1. Microsoft’s Global Network Isn’t a Flat LAN

<img src="{{ site.baseurl }}/assets/images/GlobalNetwork.PNG" alt="Microsoft Global Network Map" style="max-width:100%; height:auto; display:block; margin: 0 auto;" />

Azure is not your old on-prem datacentre. It’s a hyperscale global platform running over one of the largest private WANs in the world. Because Microsoft optimises traffic across regions, across POPs, and even between services like Front Door and Application Gateway, **packets can and will take unexpected routes unless you explicitly define the path**.

This is why you have to tell Azure what you want traffic to do. It doesn’t make assumptions like a traditional router. Instead, it gives you the flexibility—and the responsibility—to define behaviour.

---

## 2. The Azure Fabric is Built for Scale, Not Simplicity

Azure’s fabric is dynamic. Resources scale up and down, VNets come and go, IPs are allocated and reassigned. The platform needs a way to make sense of that chaos, and UDRs are part of the solution.

They allow you to segment traffic, build security zones, and push packets through inspection points like Azure Firewall, NVAs, or service endpoints.

But without UDRs? That traffic just takes the shortest internal path Azure sees fit, and that might bypass your firewall, logging, or inspection layers entirely.

---

## 3. Hybrid Networks Break Flat Design Assumptions

When you plug in on-premises networks via VPN or ExpressRoute, Azure doesn’t automatically treat them as just another subnet.

It’s a whole different routing domain. If you want your spoke VNets to talk to your ExpressRoute gateway, or force all internet traffic through your on-prem security stack, **UDRs are the glue**.

Without them, there’s no deterministic path.

---

## 4. UDRs Enable Policy-Based Design

Want to allow east-west traffic between VNets but force north-south to go through a firewall?

Want to isolate tiers of an app for PCI compliance?

Want to direct inspection traffic to an NVA in another subnet?

**UDRs make this possible.**

They give you the surgical precision to enforce rules that align with business and security policies.

---

## Summary

It might feel clunky. It might feel dated. But UDRs in Azure are there **because the network is no longer physical**—it’s virtual, dynamic, and global. And until Azure Route Server and network policy engines become more intuitive and widespread, **UDRs are your way of taking back control**.

And yes… one day, hopefully soon, the magic will become a bit more automatic. But for now? UDRs are your friends—just misunderstood ones.

---

Thanks for sticking with me. If you’ve been nodding along thinking “Yep, I’ve been there,” then you’re not alone. The next blog will get into Azure Route Server itself—and how it changes this whole game.

— Wayne, Cloud-Recon
