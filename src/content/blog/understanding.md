---
title: "How Internet works on Cruise Ships"
description: "A Breif review about my Thoughts"
pubDate: 2026-01-29
category: Tech
---

<h1>Heyy<h1>
Working as a software engineer with a focus on **Embedded Linux**, I’ve always been fascinated by how we maintain high-availability systems in extreme environments. As I explore roles as an **IT Officer at sea**, understanding the networking infrastructure of a cruise ship is priority number one.

## The Hybrid Connectivity Model

Getting a stable ping in the middle of the Atlantic isn't as simple as connecting to a local cell tower. It requires a sophisticated multi-path routing strategy.

### 1. LEO Satellites (The Starlink Revolution)
Most modern fleets have pivoted to **Low Earth Orbit (LEO)** satellites. Unlike traditional satellites that sit 35,000km away, these are only about 550km up. 
* **Latency**: Reduced from 600ms+ to sub-50ms.
* **Bandwidth**: Allows for seamless video streaming and even remote work for digital nomads.

### 2. MEO & GEO Backups
For critical systems, ships still maintain connections to **Medium Earth Orbit (MEO)** and **Geostationary (GEO)** satellites. These are the "production-ready" backups that ensure the ship stays operational even if LEO coverage drops.

### 3. Terrestrial WISP (Near Shore)
When we are docked in ports like **Dubai** or **Amsterdam**, the ship often switches to **Wireless Internet Service Providers (WISP)**. These use high-gain tracking antennas to grab 4G/5G signals from the shore, which is much cheaper than using satellite data.

## Distributing the Signal: The Onboard Stack

Once the "backhaul" reaches the ship, the IT team has to distribute it to 3,000+ devices.
* **Core Switching**: High-capacity fiber backbones running through the ship's vertical risers.
* **Access Points**: Hundreds of Wi-Fi 6 APs strategically placed to penetrate steel bulkheads.
* **Traffic Shaping**: Using specialized appliances to prioritize bridge communications and emergency systems over a passenger's Netflix stream.

---

**Systems Status**: Operational // **Region**: Global Waters

*What do you think about the shift to Starlink at sea? Leave a comment below!*