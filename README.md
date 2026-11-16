 # 🛡️Autonomous AI-SOAR Pipeline: Wazuh SIEM + Gemini AI + Telegram Active Response

An end-to-end **Security Orchestration, Automation, and Response (SOAR)** pipeline that integrates **Wazuh SIEM**, **Google Gemini AI (`gemini-3.5-flash-lite`)**, and **Telegram Bot API** to automate Level 1 SOC alert triage and execute **Human-in-the-Loop (HITL)** endpoint containment via Wazuh Active Response.

---

## 🏗️ Visual System Architecture

```text
+-----------------------------------------------------------------------------------+

| 1. ATTACK & DETECTION |
| |
| [Attacker / Hydra Brute-Force] --(SSH Flood)--> [Target Endpoint: Kali Linux] |
| | (Wazuh Agent ID: 001) |
| v |
| [Wazuh Manager (Ubuntu 24.04)] |
| Correlates Logs -> Rule 5763 |
| (Severity Level 10: Brute Force) |
+-----------------------------------------------------------------------------------+
 |
 | 2. Alert Forwarding
 | (Level >= 7 JSON)
 v
+-----------------------------------------------------------------------------------+

| 3. AI TRIAGE & ORCHESTRATION |
| |
| [Wazuh Integrator: custom-ai-soar] --(HTTP POST)--> [Flask Webhook :5001] |
| | |
| Filters Local/Invalid Events |
| | |
| v |
| [Google Gemini API] |
| Model: gemini-3.5-flash-lite |
| - MITRE ATT&CK Mapping (T1110) |
| - Risk & Confidence Scoring |
| - Containment Recommendation |
+-----------------------------------------------------------------------------------+
 |
 | 4. Human-in-the-Loop
 v
+-----------------------------------------------------------------------------------+

| 5. HUMAN APPROVAL & ACTIVE RESPONSE |
| |
| [SOC Analyst Telegram Bot] <--- Delivers AI Threat Brief & Inline Buttons |
| | |
| Analyst Taps [ Block IP] |
| | |
| v |
| [Flask Callback Listener] --(REST API :55000)--> [Wazuh Manager API] |
| | |
| Triggers `firewall-drop` Command |
| | |
| v |
| [Target Endpoint: Kali Linux (001)] |
| Attacker IP Blocked at Host Firewall |
+-----------------------------------------------------------------------------------+
```

---

## 🎯 Project Overview & Business Value

Security Operations Centers (SOCs) suffer from severe alert fatigue—analysts spend valuable time manually parsing raw JSON logs, looking up MITRE ATT&CK techniques, and logging into firewalls to block malicious IPs.

This project solves that bottleneck by building a lightweight, self-hosted **AI-SOAR** engine:
1. **Automated Ingestion:** Wazuh Manager detects high-severity threats (`Level >= 7`) and forwards structured alert JSON to a local Python Flask service.
2. **AI-Powered Tier-1 Triage:** **Google Gemini (`gemini-3.5-flash-lite`)** analyzes the raw telemetry in real time, mapping the behavior to MITRE ATT&CK, assessing confidence/severity, and generating plain-English remediation steps.
3. **Mobile Human-in-the-Loop (HITL):** Instead of risking automated false-positive lockouts, the system pushes the AI triage brief to a **Telegram SOC Bot** with interactive **` Block IP`** and **` Ignore`** buttons.
4. **One-Click Containment:** Upon analyst approval, the Flask responder authenticates against the **Wazuh REST API (`:55000`)** and executes Wazuh's native **`firewall-drop`** Active Response directly on the compromised agent.

---

## 🛠️ Tech Stack

* **SIEM & EDR:** Wazuh Manager & Dashboard v4.14 (Ubuntu 24.04 LTS)
* **Monitored Endpoint:** Kali Linux Agent (`ID: 001`)
* **AI Triage Engine:** Google GenAI SDK (`google-genai` / `gemini-3.5-flash-lite`)
* **SOAR Orchestrator:** Python 3, Flask, `requests`, `systemd`
* **ChatOps & Approval UI:** Telegram Bot API (Inline Keyboards & Long-Polling Listener)
* **Attack Simulation:** THC-Hydra (`hydra`) & OpenSSH

---

## ⚙️Working Wazuh Integration Configuration

To forward high-severity alerts (`Level >= 7`) from Wazuh Manager to the local Flask AI-SOAR webhook, the following `<integration>` block was configured inside `/var/ossec/etc/ossec.conf`:

```xml
<!-- AI-SOAR Integration -->
<integration>
 <name>custom-ai-soar</name>
 <hook_url>http://localhost:5001/analyze</hook_url>
 <level>7</level>
 <alert_format>json</alert_format>
</integration>
```

### Integrator Script Permissions (`/var/ossec/integrations/custom-ai-soar`)
Wazuh requires custom integration scripts to be owned by `root:wazuh` with `750` permissions:
```bash
sudo chown root:wazuh /var/ossec/integrations/custom-ai-soar
sudo chmod 750 /var/ossec/integrations/custom-ai-soar
```

---

## 🚀Persistent Production Deployment (`systemd`)

To ensure the webhook server and Telegram button listener run continuously in the background and survive system reboots, the application was deployed as a `systemd` service (`/etc/systemd/system/wazuh-ai-soar.service`):

```ini
[Unit]
Description=Wazuh AI-SOAR Webhook & Telegram Responder
After=network.target wazuh-manager.service

[Service]
Type=simple
User=YOUR_LINUX_USER
WorkingDirectory=/home/YOUR_LINUX_USER/wazuh-ai-soar
ExecStart=/home/YOUR_LINUX_USER/wazuh-ai-soar/venv/bin/python3 /home/YOUR_LINUX_USER/wazuh-ai-soar/app.py
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

---

##🔐  Environment & Secrets Configuration

All sensitive credentials are redacted for security. To replicate this setup, configure the following variables in `app.py` (or via environment variables):

```python
GEMINI_API_KEY = "YOUR_GOOGLE_AI_STUDIO_API_KEY"
TELEGRAM_TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"
CHAT_ID = "YOUR_TELEGRAM_CHAT_ID"

WAZUH_API_URL = "https://localhost:55000"
WAZUH_API_USER = "wazuh-wui"
WAZUH_API_PASSWORD = "YOUR_WAZUH_API_PASSWORD"
```

---

## 🧪 Live Attack Simulation & Verification

1. **Simulated Attack:** Launched a multi-threaded SSH brute-force attack using `hydra` against the monitored Kali Linux endpoint (`Agent 001`):
 ```bash
 hydra -l target_user -P wordlist.txt -t 4 -f ssh://<TARGET_AGENT_IP>
 ```
2. **Detection:** Wazuh correlated the rapid authentication failures into **Rule `5763` / `5712` (Level 10 — SSHD brute force trying to get access to the system)**.
3. **AI Triage & Notification:** Within seconds, the Telegram bot delivered the Gemini analysis identifying **MITRE ATT&CK T1110 (Brute Force)** and recommending immediate IP containment.
4. **Active Response Execution:** Clicking **` Block IP`** in Telegram triggered the Wazuh API (`PUT /active-response`), executing `firewall-drop` on Agent `001` and updating the Telegram alert status to **` CONTAINED`**.

---

## 🔧Key Troubleshooting & Engineering Lessons Learned

Building and stabilizing this pipeline in a multi-VM environment required solving several real-world integration issues:

* **OpenSSH Log Format & Correlation Thresholds:** Modern OpenSSH drops invalid usernames early (`drop connection #0 ... penalty: attempted authentication by invalid user`), triggering only Level 5 alerts (`Rule 5710`) without populating `srcip` for correlation. Testing against a valid system username generated standard `Failed password` telemetry, enabling Wazuh's decoder to extract `srcip` and escalate to Level 10 (`Rule 5763`).




