---
layout: default
title: ExpressRoute High Availability — Bow Tie vs Square
description: Practical ExpressRoute HA patterns from Azure Networking by Jose Moreno and Adam Stuart, with routing behaviours that actually hold up on a bad day.
permalink: /expressroute-high-availability/
---

<p align="center">
  <a href="https://www.amazon.co.uk/Azure-Networking-Understand-networking-architectures/dp/9365897394?crid=2SWTROT8XLSMW&dib=eyJ2IjoiMSJ9.sXCXvAe9uHTm1nWC3c9-gi2nIoc67cCEAYrymCMfk6bv5j4yjuAgxgZXzRl4BA0oD9gvOclcR0sPoGbEbZ3UoPqW_nJ3SG99vgYueNEUz1I.UhZ4fvUZaJ4Ekavbda5WQPnL9D4r2RwSfZ3EYe8UzNA&dib_tag=se&keywords=azure+networking+adam+stuart&qid=1754997342&sprefix=azure+netowrking%2Caps%2C123&sr=8-1&linkCode=ll1&tag=rucksackrecon-21&linkId=fca1a35fe38fa842f4e49b41bc4b4445&language=en_GB&ref_=as_li_ss_tl" target="_blank">
    <img src="{{ '/assets/images/book%20cover.jpg' | relative_url }}" alt="Azure Networking book cover by Jose Moreno and Adam Stuart" width="500">
  </a>
</p>

In my view there are many certifications that do not mean much in the real world, I've met a lot of engineers with few certs for many reasons, either they stuggle with the exam format through a diversity or they just don't like taking them, but CCIE is not one of them. For me, anyone with a CCIE does not walk on water, they simply moonwalk on water! The networking knowledge needed is intense. I have worked directly with Adam Stuart at Microsoft and he is not only top class technically, he is a bloody great guy too. Along with Jose Moreno he co authored <em>Azure Networking</em>. What follows pulls out the ExpressRoute high availability guidance from their book and frames it for engineers who need a design that keeps working when a circuit fails at 03:00. This book is great to read start to finish or to use it as a referance book, as I do.

## <span style="color:green">Local first steps</span>

Start with two circuits in two different peering locations. Separate providers if you can. Separate facilities always. If both circuits land in the same building you have a single point of failure you do not control. Order dual ExpressRoute circuits, terminate them in different metros, and validate that each has its own physical path. Only when the real world plumbing is diverse should you move on to routing.

## <span style="color:green">Why it matters</span>

ExpressRoute is often the lifeline to your workloads. A single circuit outage without a tested design means downtime, escalations, and long nights. A sound design means the failure is a log entry. The goal is simple: in normal operation, each site uses the nearest Azure region. In failure, traffic shifts automatically to the surviving path without violating security or policy.

## <span style="color:green">How easy it is</span>

The patterns are straightforward once you decide on the topology.

1. Build two circuits in different locations.
2. Connect them to two Azure regions.
3. Choose your topology: Bow Tie for lowest latency and clarity, or Square for cost lean scenarios where a longer failover path is acceptable.
4. Set routing preferences so the right paths are preferred in steady state and failover is automatic.

You must test. You should document exact expected paths per prefix family and you should rehearse failure modes.

## <span style="color:green">How much it will save your backside</span>

Circuits fail. Peering locations have maintenance. A Bow Tie or Square design turns a provider event into a non event for the business. The difference is people sleeping or not.

## <span style="color:green">The Bow Tie design</span>

Bow Tie connects each on premises site to both Azure regions. You then bias routing so that each site prefers its local region in steady state.

<p align="center">
  <img src="{{ '/assets/images/expressroute_bow_tie.png' | relative_url }}" alt="ExpressRoute Bow Tie design" width="500">
</p>

Operational intent:

- West site primarily uses Azure West. East site primarily uses Azure East.
- Both sites have a secondary path to the opposite region.
- If a circuit or peering location fails, traffic naturally selects the surviving path.

Practical routing notes from the book:

- From on premises into Azure, use your IGP to redistribute the near region with a better metric at the local site, and a worse metric at the remote site. Do the mirror for the other region. This makes the right path win in steady state.
- From Azure to on premises, the ExpressRoute gateways follow standard selection. Longest prefix wins. For equal prefixes, shorter AS path wins. With equal AS paths, the gateways will distribute across links. When you need to prefer a region, prepend the AS path on the non preferred circuit so the preferred one wins.
- Use weights sparingly. Default weight is zero. Treat AS path and prefix length as primary tools, then consider weight if you need an extra nudge.
- Validate symmetry for sensitive traffic. If you enable inspection devices in Azure, add UDRs carefully so the return path still matches policy.

Why engineers like Bow Tie:

- Lowest latency under normal conditions because each side uses the nearest region.
- Clean mental model. Each site has two paths to Azure. Prefer local, fail to remote.
- Clear testing story. You can drop one circuit and watch flows migrate without surprises.

## <span style="color:green">The Square design</span>

Square connects each site only to its local region. In failure, traffic rides the Azure backbone to reach the other region. You trade some latency in a failover for lower circuit costs and a simpler commercial model.

<p align="center">
  <img src="{{ '/assets/images/expressroute_square.png' | relative_url }}" alt="ExpressRoute Square design" width="500">
</p>

When Square makes sense:

- Workloads can tolerate a longer path during a failover window.
- You want to use Local or Standard circuits rather than Premium in larger geographies.
- You prefer fewer cross connects to manage on day one.

Caveats to plan for:

- The failover hop is longer. Test critical transactions under failover.
- Document the Azure to Azure leg so your operations team understands the expected path.
- Be strict with DNS and health probing so clients re home quickly when paths move.

## <span style="color:green">Adding the safety raft: VPN as failover</span>

When risk appetite demands a third way, layer site to site VPN as a last resort. Terminate on an Azure VPN gateway or NVA and advertise a limited set of critical prefixes. Keep throughput expectations realistic and test your routing so the VPN is only selected when both ExpressRoute paths are unavailable.

## <span style="color:green">Coexistence and transitivity</span>

If you run ExpressRoute and VPN together, Azure Route Server can allow the gateways to exchange routes so each overlay knows the other exists. For traffic inspection through Azure Firewall or an NVA, bind a route table to the gateway subnet and steer specific prefixes through the firewall while still allowing the gateways to learn each other’s routes for reachability. Treat Route Server as a control plane helper. It teaches routes. It does not set next hops unless you tell it to with UDRs.

## <span style="color:green">Local first steps checklist</span>

- Two ExpressRoute circuits in different peering locations. Confirm physical diversity.
- Two Azure regions selected with a clear primary for each site.
- Decide Bow Tie or Square. Write down why.
- Plan inbound routing behaviour. Metrics and redistribution per site.
- Plan outbound routing behaviour. Prefix length, AS path, and if required, weight.
- Decide on inspection and where UDRs are needed. Validate return paths.
- Write a failover runbook. Test, document outputs, and capture packet captures.

## <span style="color:green">Why it matters</span>

Because the day a circuit drops is not the day you want to start drawing diagrams. This design work prevents outages, preserves user trust, and avoids noisy board updates.

## <span style="color:green">How easy it is</span>

It is engineering, not wizardry. The patterns are known. The test cases are repeatable. The only real mistake is not building for failure and not rehearsing it.

## <span style="color:green">How much it will save your backside</span>

Enough that the outage becomes a ticket rather than a crisis. Enough that your name is associated with resilience instead of recovery.

## <span style="color:green">Go deeper</span>

This post skims key lessons from <em>Azure Networking</em> by Jose Moreno and Adam Stuart. The book goes further with redistribution metrics, weight behaviour, AS path strategies, coexistence patterns, and clean diagrams. If you work with Azure networking, buy it, read it, and keep it within reach.
