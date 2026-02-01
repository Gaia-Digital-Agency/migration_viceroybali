Client: Viceroy Bali 
Document: Phase 1 - Migration Plan
Start Activity Date: 2nd February 2026

MIGRATION PLAN

Phase 1: Infrastructure Upgrade

Objective: Move Viceroy Bali from "Shared Hosting" (Hostinger) to a "Dedicated Cloud Environment" (Google Cloud).

Goal: Improved speed increase while keeping the current website exactly as it is to protect SEO.

Business Case

The "Shared" Problem: Current server is like a shared apartment. If other websites on the same server are busy, Viceroy Bali slows down.
The "Google Cloud" Solution: We are moving into a private villa. We get 100% of the resources. This fixes the "lag" (TTFB) guests feel when they first click the site.
Prestige: Ultra-luxury brands require high-performance infrastructure. Shared hosting is for blogs; Google Cloud is for global enterprises.

2. The SEO Safety Protocol

We are using a "Mirror & Verify" strategy to ensure Google sees no difference in content.

Zero URL Changes: Not a single link will change. /villas will still be /villas.
Zero Content Changes: We are not touching the text, titles, or images.
The "Host File" Test: We will build the site on Google Cloud first, but keep it hidden from the public. We will test it privately to ensure every button and booking link works perfectly before we tell Google to look at the new server.
Zero Downtime Switch: We will use a "DNS Cutover." The old site stays live until the new one is 100% ready. The switch takes seconds.

3. Migration Roadmap



For Phase 1, the goal is to build a hosting environment that is faster than Hostinger but remains compatible with your current WordPress setup.

TECH STACK (GCP)

Phase 1: High-Performance "Single-Server" 

The Core: The Origin Server

Instead of a shared room, Viceroy Bali gets its own dedicated floor.

Machine: e2-standard-2 (2 vCPU / 8 GB RAM).
Location: Singapore (asia-southeast1).
Software: Nginx + PHP 8.3 + Redis.

2. The Bridge: Unmanaged Instance Group (UMIG)

Google Cloud CDN requires a Load Balancer, and a Load Balancer requires an "Instance Group." Since we only have one server and don't want Google to automatically delete/rebuild it, use a UMIG.

Role: It acts as a logical container that holds your e2-standard-2 so the Load Balancer can "see" it.
Benefit: Simple to manage. If you need to edit a file on the server, you just log in via SSH like a normal VPS.

3. The Shield: Global Load Balancer & Cloud CDN

This is what makes the site feel "Ultra-Luxury" to a guest in London or New York.

Load Balancer: Handles the SSL Certificate (Google-managed) and offloads the encryption work from your server.
Cloud CDN: Caches the Viceroy’s heavy 4K villa images and videos at the "Edge."
Result: The guest gets the image from a server in their own city, not from Bali.

4. The Safety Net: Weekly Snapshot Schedule

This is your insurance policy. We will attach an automated policy to your server's disk.
Frequency: Weekly (e.g., every Sunday at 3:00 AM Bali time).
Retention: 4 Weeks (We always keep the last month of backups).
Manual Trigger: We will also take a "Gold Master" snapshot immediately after the migration is finished and verified.

Technical Summary Table


