# 15 - Social Engineering Penetration Testing

> **CPENT Module 04 | Exam Reference**
>
> **Author:** Zeeshan  
> **GitHub:** https://github.com/mzeeshanzafar28  
> **LinkedIn:** https://www.linkedin.com/in/mzeeshanzafar28

---

## Table of Contents

1. [What Is Social Engineering?](#what-is-social-engineering)
2. [CPENT Module Scope](#cpent-module-scope)
3. [The Human Attack Surface](#the-human-attack-surface)
4. [Authorization & Rules of Engagement](#authorization--rules-of-engagement)
5. [Social Engineering Process](#social-engineering-process)
6. [Off-Site vs On-Site Testing](#off-site-vs-on-site-testing)
7. [Phishing](#phishing)
8. [Phone-Based Social Engineering](#phone-based-social-engineering)
9. [Social Engineering Toolkit (SET)](#social-engineering-toolkit-set)
10. [GoPhish Framework](#gophish-framework)
11. [AI-Assisted Social Engineering](#ai-assisted-social-engineering)
12. [Evidence Collection](#evidence-collection)
13. [Countermeasures](#countermeasures)
14. [Common Mistakes](#common-mistakes)

---

## What Is Social Engineering?

Social engineering is the **manipulation of people** into performing an action or disclosing information that benefits an attacker.

### Psychological Triggers Exploited

| Trigger | How It's Used |
|---------|---------------|
| **Authority** | Impersonating management, IT, or law enforcement |
| **Urgency** | "Your account will be disabled in 1 hour" |
| **Fear** | "Suspicious login detected — verify now" |
| **Curiosity** | "Confidential: Q3 layoff list (internal)" |
| **Greed** | "Congratulations! You've won a gift card" |
| **Reciprocity** | Offering help first, then asking for a favor |
| **Social Proof** | "Everyone in your department already did this" |
| **Familiarity** | Building rapport over time before making the ask |

### Attack Chain

```
Attacker → Human Decision → Action → Access
```

Unlike technical attacks (attacker → software → access), social engineering targets **human decision-making** as the control to bypass.

---

## CPENT Module Scope

EC-Council lists **Social Engineering Penetration Testing** as CPENT Module 04. The official outline covers:

- Social Engineering Penetration Testing Concepts
- Social Engineering Penetration Testing Process
- Off-Site Social Engineering Penetration Testing
- On-Site Social Engineering Penetration Testing
- Phishing
- Social Engineering Using Phone
- Social Engineering Using AI and ML
- Document Findings with Countermeasure Recommendations

**CPENT labs** primarily focus on the **Social-Engineer Toolkit (SET)** for credential-sniffing exercises.

---

## The Human Attack Surface

Targets within an organization include:

```
Employees          → Email, phone, physical entry
Contractors        → Often have weaker security training
Administrators     → Hold elevated privileges (high-value target)
Help-Desk Staff    → Trained to be helpful — exploitable
Executives         → Whaling targets, public info available
Vendors            → Third-party access, less vetted
Temporary Workers  → Short tenure, minimal security onboarding
```

Map the attack surface before selecting a test scenario:

```
Employee
   ├── Email (phishing, spear-phishing)
   ├── Phone (vishing, help desk impersonation)
   ├── VPN (credential harvesting)
   ├── Help Desk (password reset manipulation)
   └── Physical Office (tailgating, badge cloning)
```

---

## Authorization & Rules of Engagement

Social engineering directly affects **people**, so authorization must be explicit.

### Rules of Engagement Must Define

```
✓ Who may be targeted?
✓ Which departments?
✓ Which communication channels? (email, phone, SMS, in-person)
✓ Which dates/times?
✓ What pretexts are allowed?
✓ What information may be requested?
✓ What information may NEVER be requested?
✓ Can physical entry be tested?
✓ Can badges be tested?
✓ Can third parties be contacted?
✓ Can personal accounts be targeted?
```

### Stop Conditions

Define BEFORE testing:

```
- Physical confrontation
- Medical emergency
- Employee distress
- Real security incident triggered
- Law enforcement involvement
- Accidental access to sensitive production data
- Client requests immediate termination
```

> **If a real incident starts during a simulation: STOP → preserve evidence → notify authorized contact.**

---

## Social Engineering Process

```
1. Scope definition
2. Rules of engagement
3. Reconnaissance (OSINT on targets)
4. Target selection
5. Attack hypothesis
6. Scenario design (pretexts, payloads)
7. Infrastructure preparation (domains, email servers, landing pages)
8. Pre-test validation
9. Controlled execution
10. Evidence collection (timestamps, screenshots, logs)
11. Stop/abort when required
12. Analyze results
13. Risk assessment
14. Countermeasure recommendations
15. Retest (if authorized)
16. Report
```

---

## Off-Site vs On-Site Testing

### Off-Site (Remote)

| Technique | Description |
|-----------|-------------|
| **Phishing** | Mass email campaigns with malicious links/attachments |
| **Spear-phishing** | Targeted emails using OSINT-gathered personal details |
| **Whaling** | Targeting C-level executives |
| **Vishing** | Voice phishing — phone calls impersonating authority |
| **SMiShing** | SMS/text message phishing |
| **Pretext calling** | Calling help desk to reset passwords or gather info |
| **Fake job postings** | Harvesting credentials via fake career portals |

### On-Site (Physical)

| Technique | Description |
|-----------|-------------|
| **Tailgating** | Following an authorized person through a door |
| **Badge cloning** | Copying RFID/NFC badges |
| **USB drops** | Leaving infected USB drives in parking lots/break rooms |
| **Impersonation** | Posing as IT, vendor, or maintenance staff |
| **Dumpster diving** | Searching trash for sensitive documents |
| **Shoulder surfing** | Observing screens/keyboards in public areas |

---

## Phishing

### Phishing Infrastructure Setup

```bash
# Clone a target website for credential harvesting
setoolkit

# Or use GoPhish (more modern, full campaign management)
# Install: https://getgophish.com

# Set up a phishing domain (use typosquatted or lookalike domain)
# Configure email server (SMTP) for sending
# Set up landing page that captures credentials
```

### Phishing Email Elements

```
Subject: [URGENT] Password Expiration Notice
From: IT Support <it-support@c0mpany.com>    ← lookalike domain
Body:
  - Urgency trigger: "Your password expires in 2 hours"
  - Authority: "Per IT policy update..."
  - Call to action: "Click here to keep your current password"
  - Link: https://company-secure-portal.com/login  ← phishing landing page
```

### Credential Harvesting with SET

```bash
# Launch SET
sudo setoolkit

# Menu path:
# 1) Social-Engineering Attacks
# 2) Website Attack Vectors
# 3) Credential Harvester Attack Method
# 2) Site Cloner

# Set:
# IP address for POST back: YOUR_IP
# URL to clone: https://target-company.com/login

# Send phishing email with link to your cloned page
# Captured credentials appear in SET terminal output
```

---

## Phone-Based Social Engineering

### Help Desk Impersonation Script

```
1. Call help desk during lunch hours (less staff, rushed)
2. "Hi, this is [Name from OSINT] from [Department from OSINT].
   I'm traveling and locked out of my VPN. Can you reset my password?"
3. Provide verifiable details from OSINT (employee ID, manager name, recent projects)
4. If challenged: "Look, my manager [Name] is expecting the Q3 report by 3 PM.
   I really need this now."
```

### Key Principles

- **Pretext before calling** — know the target's org chart, recent events, internal lingo
- **Sound rushed but not panicked** — urgency, not desperation
- **Have answers ready** — employee ID, manager name, office location
- **Know when to abort** — if the target becomes suspicious, politely end the call

---

## Social Engineering Toolkit (SET)

SET is included in Kali Linux. Primary modules:

```bash
sudo setoolkit
```

| Menu Option | Purpose |
|-------------|---------|
| **1) Spear-Phishing Attack Vectors** | Craft and send phishing emails with payloads |
| **2) Website Attack Vectors** | Credential harvester, tabnabbing, web jacking |
| **3) Infectious Media Generator** | Generate malicious USB/CD payloads |
| **4) Create a Payload and Listener** | Generate reverse shell payloads + start listener |
| **5) Mass Mailer Attack** | Send phishing emails to multiple targets |
| **6) Arduino-Based Attack Vector** | Teensy USB HID attacks |
| **7) SMS Spoofing Attack Vector** | Spoof SMS messages |
| **8) Wireless Access Point Attack Vector** | Rogue AP for credential harvesting |
| **9) QRCode Generator Attack Vector** | Malicious QR codes |
| **10) Powershell Attack Vectors** | PowerShell-based payloads |

### Quick SET Credential Harvester

```bash
sudo setoolkit
# 1 → 2 → 3 → 2
# Set LHOST to your IP
# Enter URL to clone: https://target.login.page
# Send link to targets → credentials captured in real-time
```

---

## GoPhish Framework

GoPhish is a modern, full-featured phishing framework (more practical for CPENT than SET for mass campaigns).

```bash
# Download from https://getgophish.com
# Extract and run
./gophish

# Web UI at https://localhost:3333
# Default creds: admin / gophish (change immediately)
```

**Features:**
- Email campaign management
- Landing page builder with credential capture
- Email template editor
- Real-time campaign tracking (opens, clicks, credentials)
- Sending profile configuration (SMTP relays)

---

## AI-Assisted Social Engineering

CPENT now explicitly includes AI/ML in social engineering. Common uses:

| AI Application | Purpose |
|----------------|---------|
| **Deepfake voice** | Impersonate executives in vishing calls |
| **LLM-generated pretexts** | Create convincing, personalized phishing emails |
| **OSINT automation** | Scrape social media, build target profiles |
| **Chatbot impersonation** | AI chatbots that engage targets on messaging platforms |
| **Grammar/spelling correction** | Remove typos that make phishing obvious |

> **Exam note:** CPENT may test awareness of AI-assisted SE techniques rather than hands-on execution.

---

## Evidence Collection

Document everything during a social engineering engagement:

```
✓ Timestamps of every call/email/interaction
✓ Screenshots of phishing emails sent
✓ Record of credentials captured (redact in final report if real)
✓ Call recordings (if legally permitted)
✓ Physical access logs (time of entry, areas accessed)
✓ Photos of physical access points (badge readers, unlocked doors)
✓ Server logs showing connection timestamps
✓ Notes on employee responses and awareness levels
```

---

## Countermeasures

### Technical Controls

```
- Email filtering (SPF, DKIM, DMARC)
- URL rewriting and sandboxing
- Attachment scanning and blocking
- Multi-Factor Authentication (MFA) — defeats credential harvesting
- Caller ID verification systems
- Badge access logs and anomaly detection
```

### Process Controls

```
- Verification procedures for password resets (callback number, manager approval)
- Visitor escort policy
- Clean desk policy
- Secure document disposal (shredding)
- Vendor verification process
```

### Human Controls

```
- Regular security awareness training
- Simulated phishing campaigns with feedback
- Clear reporting channels ("Report Phishing" button)
- No-blame culture for reporting mistakes
- Tabletop exercises for social engineering scenarios
```

---

## Common Mistakes

| Mistake | Why It's Bad |
|---------|-------------|
| **No written authorization** | Illegal. Always get explicit, documented permission. |
| **Targeting everyone** | Unethical and impractical. Define specific targets. |
| **Collecting real passwords** | Never store actual employee credentials. Use redaction. |
| **Embarrassing employees** | Destroys trust. Focus on control weaknesses, not individuals. |
| **No stop conditions** | Can't abort safely if something goes wrong. |
| **Skipping OSINT** | Poor pretexts get caught immediately. Research first. |
| **Same pretext for all** | One-size-fits-all fails. Tailor to each target's context. |

---

> **Remember:** Social engineering tests the **human layer** of security. Success doesn't mean "people are stupid" — it means controls need improvement. Report constructively.
