# home-media-platform
Self-hosted media platform - architecture decisions, trade-offs, and operational lessons from a Product Manager who builds things.

# 🎬 Home Media Platform - Self-Hosted Streaming Infrastructure

> **I built this for fun. But documenting it revealed how many of the same decisions I make in enterprise product work - trade-off analysis, user-first design, build vs. buy, operational resilience - show up naturally when I build things for myself.**

---

## The Problem

I have a large personal media collection and a 4 Gbps fibre connection. I wanted a Netflix-like experience - one interface, all my content, accessible from anywhere - that I could share with friends and family. Anyone with access should be able to request new content and have it appear in the library automatically, without me manually managing the process.

The platform needed to:
- **Aggregate** all media into one interface with consistent metadata and discovery
- **Automate** the content management pipeline - from request to organized, playable media - with minimal manual intervention
- **Enable self-serve requests** so users can add content without going through me
- **Serve** multiple clients (TV, mobile, web) with adaptive streaming quality
- **Operate reliably** without constant maintenance - if something breaks, I want to know about it before my users do
- **Be accessible remotely** via my own domain, secured for public internet exposure

The constraint: run everything on a single server, keep costs near zero beyond hardware, and maintain it in ~30 minutes per month.

---

## Solution Architecture

<!-- TODO: Replace with your actual PlantUML diagram -->
<!-- Generate with: plantuml docs/architecture.puml -->

![Architecture Diagram](docs/architecture.png)

## Hardware Architecture

![Hardware Architecture](docs/hardware_architecture.png)

### How It Works (End-to-End Flow)

```
SERVING PATH (what users interact with):
User Request -> [Cloudflare] -> [Reverse Proxy] -> [Plex] -> Transcoded Stream -> Client Device
                                                      ↑
                                                  [Storage]
                                                (Media Files)

AUTOMATION PATH (what runs in the background):
User Request -> [Overseerr] -> [Sonarr/Radarr] -> [Indexers] -> [Download Client + VPN] -> [Storage]
                                                                   (network isolated)
```

### The Product Decisions Behind Each Layer

Every service in this stack runs as a Docker container on Unraid. This keeps the environment reproducible, isolated, and easy to update or roll back.

I've organized the stack into four layers. Each layer exists because of a specific product or operational requirement.

---

### Layer 1: Content Delivery - The User-Facing Product

| Component | Role | Why This Choice |
|-----------|------|-----------------|
| **Plex** | Media server + client apps | Mature ecosystem with best-in-class client apps across every platform (Android, iOS, web, smart TVs). Lifetime license purchased on sale - well worth it for the reliability and seamless experience. GPU hardware transcoding is enabled, which offloads video conversion from the CPU and allows multiple simultaneous streams without performance degradation. Compatible with non-Plex players (DLNA) as a fallback, though you lose the library UI. |


**The product thinking:**
This is the only layer end users actually see. Every other layer exists to make this one work seamlessly. The core UX requirements I was solving for:

- One-click playback from any device - TV, phone, laptop, tablet, including Android TV
- Automatic metadata, artwork, and organization - no manual sorting
- Multi-user profiles with separate watch histories
- Remote access when traveling - not just local network

**What I evaluated and rejected:**
- **Jellyfin** - The right choice for cost (free) and supporting open source, but the wrong choice for household adoption. Client apps lacked the polish and reliability of Plex, and the setup experience required more technical effort than I wanted to impose on non-technical users.

**Key metrics:**
- *Time from content available to playable:* Target < 5 minutes with zero manual steps
- *Uptime:* Must be available during normal waking hours (9 AM to 1 AM)
- *Request fulfillment rate:* ~90% of requests are fulfilled automatically. The remaining ~10% require manual intervention due to niche content (foreign language films, older titles, rare formats)

---

### Layer 2: Automation Pipeline - Eliminating Manual Toil

| Component | Role | Why This Choice |
|-----------|------|-----------------|
| **Sonarr** | TV series monitoring + acquisition | Automates tracking, quality selection, and episode management |
| **Radarr** | Movie monitoring + acquisition | Same automation pattern for films |
| **Prowlarr** | Indexer management | Centralizes indexer configuration for both Sonarr and Radarr - one place to manage content sources instead of configuring each tool separately |
| **Various Indexers** | Content discovery | Multiple indexers chosen for coverage across different genres and content types. Redundancy ensures availability when individual sources go down |
| **Overseerr** | Request management | User-facing request portal with Plex integration for authentication and library visibility. This was the single most impactful addition to the stack (see Incident 3 below) |
| **Unpackerr** | Post-processing | Automates extraction of downloaded content into the media library - the last step in the zero-touch pipeline |

**The product thinking:**
This layer is the "operations automation" of the platform. Without it, every new piece of content requires manual search, download, rename, organize, and import. That's not a sustainable product. It's a chore with a nice front end.

The design philosophy was **one-touch acquisition**: define what you want once (request a series, add a movie via the web interface), and the pipeline handles everything from discovery through to organized, playable media, at a defined quality. This is the same principle behind any good workflow automation - reduce human steps to reduce errors and operational cost.

**Architecture decisions:**
- **Why separate Sonarr and Radarr:** TV series and movies have fundamentally different acquisition patterns - seasons, episodes, and quality upgrades over time vs. one-time downloads. Sonarr and Radarr are purpose-built for their respective domains and are the most mature tools in the ecosystem. Separation of concerns mirrors how you'd design microservices around distinct domain logic.
- **Quality profiles as a product decision:** With ample storage, I prioritized bitrate and source quality to match my home theatre setup. File size alone doesn't indicate quality - the profile is configured to prefer vetted sources with high bitrate at target resolution. This is a resource allocation trade-off: storage cost vs. playback quality.

**Key metric:** Percentage of content requests requiring manual intervention - target is < 5%.

---

### Layer 3: Network & Security - Trust Boundaries

| Component | Role | Why This Choice |
|-----------|------|-----------------|
| **Nginx Proxy Manager** | Reverse proxy + SSL termination | Docker-native with a web UI for managing proxy rules and Let's Encrypt SSL certificates. Low configuration overhead - I don't want to spend hours writing Nginx configs by hand. |
| **Cloudflare** | DNS + proxy + security | DNS management, IP masking, DDoS protection, and bot mitigation in one free package. Essential for exposing a home server to the public internet. |
| **Plex User Accounts** | User management + access control | Built-in authentication and per-user permissions. Controls who can access the server and what they can do. |

**The product thinking:**
This layer defines the trust boundaries of the platform. It answers two questions: "Who can access what?" and "What traffic is visible to whom?"

In enterprise product work, I spend significant time on security and risk management - first line of defence risk assessments, data flow mapping, privacy triaging. The same principles apply here, arguably with higher stakes since this is my personal home server:

- **Nginx Proxy Manager** is the internal access control layer - it determines which services are exposed, handles SSL/TLS termination, and routes traffic to the correct container. It's the equivalent of an API gateway (e.g. Apigee) in a B2B platform. Only services that need external access get a proxy rule. Everything else is internal-only by default - a default-deny posture.

- **Cloudflare** is the external perimeter. Beyond DNS, its proxy layer hides the server's real IP address, absorbs volumetric attacks, and filters malicious traffic before it reaches my network. This is the same defense-in-depth approach used in enterprise environments - never expose your origin server directly.

- **Plex User Accounts** handle application-level access control. Each user gets a managed account with specific permissions. This is the identity layer of the platform - lightweight, but it enforces who can access what content and features.

- **Docker container isolation** provides an additional security layer. Each service runs in its own isolated environment with its own network and filesystem. I reviewed default configurations for each container and tightened port exposure and volume mounts - only giving each service access to the specific directories and ports it actually needs. This limits the blast radius if any single service is compromised.

**Architecture decisions:**
- **Why Nginx Proxy Manager:** It was the most widely adopted reverse proxy in the self-hosting community, which meant better documentation, more troubleshooting resources, and a proven track record. The web UI for managing proxy rules and SSL certificates was a key factor - I wanted something I could configure quickly without editing config files by hand. I optimized for speed of setup and community support over complexity.

- **Why Cloudflare over alternatives:** Free tier covers everything I need - DNS, proxy, SSL, DDoS protection, bot management. Trusted by enterprises, which gives me confidence in the security posture. The alternative was direct port forwarding with dynamic DNS, which exposes the server's IP and provides zero protection.

- **Why Plex accounts over a separate auth system:** Plex authentication is built-in and sufficient for this use case. Overseerr integrates with it natively, so users have a single identity across the request portal and the media server. Adding a dedicated authentication layer would add complexity without meaningfully improving security for a media platform with a small, trusted user base. If the user base grew or the platform handled sensitive data, I'd revisit this decision.

---

### Layer 4: Storage - The Data Layer

| Component | Role | Why This Choice |
|-----------|------|-----------------|
| **Unraid** | OS + storage + container host | Unraid's JBOD (Just a Bunch Of Disks) array allows mixing different drive sizes and adding capacity incrementally - buy drives on sale, slot them in. Parity drive provides single-disk fault tolerance. SSD cache accelerates frequently accessed data. Native Docker and VM support means the entire stack runs on one machine. |

**The product thinking:**
Storage is the most "boring" layer and the most critical. If this fails, everything above it is useless. For a home setup, I prioritize cost first, then reliability - which is why I only buy NAS-rated drives when they go on sale. Unraid's architecture supports this approach perfectly: I don't need to buy all my drives at once or match sizes. I expand opportunistically as deals appear.

**Architecture decisions:**
- **Why Unraid over TrueNAS:** TrueNAS requires matched drive sizes and more upfront planning - that didn't fit my philosophy of using whatever extra hardware I had lying around. I liked the idea of reducing e-waste by giving old drives a second life in the array. Unraid also hit a sweet spot between approachability and depth: easy to get running out of the box, but with the ability to dive deeper into configuration when I want to. Its native Docker management handles container isolation, networking, and port mapping through a UI - no command line required for day-to-day operations. TrueNAS felt like it demanded that depth from day one.
- **Capacity planning:** Not formally tracked because Unraid's architecture makes expansion trivial. Current setup has sufficient headroom, and new drives can be added to the array without rebuilding. One parity drive provides single-disk fault tolerance - acceptable for replaceable media content.
- **Backup strategy:** Configuration files are backed up automatically via a separate Unraid-level backup stack that covers all services, not just the media platform. Media files are not backed up because they're replaceable with minimal effort (though time-consuming to re-acquire). This is a cost-of-replacement analysis - the same framework I'd use to decide what gets disaster recovery investment in an enterprise product.
- **Updates:** The entire stack auto-updates via Unraid's update management. This keeps maintenance overhead low and ensures security patches are applied without manual intervention.

---

## Operational Realities

### What Went Wrong (And What I Learned)

**Incident 1: Power outage with no UPS**
- What happened: A power outage caused an ungraceful shutdown. The result: a full parity check (hours of downtime) and corrupted Docker images that needed rebuilding.
- Root cause: No uninterruptible power supply protecting the server.
- Fix: Purchased a UPS and configured Unraid to initiate a graceful shutdown when battery reaches a threshold.
- PM lesson: The UPS cost ~$150. The parity check took 18+ hours. Resilience investments almost always cost less than recovery. In enterprise terms, this is the argument for investing in fault tolerance before an outage forces you to.

**Incident 2: Server went down without anyone noticing**
- What happened: The server was down for an unknown period. I only found out when a user tried to access it and reported it wasn't responding.
- Root cause: No monitoring or alerting in place. Zero observability into system health.
- Fix: Set up daily heartbeat emails confirming the server is online. For a home setup, daily is sufficient - I don't need a PagerDuty-style on-call rotation.
- PM lesson: You can't fix what you can't see. The monitoring wasn't expensive or complex to implement - I just hadn't prioritized it because nothing had gone wrong *yet*. This is the classic "availability bias" trap in roadmap prioritization: reliability work feels low-priority until the first outage.

**Incident 3: My SO became my most frequent support ticket**
- What happened: My SO was requesting new content weekly, each time requiring me to manually search, download, and organize it. I had become a human ticketing system.
- Root cause: No self-serve mechanism for content requests.
- Fix: Deployed Overseerr - a request management portal that integrates with Plex for authentication and with Sonarr/Radarr for automated fulfillment.
- PM lesson: This single addition reduced my operational workload and increased user satisfaction simultaneously. It's the textbook example of why you listen to user behavior, not just user complaints. My SO never said "build me a request portal." She said "can you add this movie?" The underlying need was self-serve access, not a specific solution. In enterprise PM work, the pattern is identical - the feature request is rarely the real requirement.

---

## Infrastructure as Product: The Parallels

| Home Media Platform | Enterprise Product Equivalent |
|-------------------|-------------------------------|
| Plex media server | Customer-facing application |
| Sonarr / Radarr / Overseerr | Workflow automation / background processing |
| Nginx Proxy Manager + Cloudflare | API gateway + network security perimeter |
| Unraid storage layer | Database / data platform |
| Monitoring + heartbeat alerts | Product analytics + operational observability |
| Quality profiles | Feature configuration / tier management |
| Docker Compose orchestration | Container orchestration / deployment pipeline |
| One-touch content pipeline | End-to-end automation reducing operational toil |

---

## Repo Structure

```
home-media-platform/
├── README.md                        # You're reading this
├── docs/
│   ├── architecture.puml            # PlantUML source
│   ├── architecture.png             # Rendered diagram
│   └── hardware_architecture.png    # Hardware layout
├── docker-compose.yml               # Full stack definition (secrets redacted)
└── configs/
    ├── nginx-proxy-manager/
    │   └── [config files]
    ├── sonarr/
    │   └── [relevant config snippets]
    ├── radarr/
    │   └── [relevant config snippets]
    └── monitoring/
        └── [dashboards, alert rules]
```

---

## Roadmap

| Priority | Item | Why It Matters |
|----------|------|---------------|
| P0 | Complete documentation of current stack | Foundation for everything else |
| P1 | Set up alerting for storage capacity thresholds | Prevent the "disk full" incident from recurring |
| P2 | Implement automated health checks with restart policies | Move toward self-healing infrastructure |
| P2 | Dashboard consolidation - at-a-glance health check | Reduce context-switching during troubleshooting |
| P3 | Document disaster recovery runbook | Enable someone else to restore the platform without my involvement |

---

## Acknowledgments

The majority of this stack is built on open source software - Sonarr, Radarr, Prowlarr, Overseerr, Unpackerr, and Nginx Proxy Manager. This project wouldn't exist without the developers and communities behind these tools. If you use them, consider contributing back - whether that's code, documentation, bug reports, or donations.

---

## About Me

**Truman Leung** - Senior Product Manager specializing in enterprise identity, security, and platform products. 12 years of combined technical and product experience at RBC, building MFA platforms (6M+ users), KYC compliance systems, and API integrations driving $680M in net-new investments.

I build things to understand how they work. I document them to show how I think.

Currently exploring the intersection of AI and identity security.

[LinkedIn](https://linkedin.com/in/leungtruman) · [Email](mailto:truman.leung@gmail.com)
