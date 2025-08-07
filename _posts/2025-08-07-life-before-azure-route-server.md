---
layout: post
title: "Azure Networking: Why We Still Rely on Route Tables"
date: 2025-08-07 10:00:00 +0100
categories: azure networking
permalink: /azure-networking-why-we-still-rely-on-route-tables/
---

<img src="{{ site.baseurl }}/assets/images/Matrix.webp" alt="Matrix-style Microsoft network wizard" width="500" style="display: block; margin: 0 auto;" />

Before we get into Azure Route Server, let’s talk about something that throws nearly every on-premises network engineer off balance — especially if, like me, you come from a Cisco background and the first time you step into Azure: **the never-ending tangle of route tables**, a world that at first seems a backward step in networking.

In a traditional world, routes were simple. You had a router. It had interfaces. You knew exactly which side of the network a packet came from and where it needed to go. Routing was often automatic, mostly physical, and rarely touched unless something broke or a new site came online. For all its quirks, it *made sense*.

Then you land in Azure and discover you need to build **User Defined Routes (UDRs)** for what feels like… everything.

1. If you want traffic from a spoke to reach your firewall = UDR  
2. If you want to steer traffic through a NVA = UDR  
3. If you want to override a default internet route = UDR  
4. If you want to peer a VNet and keep it secure = UDR  
5. If you’re using Virtual WAN or ExpressRoute or Private Link, guess what… UDR

At first glance, it feels like we’ve gone backwards. All this software-defined magic and we’re still **hand-crafting route tables like it’s 2005**. But there are good reasons for it, and understanding them is the key to designing Azure networks properly.

So why does Azure need UDRs? Why is this “manual routing” still a thing?

---

## 1. Microsoft’s Global Network Isn’t a Flat LAN

<img src="{{ site.baseurl }}/assets/images/GlobalNetwork.webp" alt="Microsoft Global Network Map" width="700" style="display: block; margin: 0 auto;" />

Microsoft owns one of the largest and most advanced global networks in the world, spanning over 180 global edge sites and interconnecting datacentres in every continent. But this global network isn't flat — it's segmented, controlled, and deeply optimised for security and performance.

When you design your Azure network, you're not just placing VMs and subnets in a flat LAN. You're building on top of this vast fabric, and **every decision you make about routing affects how your traffic moves across that global backbone**.

---

## 2. Azure Doesn’t Guess Where You Want Traffic to Go

In an on-premises environment, you often had default gateways, static routes, or dynamic protocols like OSPF or BGP to handle routing logic. Azure gives you **incredible flexibility**, but that comes with a cost — **you have to be explicit**.

There is no dynamic BGP within a VNet. No “I’ll figure it out for you” engine. You want spoke-to-hub? Route it. Want to bypass the internet for private DNS or secure egress? Route it. Azure will give you all the tools, but **you have to assemble them correctly**.

---

## 3. Security in Azure Demands Precision

Hand-crafting UDRs isn’t just a legacy thing — it’s a **security design principle**. If you can see and control every path a packet takes, you can:

- Force traffic through an NVA or inspection point  
- Prevent traffic from going out via the default internet route  
- Ensure compliance with industry or legal standards  
- Detect and stop lateral movement during a breach

In a cloud world, **routing is part of your zero trust posture**. Don’t trust default behaviour. Define everything.

---

## 4. Azure Route Server Changes the Game — But Doesn’t Replace UDRs (Yet)

Azure Route Server lets you bring dynamic BGP into your virtual networks — a massive leap forward. It reduces the manual effort to keep UDRs updated when BGP-learned routes change. But make no mistake:

> **You still need a UDR to steer traffic into the Route Server path in the first place.**

It doesn’t magically route all your traffic. You still define the next hop to the firewall or Route Server using UDRs. It’s not plug-and-play. It’s **plug-and-plan**.

---

## Final Thoughts

If you're coming into Azure networking and wondering why it feels like you’ve gone back in time — you haven’t. You’ve moved into a space where routing is deliberate, visible, and secure by design.

Azure gives you building blocks. **User Defined Routes are one of the most powerful and essential tools** — not because Azure is broken, but because your network should never be left to guesswork.

---

Stay tuned for the next post: we’ll dive into Route Server itself — what it fixes, what it doesn’t, and how to use it like a pro.
