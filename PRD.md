# Product Requirements Document (PRD)
# AustinEAutos Content Engine

**Version:** 1.0
**Date:** February 16, 2026
**Author:** Bryan Flood / Onada Bros
**Status:** MVP Complete — Phase 1 Live

---

## 1. Overview

### 1.1 Product Summary
The AustinEAutos Content Engine is an automated social media content pipeline that monitors vehicle inventory changes at AustinEAutos (a pre-owned vehicle dealership in Round Rock, TX) and generates branded social media posts with AI-written copy and dynamically generated images — delivered to email for human approval before publishing.

### 1.2 Problem Statement
AustinEAutos manages a rotating inventory of ~100 pre-owned vehicles (primarily electric and hybrid, but not exclusively). Creating social media content for new arrivals, price drops, and other inventory events is time-consuming and inconsistent when done manually. The dealership needs a system that:
- Detects inventory changes daily without manual monitoring
- Generates on-brand, platform-specific social media copy automatically
- Produces branded images for each post
- Keeps a human in the loop for quality control before publishing

### 1.3 Solution
A fully automated pipeline that runs daily:
1. **Scrapes** the dealership website for current inventory
2. **Compares** against a historical inventory log to detect new arrivals, price drops, and still-available vehicles
3. **Generates** AI-written social media copy for 5 platforms (Instagram, Facebook, X, TikTok, Reels)
4. **Creates** branded 1200x1200 images using vehicle photos
5. **Emails** the content for human review and approval
6. **Logs** all generated posts for tracking and analytics

### 1.4 Target Users
- **Primary:** Brian Fosbury (dealership owner / content approver)
- **Secondary:** AustinEAutos social media team
- **Developer:** Bryan Flood / Onada Bros
- **End audience:** Austin/Round Rock area residents aged 25-55 interested in quality pre-owned vehicles (especially EV-curious buyers and EV owners)

---

## 2. Architecture

### 2.1 System Components

| Component | Service | Role |
|-----------|---------|------|
| Web Scraper | Apify (CheerioCrawler) | Extracts vehicle inventory from austineautos.com daily |
| Orchestration | Make.com (Scenario 1) | Coordinates the entire pipeline via webhook-triggered automation |
| Database | Google Sheets | Inventory Log (historical tracking) + Post Log (content tracking) |
| AI Copy | OpenAI GPT-4o | Generates platform-specific social media copy |
| Image Gen | Placid API | Creates branded 1200x1200 social images |
| Approval | Gmail | Delivers content + image for human review |
| Brand Voice | SOUL.md | Brand identity document that guides AI tone and messaging |

### 2.2 Data Flow

```
austineautos.com
       │
       ▼
   Apify Scraper (daily 8 AM CST)
       │ ~100 vehicles extracted
       ▼
   Make.com Webhook
       │
       ▼
   HTTP GET Apify Dataset
       │
       ▼
   Iterator (process each vehicle)
       │
       ▼
   Google Sheets Search (Inventory Log lookup)
       │
       ▼
   Router ─────────────────────────────────┐
       │              │                    │
   Route A         Route B             Route C
   New Arrival     Price Drop          Still Available
   (not in log)   (price decreased)   (price same/up)
       │              │                    │
   Add Row         Update Row          Update Row
   to Inventory    (new price)         (last_seen date)
       │              │
   Set Variables   [DEFERRED]
       │
   OpenAI GPT-4o (rate limited: 2 per run)
       │
   JSON Parse
       │
   Placid Image Generation (synchronous)
       │
   Gmail Approval Email
       │
   Post Log (Google Sheets)
       │
   Update Inventory Flags
```

### 2.3 Module Map (Make.com Scenario 4141755)

| Module | ID | Type | Status |
|--------|----|------|--------|
| Webhook (Apify trigger) | 72 | gateway:CustomWebHook | LIVE |
| HTTP GET (Apify dataset) | 5 | http:MakeRequest | LIVE |
| Iterator | 7 | builtin:BasicFeeder | LIVE |
| Sheets Search (Inventory) | 11 | google-sheets:filterRows | LIVE |
| Router (3 routes) | 13 | builtin:BasicRouter | LIVE |
| Add Row (Inventory Log) | 21 | google-sheets:addRow | LIVE |
| Set Variable (content_type) | 23 | util:SetVariable2 | LIVE |
| Set Variable (vehicle_name) | 26 | util:SetVariable2 | LIVE |
| OpenAI GPT-4o | 39 | http:MakeRequest | LIVE |
| JSON Parse | 50 | json:ParseJSON | LIVE |
| Placid Image Gen | 41 | http:MakeRequest | LIVE |
| Gmail Approval | 43 | google-email:sendAnEmail | LIVE |
| Post Log (Add Row) | 45 | google-sheets:addRow | LIVE |
| Update Inventory Flags | 47 | google-sheets:updateRow | LIVE |
| Price Drop Row Update | 27 | google-sheets:updateRow | Built (deferred) |
| Price Drop Variables | 31 | util:SetVariable2 | Built (deferred) |
| Still Available Update | 32 | google-sheets:updateRow | LIVE |

---

## 3. Functional Requirements

### 3.1 Inventory Scraping (FR-100)

| ID | Requirement | Status |
|----|-------------|--------|
| FR-101 | Scraper fetches vehicle sitemap daily at 8:00 AM CST | DONE |
| FR-102 | Scraper extracts all vehicle detail pages from sitemap | DONE |
| FR-103 | Each vehicle record includes: stock_number, year, make, model, trim, price, mileage, VIN, image_urls, vehicle_url, status, colors, fuel_type, days_in_stock | DONE |
| FR-104 | Scraper handles up to 200 vehicles per run | DONE |
| FR-105 | Scraper triggers Make.com webhook on successful completion | DONE |
| FR-106 | Scraper outputs to Apify default dataset (JSON) | DONE |

### 3.2 Inventory Comparison (FR-200)

| ID | Requirement | Status |
|----|-------------|--------|
| FR-201 | Each scraped vehicle is looked up in the Inventory Log by stock_number + today's date | DONE |
| FR-202 | Vehicles not found in log are classified as NEW_ARRIVAL | DONE |
| FR-203 | Vehicles found with a lower current price than logged price are classified as PRICE_DROP | DONE (route built, not connected to post chain) |
| FR-204 | Vehicles found with same or higher price are classified as STILL_AVAILABLE | DONE |
| FR-205 | New arrivals are added as new rows in the Inventory Log | DONE |
| FR-206 | Price drops update the existing row with new price | DONE |
| FR-207 | Still-available vehicles update the last_seen timestamp | DONE |

### 3.3 Content Generation (FR-300)

| ID | Requirement | Status |
|----|-------------|--------|
| FR-301 | GPT-4o generates copy for 5 platforms: Instagram, Facebook, X, TikTok, Reel script | DONE |
| FR-302 | Content follows brand voice defined in SOUL.md | DONE |
| FR-303 | Instagram captions include hashtags (15-20 tags) | DONE |
| FR-304 | X posts respect 280-character limit | DONE |
| FR-305 | Copy includes vehicle name, price, and listing URL | DONE |
| FR-306 | No high-pressure sales language ("ACT NOW", "LIMITED TIME", etc.) | DONE |
| FR-307 | GPT response is structured JSON parsed into individual fields | DONE |
| FR-308 | Rate limiter: maximum 2 posts generated per pipeline run | DONE |

### 3.4 Image Generation (FR-400)

| ID | Requirement | Status |
|----|-------------|--------|
| FR-401 | Branded 1200x1200 image generated for each post | DONE |
| FR-402 | Image includes vehicle name (title layer) | DONE |
| FR-403 | Image includes price (subline layer) | DONE |
| FR-404 | Image includes vehicle photo from inventory CDN | DONE |
| FR-405 | Image generation is synchronous (create_now: true) | DONE |
| FR-406 | Generated image URL is stored in Post Log | DONE |
| FR-407 | Generated image is displayed inline in approval email | DONE |

### 3.5 Approval Workflow (FR-500)

| ID | Requirement | Status |
|----|-------------|--------|
| FR-501 | Approval email sent to bryan@onadabros.com | DONE |
| FR-502 | Email includes: content type, vehicle name, generated image, all 5 platform captions | DONE |
| FR-503 | Email subject follows format: "{content_type}: {vehicle_name} --- AustinEAutos" | DONE |

### 3.6 Post Logging (FR-600)

| ID | Requirement | Status |
|----|-------------|--------|
| FR-601 | Each generated post is logged to the Post Log sheet | DONE |
| FR-602 | Log includes: date, content_type, stock_number, vehicle_name, platforms, ig_caption, asset_type ("image"), asset_url (Placid URL), approved flag | DONE |
| FR-603 | Inventory Log flags updated after post generation (posted_new_arrival = TRUE) | DONE |

---

## 4. Non-Functional Requirements

| ID | Requirement | Status |
|----|-------------|--------|
| NFR-01 | Pipeline completes within 5 minutes for 2 posts | MET |
| NFR-02 | Scraper handles inventory of up to 200 vehicles | MET |
| NFR-03 | System runs unattended daily (webhook-triggered) | MET |
| NFR-04 | Monthly operating cost under $50 | MET (~$35-40/mo) |
| NFR-05 | All credentials stored in Make.com connections (not hardcoded) | PARTIAL (API keys in HTTP module headers) |
| NFR-06 | Google OAuth connections may require periodic reauthorization | KNOWN LIMITATION |

---

## 5. Data Schema

### 5.1 Inventory Log (Google Sheets Tab)

| Column | Field | Type | Source |
|--------|-------|------|--------|
| A | stock_number | String | Scraper |
| B | title | String | Scraper (title_raw) |
| C | price | Number | Scraper |
| D | status | String | Scraper (available/sold) |
| E | first_seen | Date | Pipeline (M/D/YYYY) |
| F | last_seen | Date | Pipeline (M/D/YYYY) |
| G | posted_new_arrival | Boolean | Pipeline (TRUE after post) |
| H | posted_price_drop | Boolean | Pipeline |
| I | posted_sold | Boolean | Pipeline |
| J | original_price | Number | Scraper (first price recorded) |

### 5.2 Post Log (Google Sheets Tab)

| Column | Field | Type | Source |
|--------|-------|------|--------|
| A | date | Date | Pipeline (M/D/YYYY) |
| B | content_type | String | Pipeline (NEW_ARRIVAL, etc.) |
| C | stock_number | String | Scraper |
| D | vehicle_name | String | Pipeline |
| E | platforms | String | Static ("IG, FB, X, TikTok") |
| F | ig_caption | String | GPT-4o |
| G | asset_type | String | Pipeline ("image") |
| H | asset_url | URL | Placid API |
| I | approved | Boolean | Manual |
| J | metricool_id | String | Future (Phase 2) |

### 5.3 Scraper Output (Apify Dataset)

24 fields per vehicle: stock_number, year, make, model, trim, title_raw, price, original_price, price_raw, mileage, mileage_raw, vin, image_urls (array), vehicle_url, status, exterior_color, interior_color, fuel_type, days_in_stock, engine, transmission, body_style, drivetrain, is_new, vuid, scraped_at.

---

## 6. Cost Breakdown

| Service | Plan | Monthly Cost |
|---------|------|-------------|
| Apify | Free tier (sufficient) | $0 |
| Make.com | Core plan | ~$10 |
| OpenAI GPT-4o | Pay-as-you-go (~60 calls/mo) | ~$2 |
| Placid | Starter plan | $19 |
| Google Sheets | Free (Google Workspace) | $0 |
| Gmail | Free (Google Workspace) | $0 |
| **Total** | | **~$31/mo** |

---

## 7. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Google OAuth token expiration | Pipeline fails silently | Dead Man's Switch scenario (deferred); manual reauth in Make.com UI |
| CDN image URLs become stale | Placid image gen fails for that vehicle | `first(image_urls)` uses freshest image; stale = 404 only for sold/removed vehicles |
| Website structure changes | Scraper breaks | Scraper uses embedded JSON (not CSS selectors) — more resilient to layout changes |
| OpenAI API outage | No copy generated | Make.com stopOnHttpError halts that branch; other vehicles unaffected |
| Rate limit exceeded on Placid | Image gen queued instead of instant | Using create_now: true; Placid starter plan allows sufficient volume |

---

## 8. Future Roadmap

### Phase 2 — Deferred Items

| Item | Description | Priority |
|------|-------------|----------|
| Route B (Price Drop) | Connect price drop detection to the shared post chain | High |
| Scenario 3 (Dead Man's Switch) | Daily 6 PM check — if no successful run that day, send alert email | High |
| Sold Vehicle Detection | Detect vehicles removed from inventory and flag as sold | Medium |
| SOUL.md in GPT Prompt | Inject full brand voice document into OpenAI system prompt | Medium |
| Placid Template Refinement | Improve image layout, add logo overlay, branded colors | Medium |

### Phase 3 — Auto-Publish

| Item | Description | Priority |
|------|-------------|----------|
| Metricool Integration | Replace Gmail approval with direct social media scheduling | High |
| Creatomate Video Templates | Generate 15-second Reels/TikTok videos for inventory | Medium |
| Multi-image Carousels | Instagram carousel posts with multiple vehicle angles | Low |
| A/B Testing | Test different copy styles and measure engagement | Low |

---

## 9. Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Daily pipeline uptime | 95%+ | Make.com execution logs (no failures) |
| Posts generated per day | 2 new arrivals | Post Log row count per day |
| Content approval rate | 80%+ | approved=TRUE / total posts in Post Log |
| Time from inventory change to post ready | < 10 minutes | scraper finish → email delivery |
| Monthly cost | < $50 | Sum of all service invoices |

---

*This document describes the system as built and deployed. It should be updated as new features are added.*
