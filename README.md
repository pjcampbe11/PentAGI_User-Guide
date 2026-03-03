# Executive Summary  
PentAGI (Penetration Testing Artificial General Intelligence) is a self-hosted, AI-driven penetration testing platform. It runs in Docker containers and coordinates specialized AI agents to perform reconnaissance, vulnerability scanning, exploitation, and reporting using a built-in suite of ~20+ security tools (e.g. *nmap*, *Metasploit*, *Sqlmap*, etc.). This report covers comprehensive deployment methods (from source and via Docker), supported attack modules (with methodology, inputs, payloads, and caveats), integration into a HackerOne bug bounty workflow (automated recon and report generation with AI), and operational best practices (security, legal, rate limits, logging, etc.). We compare source vs Docker setups, provide configuration examples and AI prompting templates for crafting vulnerability reports, and include checklists, tables, and mermaid diagrams for workflows and timelines. All guidance is grounded in the PentAGI documentation and HackerOne policy sources.  

## 1. Deployment Setup  

### 1.1 Prerequisites  
- **Hardware/Software:** A host with Docker Engine (v24+) and Docker Compose (v2+) installed. Minimum 2 vCPU, 4 GB RAM, ~20 GB free disk. Internet access is needed to download images and updates.  
- **Operating System:** Linux is primary (x86_64 or arm64), though installer versions exist for Windows/macOS. Assuming Linux unless otherwise noted; Docker must run with sufficient privileges (root or Docker group).  

### 1.2 Installation Paths  

#### (A) Installer (Recommended)  
PentAGI provides a guided interactive installer (shell UI) to configure environment variables and launch via Docker Compose. Example (Linux): 
```
mkdir -p pentagi && cd pentagi
wget -O installer.zip https://pentagi.com/downloads/linux/amd64/installer-latest.zip
unzip installer.zip
sudo ./installer   # runs checks, configures .env, and launches
```  
The installer checks Docker, generates a `.env` (from `.env.example`), configures LLM/provider keys and search engines, sets up SSL, then `docker-compose up`. Use root or Docker-group permissions (e.g. `sudo` or add user to `docker` group)】.  

#### (B) Manual (from Source)  
1. **Clone or workspace**:  
   ```
   mkdir pentagi && cd pentagi
   git clone https://github.com/vxcontrol/pentagi.git .
   ```  
2. **Environment file**: Copy the example and edit:
   ```
   curl -o .env https://raw.githubusercontent.com/vxcontrol/pentagi/master/.env.example
   cp .env .env.bak && vim .env
   ```  
   At minimum set an LLM provider key (`OPEN_AI_KEY` or `ANTHROPIC_API_KEY`, etc.), and `PENTAGI_POSTGRES_USER/PASSWORD`. Configure any optional search or monitoring keys (Google API, Tavily, etc.)【9†L941-L950】. For secure setup, change salts (`COOKIE_SIGNING_SALT`), set `PUBLIC_URL`, and provide SSL certs via `SERVER_SSL_CRT/SERVER_SSL_KEY` if exposing public UI.  
3. **Tool configurations**: Touch or create provider YAMLs if using Ollama or custom LLMs (examples in `examples/configs/`).  
4. **Docker Compose**: Download or use `docker-compose.yml` from the repo:
   ```
   curl -O https://raw.githubusercontent.com/vxcontrol/pentagi/master/docker-compose.yml
   ```  
   Ensure the Compose file uses correct image tags (by default `vxcontrol/pentagi:latest`). If building locally, modify `image` to `local/pentagi:latest` and run `docker compose build`, or use build profiles.  
5. **Start services**:  
   ```
   docker compose up -d
   ```  
   By default, the web UI is on `https://localhost:8443` (login: `admin / admin`). If running in production, consider enabling monitoring/analytics stacks via additional Compose files (`docker-compose-observability.yml`, `docker-compose-langfuse.yml`, `docker-compose-graphiti.yml`) for Grafana/Prometheus, Langfuse, and knowledge graph.  

**Common Errors/Troubleshooting:**  
- *Docker Socket Access:* Ensure PentAGI container has access to Docker socket (default Compose uses `volumes: /var/run/docker.sock:/var/run/docker.sock`). If using Docker via TCP, remove root privileges and run as user. Error with socket indicates permission issues.  
- *PostgreSQL Connection:* The system uses PostgreSQL + pgvector. Check `PENTAGI_POSTGRES_USER/PASSWORD` match Compose, and that the pgvector extension is present. Connection failures may appear if DB not ready – use `docker compose logs pentagi` to diagnose.  
- *Missing API Keys:* If LLM or search keys are missing/invalid, agents cannot query. The UI/CLI will error on missing providers – recheck `.env`.  
- *Embedding Issues:* If you change embedding provider, run the `etester` utility to reindex memory (see below).  
- *Port Conflicts:* PentAGI UI defaults to 8443; ensure that port is free or set `SERVER_PORT` accordingly. If using separate networks (two-node mode), ensure worker ranges are reachable.  
   
> **Tip:** Use `docker compose logs -f pentagi` and other service logs (Postgres, etc.) to troubleshoot startup issues. The `ftester` and `etester` tools (in the backend) can test function calls and embedding connectivity
docker exec -it pentagi /opt/pentagi/bin/etester test
```  
and check errors (e.g. database errors). 

### 1.3 Docker vs Source: Pros & Cons  

| Aspect                   | Running via Docker (Compose)                                          | Running from Source                                           |
|--------------------------|----------------------------------------------------------------------|----------------------------------------------------------------|
| **Setup Speed**          | Very quick: pull/prebuilt images and run (`docker-compose up -d`)【27†L114-L123】.| Slower: requires cloning repo and building, plus dependencies. |
| **Isolation**            | All components containerized (recommended for sandboxing)【6†L354-L362】. | Can run with containers too but more manual management.        |
| **Customization**        | Limited to provided images; custom tools require custom images.        | Full control: can modify code or images (e.g. change default pentest image). |
| **Performance**          | Stable/performance tuned images; easy scaling.                        | Might require manual tuning (e.g. ensure correct Go build).   |
| **Updates**              | Pull latest image from Docker Hub; easy upgrades.                    | Must pull latest code and rebuild; more manual.               |
| **Security**             | Runs as root by default for Docker socket access (less secure)【10†L43-L47】; easier sandbox. | Can run Docker in rootless mode or on a separate host for security. |
| **Debugging**            | Easier isolation; logs via `docker logs`.                             | Debug with local logs, but tool binaries are accessible.      |

## 2. Attack Modules & Capabilities  

PentAGI orchestrates AI agents that use integrated open-source pentest tools. Although the system auto-plans attacks, we categorize its capabilities by attack type, underlying tool, inputs, typical payloads, detection/evasion strategies, and limitations. (The agent *decides* which attacks to launch based on context, but understanding each class is crucial.)

- **Reconnaissance (Information Gathering):**  
  - *Passive Recon:* PentAGI uses web intelligence (built-in headless browser) and external search APIs (Google, DuckDuckGo, Tavily, etc.)【6†L362-L370】. The agent prompts the browser to crawl target domains or use search engines to gather domain names, technologies, open source intelligence (OSINT), metadata, and employee info. Detection: very stealthy (like normal browsing). Limitations: relies on what’s public; requires queries via `.env` (enable `DUCKDUCKGO_ENABLED`, etc.)【9†L941-L950】.  
  - *Active Discovery:* DNS/subdomain enumeration using tools like `subfinder`, `amass` (implied via Kali integration). Input: target domain. Output: list of subdomains/IPs. Payloads: DNS queries, brute-forcing hostnames. Evasion: none needed (passive DNS is low-key). Limitations: misses non-public subdomains.  
  - *Host/Port Scanning:* Uses **Nmap** (e.g. SYN scan `-sS`, UDP scan) to find live hosts and open ports【48†L872-L879】. Inputs: target IP/range. Payloads: standard Nmap probes. Detection: SYN scans can trigger IDS unless throttled. PentAGI may use timing adjustments (e.g. `-T3`). Limitations: slow on large ranges; might miss filtered ports.

- **Service Enumeration & Version Detection:**  
  After ports are found, PentAGI runs Nmap with service/OS detection (`-sV -O`), banner grabbing, and specialized tools (e.g. `snmpwalk`, `enum4linux`, `hydra` for SMB/SSH) as needed. Inputs: port numbers from scan. Payloads: protocol probes (SSH banner, SNMP OIDs). Evasion: VLAN sandboxing; can adjust scan speed. Limitation: blocked by firewalls/IDS if aggressive; reliance on tool DB for OS detection.

- **Web Application Scanning:**  
  - *Common Vuln Scanning:* **Nikto** (web server scan) and **Nuclei** (template-based scanning) check for default scripts, outdated software, common vulnerabilities【48†L872-L879】. Input: target URL. Payloads: HTTP requests with known patterns. Detection: moderate (Nikto hits many URLs), but may be rate-limited.  
  - *Directory/Resource Bruteforce:* **Gobuster/Dirb** with wordlists to find hidden directories/files. Input: base URL and list. Payload: HTTP GET to guessed paths. Detection: can be noisy; use slow bruteforce or moderate concurrency.  
  - *Parameters Fuzzing:* Nuclei or custom payloads to test injection (SQL, XSS, LFI, SSRF). For example, injecting `' OR 1=1--` into query params and analyzing responses. Payloads: common SQL meta, `<script>` tags. Detection: some IDS may catch typical attack strings; could use URL encoding or obfuscation. Limitations: may produce many false positives (PentAGI uses agent parsing to filter).  
  - *SQL Injection:* **Sqlmap** is specifically integrated for SQLi. When PentAGI suspects an injectable parameter (based on anomaly scanning or known patterns), it can invoke `sqlmap` automatically【48†L872-L879】. Inputs: vulnerable URL/param. Payloads: series of SQL injection payloads. Detection: high risk (noise) – agent might throttle or fallback to OOB testing (e.g. Burp Collaborator). Limitations: false flags if web app has filtering; requires response analysis.  
  - *Cross-Site Scripting (XSS):* The system may attempt XSS by injecting script payloads (`<script>alert(1)</script>`). It can use its internal browser to test if scripts execute. Detection: XSS payloads are classic, but agent can use novel variations or blind tests. Limitations: modern CSPs may block; reading outcomes requires interactive checks.  
  - *XML External Entities (XXE), CSRF, SSRF:* The AI might craft payloads for these if code patterns (XML parsers, URL fetchers) are found. For SSRF/XXE, PentAGI can use its own server as an OOB listener to catch callbacks (enabled by dedicated OOB port ranges【9†L891-L900】). These require the target server to trigger out-of-band DNS/HTTP to PentAGI.  
  - *Authentication Brute Force:* Using **Hydra** or built-in tools for HTTP auth, FTP, SSH, SMTP, etc. Agent input includes login pages; it will load default or custom username/password lists. Payload: typical creds. Detection: login lockouts or rate limits; PentAGI must respect rate-limiting.  
  - *Social Engineering / Phishing:* PentAGI can simulate Slack/Gmail enumeration (implied by Slackhound-style integration, though not explicitly documented) and generate phishing emails for testing (the frontend has an Editor). Payload: crafted messages. Detection: high risk (should only do in controlled CTF/engagement with permission). Limitations: legal issues – see Ethical Considerations.  

- **Exploitation:**  
  - *Metasploit Modules:* PentAGI can launch Metasploit exploits for known vulns (CVE-driven). Input: service info and version. Payloads: exploit code, reverse shell payloads. Detection: likely high (exploitation is noisy), but important for POC. Limitations: needs correct versions; trial-and-error.  
  - *Impacket Tools:* For network attacks (SMB relay, LLMNR poisoning, NTLM capture). Inputs: network share access. Payloads: relay attacks. Detection: can be stealthier than metasploit; still needs network presence.  
  - *Privilege Escalation:* If it gains an account, PentAGI can use **Evil-WinRM** (WinRM brute/exec) or local privesc scripts. Input: valid credentials. Payloads: PowerShell or binaries. Detection: attempts often logged; limited use only if permitted.  
  - *Client-Side Attacks:* The AI can simulate visiting malicious URLs in its browser (scraper) to trigger browser exploits or cookie theft. Payload: drive-by XSS or CORS trickery. Limitations: in a sandbox, not real user; more theoretical for report.  

Each attack’s **output** feeds back to the AI’s memory and knowledge graph (via Graphiti), allowing dynamic planning.  Because PentAGI uses actual tools, its *methodology* mirrors manual pentesting, but orchestrated by AI. For instance, an agent may plan: 1) Scan network → 2) enumerate hosts → 3) scan web servers → 4) attempt web vulns (SQLi/XSS) → 5) if creds found, try SSH/SMB→ 6) report. Detection/Evasion: PentAGI allows adjusting scan speeds; it performs a “two-node” setup where attack containers are isolated by network segmentation【9†L892-L901】. Limitations include dependency on API keys for intelligence and the possibility of false positives (AI may misinterpret tool output) – the operator should verify findings manually.

## 3. HackerOne Bug Bounty Workflow  

We outline an end-to-end flow for using PentAGI in a HackerOne (H1) bounty engagement, from program targeting through automated report submission. This assumes the program permits automated scanning (check scope first). We integrate PentAGI’s findings with AI-assisted report writing and H1’s API for submission.

```
flowchart TD
    A[Identify H1 Program] --> B[Review Scope & Authorization]
    B --> C[Configure PentAGI Target]
    C --> D[Automated Recon & Scanning]
    D --> E[Analyze Findings & Confirm]
    E --> F[Generate Report Draft (AI)]
    F --> G[Populate H1 Report Template]
    G --> H[Review & Submit via API]
    H --> I[Acknowledge H1 Response]
```

1. **Pre-Engagement / Scope Check:** Ensure the target is authorized under H1 program rules (public program listing on HackerOne, in-scope domains). Abide by program policies (e.g. allowed testing times, depth). Document scope and get written permission if not clear.

2. **Automated Recon & Scanning:**  
   - **Recon:** Feed the target (domain/IP, endpoints) into PentAGI. Use the web UI or CLI (ftester) to start a pentest *flow* (e.g., “Scan `target.com` for web vulnerabilities”). The AI agents will automatically run reconnaissance as above, using search/web intelligence and tool-based scans.  
   - **Customization:** Optionally supply hints (e.g. “Focus on web endpoints only”) via the interface. Configure rate-limits in PentAGI to avoid overwhelming target (use `.env`: e.g. `PENTAGI_REQUEST_TIMEOUT`, or adjust Nmap timing).  
   - **Collection:** PentAGI will produce logs and a knowledge graph of discovered info. It may generate draft vulnerability findings internally. Monitor the Grafana UI or Langfuse (if enabled) for progress.

3. **Review & Verification:** The pentester reviews automated findings. False positives can occur (especially with fuzzed inputs); verify by reproducing high-risk findings manually if needed. Confirm exploitability and gather concrete evidence (screenshots, request/response). The knowledge graph can help correlate which hosts had similar issues.

4. **Report Generation (AI-assisted):** Using PentAGI’s “Detailed Reporting” feature or external AI (e.g. ChatGPT), generate a vulnerability report. We propose an AI prompt template. For each confirmed issue:  
   - **Prompt Example:**  
     ```
     "Write a HackerOne-style vulnerability report. 
     Issue: SQL Injection in login form of example.com. 
     Impact: Attacker can bypass login. 
     Provide sections: Title, Summary, Steps to Reproduce, Impact, Remediation, Proof-of-Concept, Severity." 
     ```  
   - This produces a draft with the required sections (as recommended by H1). Format using Markdown.  

5. **Filling H1 Template:** Customize the HackerOne submission form (via Program Settings) to include a template. Typical fields to fill (based on H1 guidelines):  
   - *Summary/Title:* A concise statement of the flaw (e.g. “SQLi in example.com login allows privilege escalation”).  
   - *Steps to Reproduce:* Numbered steps (login to, insert payload, observe behavior).  
   - *Expected vs Actual:* Briefly note expected secure behavior vs observed.  
   - *Impact:* Describe severity (e.g. data exposure, account takeover).  
   - *Remediation:* Suggest fixes (e.g. parameterized queries, input sanitization).  
   - *Proof-of-Concept:* Include payloads, screenshots, or code demonstrating the exploit (formatted in markdown code blocks).  
   - *Severity:* Map to H1 categories (e.g. Low/Medium/High/Critical) based on impact. Use H1’s rating guide if available.  
   
   Sample H1 report template sections (as Markdown):  
   ```
   # Summary
   Insert a brief, clear description of the issue.  
   
   ## Steps to Reproduce
   1. ...  
   2. ...  
   3. ...  

   ## Expected Behavior
   What should happen (sanitization, rejection).  

   ## Actual Behavior
   What happens instead (e.g. payload executes).  

   ## Impact
   Describe potential consequences (data leak, account access).  

   ## Remediation
   Suggest how to fix (parameterize inputs, add validation).  

   ## Proof of Concept
   *Here include payloads, example requests/responses, screenshots.*  

   **Severity:** [e.g. High/Critical/Medium/Low]
   ```

6. **Automated Submission:** Use the HackerOne API to submit the report programmatically. The H1 API (`v1`) requires authentication (an API token). Example (via `curl`):  
   ```
   curl -X POST "https://api.hackerone.com/v1/reports" \
     -H "Authorization: Bearer YOUR_API_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
           "report": {
             "reporter_email": "[email protected]",
             "submission": {
               "program_handle": "target_program_handle",
               "title": "SQL Injection in login form of example.com",
               "description": "## Steps... <full Markdown report here>"
             }
           }
         }'
   ```  
   (The JSON must include the formatted Markdown description). Check H1 docs for exact fields and use a test submission first (HackerOne often provides a test endpoint or dummy program).  

7. **Follow-Up & Remediation Tracking:** Once submitted, track your report in H1 (ensure triage compliance). Maintain logs of tools used (PentAGI logs are timestamped and can serve as evidence of methodology). Collaborate via H1 comments as needed (PentAGI’s report can be used as initial content).

```
gantt
title Setup & Deployment Timeline
dateFormat  YYYY-MM-DD
section Preparation
Install Docker & Compose   :done, 2026-02-20, 1d
Clone Repo & Config .env   :done, 2026-02-21, 1d
Setup LLM & Search Keys    :done, 2026-02-22, 0.5d
Download Docker Compose    :done, 2026-02-22, 0.5d
section Deployment
Start Services (docker-compose up) :active, 2026-02-23, 1d
Validate UI Access         :done, 2026-02-24, 0.5d
Run Test Scan (vulnbank)   :done, 2026-02-24, 0.5d
section Integration
Create H1 Report Template  :2026-02-25, 0.5d
Generate Report with AI    :2026-02-25, 0.5d
Submit via API            :2026-02-26, 0.5d
```

## 4. Best Practices & Operational Security  

- **Authorization & Ethics:** Only target systems explicitly in-scope (per HackerOne program policy). PentAGI’s EULA and site emphasize lawful use and permission【58†L58-L64】【54†L5-L8】. Document written authorization. Do *not* scan external networks indiscriminately (violates laws like CFAA). Use PentAGI only for agreed-upon assets.  
- **Rate Limiting:** Respect target’s availability. Configure scan rates (PentAGI’s Nmap parameters or global settings like `WEBSCRAPER_REQUEST_LIMIT`) to avoid DoS. Use progressive scans (start slow, then deeper if safe).  
- **Isolation:** Run PentAGI on a dedicated host or VM. The recommended two-node setup isolates “attacker” containers on a separate machine with limited network access【9†L892-L901】. Use firewall rules: PentAGI’s container might use Docker’s `NET_ADMIN`/`NET_RAW` caps (for scanning) but restrict outbound/inbound as needed.  
- **Credentials Management:** Store API keys (LLM, search, DB) securely (not in public repo). Use least-privilege for PostgreSQL and avoid exposing them.  
- **Logging & Monitoring:** Enable PentAGI’s observability (Grafana, Jaeger, etc. stacks). Collect logs of all agent actions. These logs provide audit trails and help detect anomalies (e.g. unintended scans). Use the knowledge graph logs for retrospective analysis.  
- **False Positives Handling:** AI may generate false alerts. Cross-verify critical findings manually or with multiple tools. Always include proof-of-concept evidence in reports. If a PentAGI finding seems dubious, try the corresponding tool directly (e.g., rerun Nmap or sqlmap manually).  
- **Collateral Damage Avoidance:** Avoid attacking third parties or shared infrastructure (DNS servers, multi-tenant). Do not use PentAGI’s web scraper to scrape disallowed content. For web apps, avoid leaving malicious payloads behind (clean up test accounts if created).  
- **Legal Constraints:** Understand local laws. For bug bounty scopes, abide by disclosure policies. PentAGI’s report generation should not include confidential data (e.g. any test data encountered).  
- **Incident Response:** If PentAGI triggers unexpected behavior on target (e.g. takes down a service), be prepared to abort testing. The reporting stack (Grafana alerts) should notify operators of pentest failures or timeouts.  
- **Access Controls:** Use firewall and network segmentation so only PentAGI containers and needed services communicate. Limit SSH to PentAGI host. Use strong passwords for PentAGI UI and PostgreSQL, rotate them.  
- **Updates & Patching:** Keep PentAGI images up to date (pull latest tags). Regularly update the pentest tools (via rebuilding Kali image or pulling new PentAGI updates) to include latest exploits and CVEs.

## 5. Security & Misuse Safeguards  

- **Privileged Actions:** PentAGI runs OS commands inside containers. An attacker compromising PentAGI could run arbitrary Docker commands. Mitigate by running PentAGI on dedicated, isolated infrastructure; avoid installing additional untrusted tools.  
- **Prompt Injection & LLM Safety:** PentAGI uses LLMs to reason. Guard against malicious prompts (e.g. a fake vulnerability could be suggested by an attacker to provoke unsafe tool use). The system should sanitize user input and have an internal “verification” agent to question impossible actions.  
- **API Abuse:** Rate-limit PentAGI’s own web API to prevent external misuse. Use authentication (API keys/HTTPS) to restrict who can trigger scans.  
- **False Positive Detection:** Build checks in the workflow: if an AI-generated finding lacks standard POC (error logs, server response), mark it as low confidence. Encourage multi-agent validation: e.g., one agent suggests an attack, another should try verifying it.  
- **Ethical LLM Constraints:** If using external LLMs, ensure outputs are not logging sensitive info (PentAGI’s memory stores embeddings — protect that DB). Do not feed actual sensitive data into the LLM engine if using external API.  
- **Logging Privacy:** The memory store logs command outputs and context. Secure the PostgreSQL (with strong creds and SSL) to prevent data leakage.  
- **Safeguarding API Keys:** Limit scope of API keys (e.g. Google keys with domain restrictions). Do not expose `.env` publicly.

## 6. Summary Tables  

**Attack Module Support (Examples)**: The table below summarizes major modules and their purposes (see section 2 for details).

| Attack Category      | Tools/Methods             | Inputs                        | Example Payloads / Checks                    | Notes / Limitations |
|----------------------|---------------------------|-------------------------------|----------------------------------------------|---------------------|
| Recon (Passive)      | Web scraper, Tavily, Duck | Target domain/name            | Open-source info, subdomain lists           | Relies on public data; LLM accuracy matters |
| Recon (Active)       | subfinder, amass, whois   | Target domain                 | DNS queries, brute subdomains               | Might miss hidden hosts |
| Port Scanning        | nmap (TCP/UDP), masscan   | IP range, CIDR                | SYN packets, ICMP echo (ping)               | IDS/IPS can detect; use cautious rates |
| Service Enumeration  | nmap -sV, SMB tools       | Open port list                | Protocol probes, banner grabs              | May require manual creds (e.g. SNMP) |
| Web Scanning         | nikto, nuclei, httpx      | URL (website)                 | CVE signatures, wordlist fuzzing            | Noisy, may hit false positives |
| SQL Injection        | sqlmap, custom payloads   | URL/param suspected injectable| `' OR 1=1--`, time-based payloads            | Detected by WAF; false flags possible |
| XSS / SSRF           | Custom injection payloads | Website forms/params          | `<script>` tags, `<img src=http://pentagi>`| Often needs human verification |
| Auth Bruteforce      | Hydra, medusa            | SSH/FTP/HTTP login pages      | List of common creds (e.g. admin/admin)     | Account lockouts possible |
| Exploitation         | Metasploit, Impacket     | Known vulnerable service info | Metasploit payloads, SMB relays            | No success if target patched |
| Post-Exploitation    | Evil-WinRM, BloodHound   | Credentials obtained          | Lateral movement tools (PS Remoting)       | Only inside network; scope limits |
| Reporting            | AI Agent Analysis        | Tool outputs, logs            | Structured vulnerability findings          | Can hallucinate; operator review needed |

**Source vs Docker Deployment:** (See section 1.3 above for qualitative pros/cons).  

**Pre-Engagement Checklist:**  
- Confirm target in-scope (per H1 program).  
- Read program rules (e.g. no social media, no active DoS).  
- Ensure all tools/shells ready (Docker, .env keys, etc.).  
- Configure proxies or VPNs if needed (PentAGI supports `PROXY_URL`).  
- Backup `.env` and security configs.

**Submission Checklist:** (Aligns with H1 Quality reports):  
- Clear Title summarizing the flaw.  
- Repro steps (numbered) with all required details.  
- Expected vs Actual behavior.  
- Impact assessment (why it matters).  
- Proof-of-concept (payloads, code snippets, screenshots).  
- Remediation suggestion (if known).  
- Severity rating.  
- **Before submission:** check it’s not duped, fix formatting (Markdown, code blocks).

## 7. Examples  

- **.env Snippet:** (set your keys in PentAGI’s `.env`)
  ```
  # Required LLM provider
  OPEN_AI_KEY=sk-XXXXXXXXXXXXXXXX
  # Optional search engines
  DUCKDUCKGO_ENABLED=true
  GOOGLE_API_KEY=AAAAA
  GOOGLE_CX_KEY=BBBBB
  # Postgres creds
  PENTAGI_POSTGRES_USER=pentauser
  PENTAGI_POSTGRES_PASSWORD=strongpass123
  ```

- **Docker Commands:**  
  ```
  # Build locally (source mode)
  docker compose build pentagi
  docker compose up -d
  # Or pull latest image
  docker pull vxcontrol/pentagi:latest
  docker compose up -d
  ```

- **PentAGI Function Tester:** Example to run a terminal command via ftester:  
  ```
  cd backend
  go run cmd/ftester/main.go terminal -command "nmap -sS -O example.com" -message "Perform TCP scan"
  ```

- **AI Prompt Template (for GPT, etc.):**  
  ```
  Title: SQL Injection in Login Page of example.com
  Summary: The 'username' parameter in the login form does not sanitize input, allowing SQL Injection.
  Steps: 
  1. Go to https://example.com/login.
  2. Enter `' OR 1=1--` as username.
  3. Observe that login is bypassed.
  Impact: Successful exploitation allows an attacker to bypass authentication and gain account access.
  Remediation: Use prepared statements or input validation.
  Severity: High (Authentication Bypass).
  Proof-of-Concept: [show example POST request and DB response].
  ```

- **Sample AI Output (H1-style Report):**  
  *This illustrates how an LLM might format the above:*
  ```
  **Summary:** The login form on example.com is vulnerable to SQL Injection, allowing authentication bypass.

  **Steps to Reproduce:** 
  1. Navigate to https://example.com/login.
  2. In the "Username" field, enter `' OR 1=1--` and any password.
  3. Click "Submit" and observe that you are logged in without valid credentials.

  **Expected Behavior:** The application should sanitize inputs and reject the malicious payload, returning an error.

  **Actual Behavior:** The payload alters the SQL query (`SELECT * FROM users WHERE username='' OR 1=1-- AND password='...'`), causing a true condition and logging in the attacker.

  **Impact:** An attacker can bypass authentication entirely, gaining unauthorized access to user accounts and potentially administrative areas. This is a critical issue as it compromises all user data.

  **Remediation:** Implement parameterized queries or escape user inputs. Ensure all database queries use prepared statements.

  **Proof of Concept:** Example HTTP request demonstrating the SQLi:
  ```http
  POST /login HTTP/1.1
  Host: example.com
  Content-Type: application/x-www-form-urlencoded

  username=' OR 1=1--&password=test
  ```
  The response shows a successful login despite invalid credentials.

  **Severity:** High
  ```  

*Sources:* PentAGI setup and feature details; HackerOne report guidance. These outputs demonstrate how to align PentAGI findings with H1’s expected report format for efficient submission.
