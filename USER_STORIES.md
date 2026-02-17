# User Stories
# AustinEAutos Content Engine

**Version:** 1.0
**Date:** February 16, 2026

---

## Epic 1: Inventory Monitoring

### US-101: Daily Inventory Scraping
**As a** dealership operator,
**I want** the system to automatically scrape my website inventory every morning,
**So that** I don't have to manually check for new vehicles or changes.

**Acceptance Criteria:**
- [x] Scraper runs daily at 8:00 AM CST via Apify scheduled task
- [x] All vehicle detail pages are discovered from the sitemap
- [x] Each vehicle record captures: stock number, year, make, model, trim, price, mileage, VIN, images, URL, status, colors, fuel type, days in stock
- [x] Scraper handles up to 200 vehicles per run
- [x] Results stored in Apify dataset as structured JSON
- [x] Make.com webhook triggered automatically on scraper completion

**Status:** DONE

---

### US-102: New Arrival Detection
**As a** dealership operator,
**I want** the system to automatically detect when a new vehicle appears in my inventory,
**So that** I can post about it on social media the same day it's listed.

**Acceptance Criteria:**
- [x] Each scraped vehicle is looked up in the Inventory Log by stock_number + today's date
- [x] Vehicles not found in the log are classified as NEW_ARRIVAL
- [x] New arrivals are added as new rows in the Inventory Log with first_seen = today
- [x] New arrivals trigger the content generation pipeline

**Status:** DONE

---

### US-103: Price Drop Detection
**As a** dealership operator,
**I want** the system to detect when a vehicle's price drops,
**So that** I can create "price drop" social media posts to drive interest.

**Acceptance Criteria:**
- [x] Vehicles already in log with a lower current price are classified as PRICE_DROP
- [x] The Inventory Log row is updated with the new price
- [ ] Price drop triggers content generation with PRICE_DROP template
- [ ] Price drop posts mention the old price and savings amount

**Status:** PARTIAL — Route B built but not connected to the shared post chain (deferred)

---

### US-104: Still Available Tracking
**As a** dealership operator,
**I want** the system to track vehicles that remain in inventory day-over-day,
**So that** I have an accurate record of how long each vehicle has been listed.

**Acceptance Criteria:**
- [x] Vehicles found with same or higher price are classified as STILL_AVAILABLE
- [x] The Inventory Log row's last_seen date is updated to today
- [x] No content is generated for still-available vehicles (no post noise)

**Status:** DONE

---

## Epic 2: AI Content Generation

### US-201: Multi-Platform Copy Generation
**As a** dealership operator,
**I want** AI-generated social media copy for each new arrival,
**So that** I have ready-to-post content for all my social media channels without writing it myself.

**Acceptance Criteria:**
- [x] GPT-4o generates copy for 5 platforms: Instagram, Facebook, X, TikTok, Reel script
- [x] Each platform's copy follows its specific format and character limits
- [x] Instagram: up to 2200 chars with hashtags and line breaks
- [x] Facebook: 3-4 conversational sentences plus hashtags
- [x] X (Twitter): max 280 characters including hashtags
- [x] TikTok: max 150 chars, casual trending style
- [x] Reel: 15-second voiceover script, conversational
- [x] Copy includes vehicle name, price, and listing URL

**Status:** DONE

---

### US-202: Brand Voice Alignment
**As a** dealership operator,
**I want** all generated content to match my dealership's brand voice,
**So that** posts feel authentic and consistent with how we actually communicate.

**Acceptance Criteria:**
- [x] SOUL.md brand voice document created from Instagram feed and Google reviews analysis
- [x] GPT-4o system prompt enforces brand voice rules
- [x] No high-pressure sales language ("ACT NOW", "LIMITED TIME", etc.)
- [x] Mentions no-haggle pricing policy naturally
- [x] References indoor showroom when relevant
- [x] Uses Austin/Round Rock local references
- [x] Hashtags always include #AustinEAutos #AustinEV #RoundRockTX
- [x] Never fabricates specs or features

**Status:** DONE

---

### US-203: Rate Limiting
**As a** dealership operator,
**I want** the system to limit content generation to 2 posts per daily run,
**So that** I'm not overwhelmed with approval emails and my social feeds aren't flooded.

**Acceptance Criteria:**
- [x] Maximum 2 vehicles per run trigger content generation
- [x] Rate limit uses Iterator bundle index (`__IMTINDEX__ <= 2`), not `__IMTLENGTH__` (which is broken/ambiguous in Make.com)
- [x] Remaining vehicles are still logged in Inventory Log but don't generate posts

**Status:** DONE

---

## Epic 3: Image Generation

### US-301: Branded Social Media Images
**As a** dealership operator,
**I want** a branded image automatically generated for each social post,
**So that** my posts look professional and on-brand without manual design work.

**Acceptance Criteria:**
- [x] 1200x1200 branded image generated via Placid API
- [x] Image includes vehicle name as title text
- [x] Image includes price as subline text
- [x] Image includes the vehicle's primary photo from the inventory CDN
- [x] Image generation is synchronous (no polling needed)
- [x] Generated image URL is stored in the Post Log

**Status:** DONE

---

### US-302: Image in Approval Email
**As a** dealership operator,
**I want** to see the generated image directly in the approval email,
**So that** I can review both the copy and the image without clicking through to another tool.

**Acceptance Criteria:**
- [x] Generated Placid image displayed inline in the Gmail approval email
- [x] Image is clickable (links to full-size version)
- [x] Image renders at 400px width for email readability

**Status:** DONE

---

## Epic 4: Approval Workflow

### US-401: Email-Based Content Review
**As a** dealership operator,
**I want** to receive an email with the generated content for each vehicle,
**So that** I can review, approve, or modify the content before it goes live.

**Acceptance Criteria:**
- [x] Email sent to bryan@onadabros.com for each generated post
- [x] Subject line: "{content_type}: {vehicle_name} --- AustinEAutos"
- [x] Email body includes: content type, vehicle name, generated image, Instagram caption, Facebook post, X post, TikTok caption, Reel script
- [x] Each platform's content is in its own labeled section
- [x] Email includes a note that content will be logged to Post Log

**Status:** DONE

---

## Epic 5: Post Tracking

### US-501: Post Log
**As a** dealership operator,
**I want** every generated post to be logged in a spreadsheet,
**So that** I can track what content has been created, for which vehicles, and whether it was approved.

**Acceptance Criteria:**
- [x] Each post logged with: date, content_type, stock_number, vehicle_name, platforms, Instagram caption, asset_type, asset_url, approved flag
- [x] Asset type is "image" and asset URL is the Placid-generated image URL
- [x] Log is in the "Post Log" tab of the shared Google Sheet

**Status:** DONE

---

### US-502: Inventory Flag Updates
**As a** dealership operator,
**I want** the Inventory Log to track which posts have been generated for each vehicle,
**So that** the system doesn't create duplicate posts for the same vehicle and event.

**Acceptance Criteria:**
- [x] After generating a NEW_ARRIVAL post, the Inventory Log's "posted_new_arrival" column is set to TRUE
- [x] The system can be extended to set posted_price_drop and posted_sold flags

**Status:** DONE

---

## Epic 6: System Reliability (Deferred)

### US-601: Dead Man's Switch
**As a** dealership operator,
**I want** to be alerted if the pipeline fails to run on any given day,
**So that** I can investigate and fix issues before they compound.

**Acceptance Criteria:**
- [ ] Scenario 3 runs daily at 6:00 PM CST
- [ ] Checks if Scenario 1 had a successful execution that day
- [ ] If no successful run found, sends alert email to bryan@onadabros.com
- [ ] Alert includes: scenario name, last successful run date, error details if available

**Status:** NOT STARTED (deferred)

---

### US-602: Sold Vehicle Detection
**As a** dealership operator,
**I want** the system to detect when a vehicle is removed from inventory,
**So that** I can create "SOLD" celebration posts and update my records.

**Acceptance Criteria:**
- [ ] Compare current scrape results against Inventory Log to find vehicles no longer listed
- [ ] Flag removed vehicles as "sold" in Inventory Log
- [ ] Optionally trigger a JUST_SOLD content generation

**Status:** NOT STARTED (deferred)

---

## Epic 7: Auto-Publishing (Phase 2)

### US-701: Direct Social Media Publishing
**As a** dealership operator,
**I want** approved posts to be automatically published to my social media accounts,
**So that** I don't have to manually copy-paste content from emails to each platform.

**Acceptance Criteria:**
- [ ] Integration with Metricool (or similar) for scheduling posts
- [ ] Posts scheduled to optimal times per platform
- [ ] Approved flag in Post Log triggers publishing
- [ ] Published post IDs stored in Post Log (metricool_id column)

**Status:** NOT STARTED (Phase 2)

---

### US-702: Video Content Generation
**As a** dealership operator,
**I want** short-form video content generated for Reels and TikTok,
**So that** I can engage audiences on video-first platforms.

**Acceptance Criteria:**
- [ ] 15-second vertical video (1080x1920) generated via Creatomate
- [ ] Video includes multiple vehicle images, text overlays, branded elements
- [ ] Reel script from GPT-4o used as voiceover or caption
- [ ] Video URL stored in Post Log

**Status:** NOT STARTED (Phase 2 — after image MVP is validated)

---

## Story Map Summary

```
                    DONE                          DEFERRED / PHASE 2
               ┌─────────────┐              ┌──────────────────────┐
  Inventory    │ US-101 Scrape│              │ US-103 Price Drop    │
  Monitoring   │ US-102 New   │              │   (route built,      │
               │ US-104 Still │              │    not connected)    │
               └─────────────┘              │ US-602 Sold Detect   │
                                            └──────────────────────┘
               ┌─────────────┐              ┌──────────────────────┐
  Content      │ US-201 Copy  │              │ US-702 Video Gen     │
  Generation   │ US-202 Voice │              │                      │
               │ US-203 Limit │              │                      │
               └─────────────┘              └──────────────────────┘
               ┌─────────────┐
  Image Gen    │ US-301 Image │
               │ US-302 Email │
               └─────────────┘
               ┌─────────────┐              ┌──────────────────────┐
  Workflow     │ US-401 Email │              │ US-601 Dead Man's    │
               │ US-501 Log   │              │ US-701 Auto-Publish  │
               │ US-502 Flags │              │                      │
               └─────────────┘              └──────────────────────┘
```

---

*12 stories complete, 5 stories deferred. MVP is fully operational.*
