---
layout: post
title: "Azure Networking: Why We Still Rely on Route Tables"
permalink: /azure-networking-why-we-still-rely-on-route-tables/
---

## 1. Microsoft's Global Network Isn't a Flat LAN

![Microsoft Global Network Map](/Azure-Blog/assets/images/MSFT-GB-Network.webp)

Microsoft owns one of the largest and most advanced global networks in the world, spanning over 180 global edge sites and interconnecting every Azure region with lightning-fast private fibre. When you connect a spoke VNet in UK West to a service in UK South, that traffic doesn’t touch the public internet, it stays entirely inside Microsoft's backbone.

That’s brilliant for performance and security, but it also means you can’t rely on legacy routing behaviour. Traffic might take unexpected paths because Azure assumes *global reachability is a good thing*, and it is — until you need inspection, segregation, or policy enforcement. That’s where UDRs come in. They act like signs at junctions, telling Azure exactly where to send traffic instead of letting the system decide for you.

## 2. What Route Server Tries to Solve

Azure Route Server was built to modernise how routing is managed. Instead of static UDRs, it allows NVAs to dynamically advertise routes using BGP. This makes your routing topology more resilient and more in line with how enterprise networks function outside the cloud.

But in many environments, especially where inspection or policy enforcement is tightly controlled, UDRs are still the simplest and most deterministic option. You tell the platform exactly what to do, and it obeys.

## 3. Real-World Use Case

A global legal firm used traditional UDRs in “transition mode” across their UK South environment. Within days, they identified unexpected traffic paths to a storage account hosting confidential case files. After refining policy, they secured the data perimeter without affecting operations.

NSGs could not have done this alone. They act like a hotel key card: access control. UDRs act like the corridors: traffic direction. You need both to stay secure.

## 4. NSGs Are Not Enough

NSGs are full of limitations because they don’t control the path — they only inspect traffic **after** it arrives at a destination subnet. Without UDRs, traffic might not go where you expect, and you may never realise. This is especially critical for hub-and-spoke or hybrid scenarios where deterministic flow matters.

## 5. Summary

Until Azure-native BGP convergence, monitoring, and policy enforcement can replace the predictability of UDRs, they remain essential. If you need to **guarantee** where traffic goes and which NVA or service inspects it, UDRs are your best tool.

Yes, it’s 2025.  
Yes, we still rely on route tables.
