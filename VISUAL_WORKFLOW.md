# 🎨 LATERAL RECON - VISUAL WORKFLOW

## 📍 High-Level Process Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│                    LATERAL RECONNAISSANCE                           │
│                         WORKFLOW                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

        START: You have a target domain in bug bounty scope
                                │
                                ▼
        ┌───────────────────────────────────────────────┐
        │  STEP 1: DNS RESOLUTION                       │
        │  ─────────────────────                       │
        │  Input:  target.com                           │
        │  Tool:   dig +short                           │
        │  Output: 203.0.113.45                         │
        └───────────────┬───────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────────────┐
        │  STEP 2: REVERSE IP LOOKUP                    │
        │  ─────────────────────────                   │
        │  Input:  203.0.113.45                         │
        │  Tool:   VirusTotal API                       │
        │  Output: List of co-hosted domains            │
        │          ├─ target.com                        │
        │          ├─ client2.vendor.net               │
        │          ├─ test.client3.vendor.net          │
        │          └─ staging-client4.vendor.io        │
        └───────────────┬───────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────────────┐
        │  STEP 3: PATTERN ANALYSIS                     │
        │  ────────────────────                        │
        │  Count domain patterns:                       │
        │    47 vendor.net    ← VENDOR IDENTIFIED!     │
        │    12 hosting.io                              │
        │     5 cloud.com                               │
        └───────────────┬───────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────────────┐
        │  STEP 4: SUBNET CALCULATION                   │
        │  ──────────────────────                      │
        │  IP:     203.0.113.45                         │
        │  Subnet: 203.0.113.0/24                       │
        │  Range:  203.0.113.1 - 203.0.113.254         │
        └───────────────┬───────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────────────┐
        │  STEP 5: SUBNET SCANNING                      │
        │  ───────────────────                         │
        │  For each IP in range:                        │
        │    Query VT for domains                       │
        │    Filter for test keywords                   │
        │                                               │
        │  Results:                                     │
        │    .45  → target.com (prod)                  │
        │    .46  → staging-target.vendor.net ✓        │
        │    .47  → test.target.vendor.net ✓           │
        │    .48  → dev-target.vendor.net ✓            │
        └───────────────┬───────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────────────┐
        │  STEP 6: VERIFICATION & TESTING               │
        │  ──────────────────────────                  │
        │  ✓ Check scope (are test servers allowed?)   │
        │  ✓ Port scan                                  │
        │  ✓ Look for vulnerabilities                   │
        │  ✓ Report findings                            │
        └───────────────────────────────────────────────┘
                                │
                                ▼
                            SUCCESS!
```

---

## 🔄 Data Flow Detailed

### Input → Processing → Output Chain

```
┌────────────────┐     ┌──────────────┐     ┌────────────────┐
│  Target Domain │────▶│ DNS Resolver │────▶│   IP Address   │
│  example.com   │     │     (dig)    │     │  203.0.113.45  │
└────────────────┘     └──────────────┘     └────────┬───────┘
                                                      │
                              ┌───────────────────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │ VirusTotal API  │
                     │  Reverse Lookup │
                     └────────┬────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │   Co-hosted Domains List      │
              ├───────────────────────────────┤
              │ example.com                   │
              │ client-a.vendor.net           │
              │ client-b.vendor.net           │
              │ test.client-c.vendor.net      │ ← Pattern!
              │ staging-client-d.vendor.io    │ ← Pattern!
              │ www.client-e.vendor.net       │
              └───────────────┬───────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Pattern Analysis │
                    │   (awk + sort)   │
                    └────────┬─────────┘
                             │
                             ▼
                  ┌───────────────────────┐
                  │  Vendor Identified:   │
                  │    vendor.net (47)    │ ← Most common
                  │    vendor.io (12)     │
                  │    hosting.com (5)    │
                  └───────────┬───────────┘
                              │
                              ▼
                   ┌────────────────────┐
                   │ Subnet Extraction  │
                   │   203.0.113.0/24   │
                   └──────────┬─────────┘
                              │
                              ▼
                   ┌────────────────────┐
                   │  IP Range Scan     │
                   │  (254 iterations)  │
                   └──────────┬─────────┘
                              │
                              ▼
             ┌────────────────────────────────┐
             │   Test Instances Discovered    │
             ├────────────────────────────────┤
             │ 203.0.113.46 | test-*.net      │
             │ 203.0.113.47 | staging-*.net   │
             │ 203.0.113.48 | dev-*.io        │
             └────────────────────────────────┘
```

---

## 🎯 Attack Surface Discovery Flow

```
                        ┌─────────────┐
                        │   TARGET    │
                        │ example.com │
                        └──────┬──────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
    ┌────────┐          ┌────────────┐        ┌──────────┐
    │ Direct │          │   Vendor   │        │ Related  │
    │  Scan  │          │  Discovery │        │ Services │
    └────────┘          └──────┬─────┘        └──────────┘
         │                     │
         │                     ▼
         │            ┌─────────────────┐
         │            │ Vendor has 100  │
         │            │ other clients   │
         │            └────────┬────────┘
         │                     │
         │                     ▼
         │            ┌─────────────────────┐
         │            │ Find test instances │
         │            │ of YOUR target      │
         │            └────────┬────────────┘
         │                     │
         └─────────────────────┼──────────┐
                               │          │
                               ▼          ▼
                       ┌─────────────┐  ┌──────────────┐
                       │ Production  │  │ Test/Staging │
                       │   (Hard)    │  │    (Easy)    │
                       └─────────────┘  └──────────────┘
                                              │
                                              ▼
                                    ┌──────────────────┐
                                    │  Vulnerabilities │
                                    │  • Default creds │
                                    │  • Open ports    │
                                    │  • Debug enabled │
                                    │  • No auth       │
                                    └──────────────────┘
```

---

## 🗺️ Subnet Topology Visualization

### How Hosting Providers Allocate IPs

```
╔══════════════════════════════════════════════════════════════╗
║           Hosting Provider: "MegaHost Solutions"             ║
║             IP Block: 203.0.113.0/24 (256 IPs)              ║
╚══════════════════════════════════════════════════════════════╝

    203.0.113.0    ┌─────────────────────────────────┐
                   │  Reserved (Network Address)     │
    203.0.113.1    ├─────────────────────────────────┤
                   │  Gateway / Router               │
    203.0.113.2-10 ├─────────────────────────────────┤
                   │  Infrastructure (DNS, etc.)     │
    ───────────────┼─────────────────────────────────┤
    203.0.113.11   │                                 │
    203.0.113.12   │  CLIENT A - Production          │
    203.0.113.13   │  ├─ Web server                  │
    203.0.113.14   │  ├─ API server                  │
    203.0.113.15   │  └─ Database server             │
    ───────────────┼─────────────────────────────────┤
    203.0.113.16   │  CLIENT A - Test Env ⭐         │
    203.0.113.17   │  └─ Staging server ⭐           │
    ───────────────┼─────────────────────────────────┤
    203.0.113.18   │                                 │
    203.0.113.19   │  CLIENT B - Production          │
    203.0.113.20   │  └─ App servers                 │
    ───────────────┼─────────────────────────────────┤
    203.0.113.40   │                                 │
    203.0.113.41   │  YOUR TARGET - Production       │ ← Start here
    203.0.113.42   │  ├─ Main site                   │
    203.0.113.43   │  ├─ API                         │
    203.0.113.44   │  └─ CDN origin                  │
    ───────────────┼─────────────────────────────────┤
    203.0.113.45   │  YOUR TARGET - Test ⭐⭐⭐      │ ← FOUND!
    203.0.113.46   │  ├─ Staging site ⭐⭐⭐          │
    203.0.113.47   │  ├─ Dev environment ⭐⭐⭐       │
    203.0.113.48   │  └─ QA server ⭐⭐⭐             │
    ───────────────┼─────────────────────────────────┤
    203.0.113.49   │  CLIENT C - Services            │
       ...         │         ...                     │
    203.0.113.254  │  Reserved (Broadcast)           │
                   └─────────────────────────────────┘

⭐ = Potential High-Value Target (Weaker Security)
```

---

## 🔍 Pattern Matching Logic

### Keyword Filtering Process

```
ALL DOMAINS FROM IP           FILTER APPLIED              RESULTS
─────────────────────        ─────────────────           ────────────

www.client1.vendor.net                                   (excluded)
                                    │
api.client2.vendor.net       grep -iE "test|             (excluded)
                                    │  staging|
prod.client3.vendor.io              │  dev|              (excluded)
                                    │  uat|
TEST.client4.vendor.net ────────────┴──► qa|          ✓  TEST...
                                         demo|
staging-api.client5.net ────────────────► "           ✓  staging...
                                                      
dev-portal.client6.io ──────────────────────►         ✓  dev...
                                                      
uat.client7.vendor.com ─────────────────────►         ✓  uat...
                                                      
www.production.net                                       (excluded)

demo-app.client8.io ────────────────────────►         ✓  demo...
```

### Why These Keywords?

```
test      → Developers create test instances
staging   → Pre-production environment
dev       → Development servers
uat       → User Acceptance Testing
qa        → Quality Assurance
demo      → Demonstration/sales environments
sandbox   → Isolated testing environment
preprod   → Pre-production staging
beta      → Beta testing phase
alpha     → Alpha testing phase
```

---

## 📊 Decision Tree

### Should I Test This Instance?

```
                  ┌─────────────────────┐
                  │ Found test instance │
                  └──────────┬──────────┘
                             │
                  ┌──────────▼──────────┐
                  │ Is it in written    │
                  │ program scope?      │
                  └──────┬──────┬───────┘
                         │      │
                    YES  │      │  NO
                         │      │
                ┌────────▼──┐   └────────┐
                │ Proceed   │            │
                │ carefully │     ┌──────▼────────┐
                └─────┬─────┘     │ DON'T TEST!   │
                      │           │ Ask program   │
                      │           │ first         │
                      │           └───────────────┘
          ┌───────────▼────────────┐
          │ Check scope document:  │
          │ *.target.com ?         │
          └───────┬───────┬────────┘
                  │       │
             YES  │       │  UNCLEAR
                  │       │
         ┌────────▼───┐   └──────────┐
         │ Test it!   │              │
         │ (ethically)│   ┌──────────▼───────────┐
         └────────────┘   │ Message program:     │
                          │ "Can I test          │
                          │ test.target.com?"    │
                          └──────────────────────┘
```

---

## 🎯 Complete Workflow (With Timings)

```
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  FULL LATERAL RECON TIMELINE (Free VT API)           ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛

00:00  ├─ Install tools (if needed)           [5 min]
00:05  ├─ Get VT API key (if needed)           [5 min]
00:10  ├─ Configure script                     [2 min]
       │
00:12  ├─ START SCRIPT
       │
00:13  ├─ DNS resolution                       [1 sec]
       │   └─ Result: IP address
       │
00:14  ├─ Reverse IP lookup (VT API)          [15 sec]
       │   └─ Result: Co-hosted domains list
       │
00:15  ├─ Pattern analysis                     [5 sec]
       │   └─ Result: Vendor identified
       │
00:16  ├─ Subnet calculation                   [1 sec]
       │   └─ Result: IP range defined
       │
00:17  ├─ Choose scan mode
       │   ├─ Full scan (254 IPs)              [63 min]
       │   │   (254 × 15 sec = 3,810 sec)
       │   │
       │   └─ Partial scan (10 IPs)            [2.5 min]
       │       (10 × 15 sec = 150 sec)
       │
01:20  ├─ Generate report                      [10 sec]
       │
01:21  └─ COMPLETE
              └─ Review findings               [10 min]

TOTAL TIME:
  • Partial scan: ~35 minutes
  • Full scan:    ~90 minutes
```

---

## 🚦 Risk Assessment Matrix

```
┌─────────────────────────────────────────────────────────────┐
│              TESTING RISK LEVELS                            │
├────────────────┬────────────────────────────────────────────┤
│                │  PROD    │  TEST    │  STAGING  │  DEV     │
├────────────────┼──────────┼──────────┼───────────┼──────────┤
│ Default Creds  │  🔴 Rare │  🟢 Common│ 🟡 Sometimes│🟢 Common│
├────────────────┼──────────┼──────────┼───────────┼──────────┤
│ Open Ports     │  🔴 Rare │  🟢 Common│ 🟡 Sometimes│🟢 Common│
├────────────────┼──────────┼──────────┼───────────┼──────────┤
│ Debug Mode     │  🔴 Never│  🟢 Often │ 🟡 Sometimes│🟢 Always│
├────────────────┼──────────┼──────────┼───────────┼──────────┤
│ Auth Bypass    │  🔴 Rare │  🟡 Possible│🟡 Possible│🟢 Common│
├────────────────┼──────────┼──────────┼───────────┼──────────┤
│ Exposed Admins │  🔴 Rare │  🟢 Common│ 🟢 Common │🟢 Always│
├────────────────┼──────────┼──────────┼───────────┼──────────┤
│ .git Exposure  │  🔴 Never│  🟡 Sometimes│🟢 Often │🟢 Common│
├────────────────┼──────────┼──────────┼───────────┼──────────┤
│ Directory List │  🔴 Never│  🟡 Sometimes│🟢 Often │🟢 Always│
└────────────────┴──────────┴──────────┴───────────┴──────────┘

🔴 = Low probability    (Hard to find)
🟡 = Medium probability (Possible)
🟢 = High probability   (Common)
```

---

## 🎓 Learning Path Visualization

```
BEGINNER                  INTERMEDIATE              ADVANCED
   │                           │                       │
   ├─ Week 1                   ├─ Week 2               ├─ Week 3-4
   │  └─ Understand DNS        │  └─ Pattern Analysis  │  └─ Full Automation
   │     • dig commands        │     • Regex mastery   │     • Pipeline building
   │     • Basic lookups       │     • Data analysis   │     • Multi-tool integration
   │                           │                       │
   ▼                           ▼                       ▼
┌──────────┐              ┌─────────┐              ┌───────────┐
│ Run      │             │ Modify  │              │ Build     │
│ Script   │──────────▶│ Script  │──────────▶│ Framework │
│ (Manual) │             │ (Custom)│              │ (Auto)    │
└──────────┘              └─────────┘              └───────────┘
```

---

## 📋 Mental Model: How Attackers Think

```
DEFENDER MINDSET                ATTACKER MINDSET
────────────────                ────────────────

"Secure our main site"          "Find the test sites"
        │                               │
        │                               │
   ┌────▼────┐                     ┌────▼────┐
   │ Prod    │                     │ Search  │
   │ Server  │                     │ Adjacent│
   │         │                     │ IPs     │
   │ ✓ Patched│                    └────┬────┘
   │ ✓ Monitored│                       │
   │ ✓ Firewalled│              ┌───────▼────────┐
   └─────────┘                  │ Find test site │
                                │ • No patches   │
                                │ • No monitoring│
                                │ • Default creds│
                                └────────────────┘

KEY INSIGHT: Lateral thinking finds weak points!
```

---

## 🎯 Success Metrics

### How to Measure Your Progress

```
RECONNAISSANCE QUALITY SCORE
════════════════════════════

┌─────────────────────────────────────────────┬────────┐
│ Metric                                      │ Points │
├─────────────────────────────────────────────┼────────┤
│ Found vendor infrastructure                 │   +10  │
│ Identified co-hosted domains               │   +10  │
│ Discovered test instances                   │   +20  │
│ Verified instances are in scope            │   +15  │
│ Documented complete methodology             │   +15  │
│ Found actual vulnerabilities                │   +30  │
│ Reported responsibly                        │   +20  │
├─────────────────────────────────────────────┼────────┤
│ TOTAL POSSIBLE                              │   120  │
└─────────────────────────────────────────────┴────────┘

RANKING:
• 0-30:   Beginner (Keep practicing!)
• 31-60:  Intermediate (Good progress!)
• 61-90:  Advanced (Excellent work!)
• 91-120: Expert (You're crushing it!)
```

---

Remember: This visual guide is your mental map. Print it, reference it, master it! 🎯
