# LATERAL RECONNAISSANCE - COMPLETE EDUCATIONAL GUIDE

## 📚 Table of Contents
1. [Core Concepts Explained](#core-concepts)
2. [How Each Command Works](#command-breakdown)
3. [Understanding the Data Flow](#data-flow)
4. [API Deep Dive](#api-details)
5. [Pattern Recognition Guide](#pattern-recognition)
6. [Hands-On Exercises](#exercises)

---

## 🎓 CORE CONCEPTS EXPLAINED

### What is Lateral Reconnaissance?

**Traditional Recon:**
```
You → Scan target.com directly
```

**Lateral Recon:**
```
You → Find target's vendor → Scan vendor infrastructure → Find test instances
```

**Why it works:**
- Companies outsource hosting to vendors (AWS, Digital Ocean, managed hosting)
- Vendors host MULTIPLE clients on same infrastructure
- Test/staging servers often have weaker security
- Vendors group related servers in IP ranges

**Real-world analogy:**
```
Target: A bank's production website
↓
Find: The bank uses "MegaHost Solutions" for hosting
↓
Discover: MegaHost has 100 other clients
↓
Find: The bank's test server is at test-bank.megahost.com
↓
Result: Test server has default passwords, exposed admin panel
```

---

### Understanding IP Addresses and Subnets

**IP Address Structure:**
```
203.0.113.45
│   │  │  │
│   │  │  └─ Host (1-254) ← Individual server
│   │  └──── Network part
│   └─────── Network part
└────────── Network part
```

**Subnet Notation (/24):**
```
203.0.113.0/24 means:
- First 24 bits are fixed (203.0.113)
- Last 8 bits can vary (0-255)
- Total: 256 addresses (actually 254 usable)
- Range: 203.0.113.1 to 203.0.113.254
```

**Why vendors use subnets:**
```
Vendor gets: 203.0.113.0/24 (256 IPs)
Allocation:
  203.0.113.1-10:   Infrastructure (DNS, gateways)
  203.0.113.11-50:  Client A's servers
  203.0.113.51-90:  Client B's servers
  203.0.113.91-130: Client C's servers
  203.0.113.131-170: Your target ← You found .145
  203.0.113.171-210: Client E's servers
  203.0.113.211-254: Test/staging pool ← JACKPOT!
```

---

### DNS Resolution Deep Dive

**What happens when you type a domain?**

```bash
You type: example.com
│
├─> Your computer asks: "Where is example.com?"
│   DNS Server responds: "It's at 203.0.113.45"
│
└─> Your computer connects to 203.0.113.45
```

**The `dig` command:**
```bash
dig +short example.com
```

**What it does:**
1. Queries configured DNS servers (usually your ISP's)
2. Asks: "What IP does example.com point to?"
3. Returns the answer (just the IP with +short flag)

**Full dig output (without +short):**
```
; <<>> DiG 9.18.1 <<>> example.com
;; ANSWER SECTION:
example.com.    3600    IN    A    203.0.113.45
                  ↑      ↑   ↑      ↑
                  │      │   │      └─ IP address
                  │      │   └──────── Record type (A = IPv4)
                  │      └──────────── Class (IN = Internet)
                  └─────────────────── TTL (time to cache)
```

**Why we use grep:**
```bash
dig +short example.com | grep -E '^[0-9]'
```
- Filters out non-IP lines (like CNAME records)
- Ensures we only get lines starting with digits
- Regular expression: ^[0-9] = "line starts with number"

---

### Reverse IP Lookup Explained

**Normal DNS:**
```
Domain → IP
example.com → 203.0.113.45
```

**Reverse lookup:**
```
IP → All domains
203.0.113.45 → [example.com, client2.com, test.client3.com, ...]
```

**Why this reveals vendor relationships:**

```
You query: 203.0.113.45
Results show:
  - example.com (your target)
  - client2-website.com
  - app.startup.io
  - test-banking.megahost.net ← Notice the pattern!
  - staging-ecommerce.megahost.net ← Same vendor!

Pattern spotted: *.megahost.net
Conclusion: They all use "MegaHost" for hosting
```

**VirusTotal stores this data by:**
- Crawling the entire internet
- Recording DNS resolutions over time
- Building historical maps of IP ↔ Domain relationships

---

## 🔧 COMMAND BREAKDOWN

### Command 1: Get IP from Domain
```bash
IP=$(dig +short $TARGET | grep -E '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$' | head -1)
```

**Step-by-step breakdown:**

1. **`dig +short $TARGET`**
   - Asks DNS: "What's the IP of $TARGET?"
   - Returns: IP addresses and/or CNAME records
   
2. **`grep -E '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$'`**
   - Regular expression filter
   - `^` = Start of line
   - `[0-9]+` = One or more digits
   - `\.` = Literal dot (escaped)
   - `$` = End of line
   - Ensures we ONLY get valid IPv4 addresses
   
3. **`head -1`**
   - Takes the first line only
   - Useful when multiple IPs are returned
   
4. **`IP=$(...)`**
   - Stores the result in variable IP

**Example execution:**
```bash
$ dig +short example.com
93.184.216.34
2606:2800:220:1:248:1893:25c8:1946

$ dig +short example.com | grep -E '^[0-9]'
93.184.216.34  ← Only the IPv4 address

$ IP=$(dig +short example.com | grep -E '^[0-9]' | head -1)
$ echo $IP
93.184.216.34  ← Stored in variable
```

---

### Command 2: Reverse IP via VirusTotal API
```bash
curl -s "https://www.virustotal.com/api/v3/ip_addresses/$IP/resolutions" \
  -H "x-apikey: $VT_API_KEY" | jq -r '.data[].attributes.host_name'
```

**Breakdown:**

1. **`curl -s`**
   - Makes HTTP request
   - `-s` = Silent mode (no progress bar)

2. **`"https://www.virustotal.com/api/v3/ip_addresses/$IP/resolutions"`**
   - VirusTotal API endpoint
   - `/resolutions` = Get all domains that resolved to this IP

3. **`-H "x-apikey: $VT_API_KEY"`**
   - HTTP Header
   - Authenticates you with VirusTotal
   - Required for API access

4. **`| jq -r '.data[].attributes.host_name'`**
   - Pipes response to jq (JSON processor)
   - `.data[]` = For each item in data array
   - `.attributes.host_name` = Extract the hostname field
   - `-r` = Raw output (no quotes)

**Example API Response:**
```json
{
  "data": [
    {
      "attributes": {
        "host_name": "example.com",
        "ip_address": "203.0.113.45",
        "date": 1640995200
      },
      "type": "resolution"
    },
    {
      "attributes": {
        "host_name": "test.example.com",
        "ip_address": "203.0.113.45",
        "date": 1640908800
      },
      "type": "resolution"
    }
  ]
}
```

**What jq extracts:**
```bash
example.com
test.example.com
```

---

### Command 3: Pattern Analysis
```bash
cat co_hosted.txt | awk -F. '{print $(NF-1)"."$NF}' | sort | uniq -c | sort -rn
```

**Step-by-step:**

1. **`cat co_hosted.txt`**
   - Read file contents
   - Sends each line to next command

2. **`awk -F. '{print $(NF-1)"."$NF}'`**
   - AWK = Text processing language
   - `-F.` = Field separator is dot (.)
   - `NF` = Number of Fields
   - `$(NF-1)` = Second-to-last field
   - `$NF` = Last field
   
   **Example:**
   ```
   Input:  test.client.megahost.com
   Fields: [test] [client] [megahost] [com]
                           ↑NF-1      ↑NF
   Output: megahost.com
   ```

3. **`sort`**
   - Alphabetically arranges lines
   - Groups identical lines together

4. **`uniq -c`**
   - Counts consecutive duplicate lines
   - `-c` = Prepend count
   
   **Example:**
   ```
   megahost.com
   megahost.com
   megahost.com  ← After sort
   vendor2.net
   vendor2.net
   
   After uniq -c:
   3 megahost.com
   2 vendor2.net
   ```

5. **`sort -rn`**
   - `-r` = Reverse order (highest first)
   - `-n` = Numeric sort
   
   **Final output:**
   ```
   47 megahost.com      ← Most common (likely the vendor!)
   12 cloudprovider.io
   5  digitalocean.com
   2  aws.amazon.com
   1  standalone.org
   ```

---

### Command 4: Subnet Extraction
```bash
SUBNET=$(echo $IP | awk -F. '{print $1"."$2"."$3}')
```

**Breakdown:**

1. **`echo $IP`**
   - Outputs the IP address
   - Example: 203.0.113.45

2. **`awk -F. '{print $1"."$2"."$3}'`**
   - `-F.` = Split by dot
   - `$1` = First field (203)
   - `$2` = Second field (0)
   - `$3` = Third field (113)
   - Reconstructs: 203.0.113

**Visual representation:**
```
IP:     203.0.113.45
        │   │  │  │
Field:  $1 $2 $3 $4

SUBNET: $1.$2.$3
Result: 203.0.113
```

---

### Command 5: Subnet Scanning Loop
```bash
for i in {1..254}; do
  TEST_IP="$SUBNET.$i"
  curl -s "https://www.virustotal.com/api/v3/ip_addresses/$TEST_IP/resolutions" \
    -H "x-apikey: $VT_API_KEY" | \
    jq -r '.data[].attributes.host_name' | \
    grep -iE "test|staging|dev|uat|qa|demo"
  sleep 1
done
```

**Breakdown:**

1. **`for i in {1..254}`**
   - Bash brace expansion
   - Creates sequence: 1, 2, 3, ..., 254
   - Loops 254 times

2. **`TEST_IP="$SUBNET.$i"`**
   - Constructs IP addresses
   - Iteration 1: 203.0.113.1
   - Iteration 2: 203.0.113.2
   - ...
   - Iteration 254: 203.0.113.254

3. **`curl ... | jq ... | grep -iE "test|staging|dev"`**
   - Same as before, but adds grep filter
   - `-i` = Case-insensitive
   - `-E` = Extended regex (allows |)
   - `test|staging|dev` = Match ANY of these words

4. **`sleep 1`**
   - Pauses 1 second
   - Prevents hitting API rate limits
   - VirusTotal free tier: 4 requests/minute

**Pattern matching explained:**
```
grep -iE "test|staging|dev|uat|qa|demo"

Will match:
✓ test.example.com
✓ staging-api.vendor.net
✓ dev-portal.client.io
✓ uat.banking.com
✓ qa-environment.app
✓ demo.product.org
✓ TEST-SERVER.company.com (case-insensitive)

Will NOT match:
✗ www.example.com
✗ api.production.net
✗ secure.client.io
```

---

## 📊 DATA FLOW VISUALIZATION

### Complete Reconnaissance Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    1. TARGET INPUT                          │
│                   User provides: example.com                │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                   2. DNS RESOLUTION                         │
│    dig +short example.com → 203.0.113.45                   │
│    Store in $IP variable                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              3. REVERSE IP LOOKUP (VT API)                  │
│    Query: IP 203.0.113.45 → What domains?                  │
│    Response: [example.com, client2.com, test.client3.com]  │
│    Save to: co_hosted.txt                                   │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                 4. PATTERN ANALYSIS                         │
│    Extract second-level domains (SLDs)                      │
│    Count frequency → Find vendor                            │
│    Result: 47 domains use "megahost.com"                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                 5. SUBNET EXTRACTION                        │
│    IP: 203.0.113.45 → Subnet: 203.0.113.0/24               │
│    Hypothesis: Nearby IPs host related servers              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                 6. SUBNET SCANNING                          │
│    For each IP in 203.0.113.1-254:                         │
│      - Query VT for domains                                 │
│      - Filter for test keywords                             │
│      - Save matches                                         │
│    Result: Found test-example.megahost.com at .212         │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                  7. REPORT GENERATION                       │
│    Summary of findings                                      │
│    Next steps for exploitation                              │
│    Save to output directory                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔍 PATTERN RECOGNITION GUIDE

### Common Test/Staging Naming Patterns

**Prefix patterns:**
```
test-{company}.{vendor}.com
staging-{company}.{vendor}.net
dev-{company}.{vendor}.io
uat-{company}.{vendor}.cloud
qa-{company}.{vendor}.org
demo-{company}.{vendor}.app
sandbox-{company}.{vendor}.tech
```

**Subdomain patterns:**
```
test.{company}.com
staging.{company}.com
dev.{company}.com
{company}-test.com
{company}-staging.net
```

**Environment indicators:**
```
preprod-*     (pre-production)
beta-*        (beta testing)
alpha-*       (alpha testing)
int-*         (integration)
sit-*         (system integration test)
perf-*        (performance testing)
load-*        (load testing)
```

**Number-based patterns:**
```
test1.*, test2.*       (Multiple test instances)
staging-v2.*           (Version-specific)
dev-2024.*             (Year-based)
```

### Vendor Identification Clues

**Hosting provider patterns:**
```
*.digitalocean.com
*.linode.com
*.vultr.com
*.ovh.net
*.hetzner.com
*-{region}.compute.amazonaws.com
*.azurewebsites.net
*.cloudapp.net
```

**Managed hosting patterns:**
```
client*.hostinger.com
*.siteground.biz
*.wpengine.com
*.pantheonsite.io
*.acquia-sites.com
```

---

## 🎯 HANDS-ON EXERCISES

### Exercise 1: Basic DNS Resolution
```bash
# Task: Resolve these domains and note their IPs
dig +short google.com
dig +short github.com
dig +short youtube.com

# Question: Do multiple domains share IPs?
```

### Exercise 2: Manual Reverse Lookup
```bash
# Get your VirusTotal API key first
VT_KEY="your_key_here"

# Pick an IP (Google's DNS)
IP="8.8.8.8"

# Query VT
curl -s "https://www.virustotal.com/api/v3/ip_addresses/$IP/resolutions" \
  -H "x-apikey: $VT_KEY" | jq '.data[].attributes.host_name' | head -20

# Question: What domains use this IP?
```

### Exercise 3: Pattern Extraction Practice
```bash
# Create test data
cat > test_domains.txt << EOF
test.client1.megahost.com
staging.client2.megahost.com
www.client3.megahost.com
dev.client4.vendor2.net
prod.client5.vendor2.net
EOF

# Extract second-level domains
cat test_domains.txt | awk -F. '{print $(NF-1)"."$NF}' | sort | uniq -c

# Expected output:
# 3 megahost.com
# 2 vendor2.net
```

### Exercise 4: Regex Filtering Practice
```bash
# Test your grep patterns
echo "test.example.com" | grep -iE "test|staging"  # Should match
echo "prod.example.com" | grep -iE "test|staging"  # Should NOT match
echo "STAGING-API.example.com" | grep -iE "test|staging"  # Should match
```

---

## 🚨 TROUBLESHOOTING GUIDE

### Problem: "No IP resolved"
**Causes:**
- Domain doesn't exist
- DNS server unreachable
- Domain behind CDN (returns CDN IP, not origin)

**Solutions:**
```bash
# Try alternative DNS servers
dig @8.8.8.8 +short example.com  # Google DNS
dig @1.1.1.1 +short example.com  # Cloudflare DNS

# Use nslookup instead
nslookup example.com

# Check if domain exists
whois example.com
```

### Problem: "API returns error 401"
**Cause:** Invalid API key

**Solution:**
```bash
# Verify your API key
echo $VT_API_KEY

# Test with curl
curl -s "https://www.virustotal.com/api/v3/domains/google.com" \
  -H "x-apikey: $VT_API_KEY"

# Should return JSON, not error
```

### Problem: "No co-hosted domains found"
**Causes:**
- IP is dedicated (not shared hosting)
- VirusTotal has limited data
- IP is very new

**Solutions:**
```bash
# Try other reverse IP tools
# Shodan
shodan host YOUR_IP

# Bing search
# Visit: https://www.bing.com/search?q=ip:YOUR_IP

# ViewDNS
# Visit: https://viewdns.info/reverseip/
```

### Problem: "Rate limit exceeded"
**Cause:** Too many requests too fast

**Solution:**
```bash
# Increase sleep delay
sleep 15  # Instead of sleep 1

# Or use premium API key
# VT Premium: 1000 requests/min
```

---

## 📝 FINAL CHECKLIST

Before running the script on a real target:

- [ ] I have a valid VirusTotal API key
- [ ] I understand what each command does
- [ ] I've tested on practice domains
- [ ] I've verified the target is in scope
- [ ] I have permission to perform reconnaissance
- [ ] I understand rate limits (4 req/min for free tier)
- [ ] I'm prepared to wait ~1 hour for full subnet scan
- [ ] I know how to interpret the results
- [ ] I will report findings responsibly

---

## 🎓 NEXT LEARNING STEPS

1. **Week 1:** Practice manual queries
   - Resolve 20 different domains
   - Perform reverse IP on each
   - Document patterns you find

2. **Week 2:** Automation
   - Run the script on 5 targets
   - Modify it to add custom features
   - Build your own tools

3. **Week 3:** Advanced techniques
   - Learn certificate transparency log mining
   - Study Shodan/Censys advanced searches
   - Explore cloud-specific recon (AWS, Azure, GCP)

4. **Week 4:** Real-world application
   - Pick a bug bounty program
   - Perform complete infrastructure mapping
   - Document and share your methodology

---

**Remember:** This technique requires patience and curiosity. You're not just running commands—you're thinking like an attacker, understanding infrastructure, and connecting the dots!

Good luck, and hack responsibly! 🚀
