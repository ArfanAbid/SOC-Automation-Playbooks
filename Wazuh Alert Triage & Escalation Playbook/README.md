![alt text](<Screenshot 2025-09-11 164018.png>)

# Wazuh Alert Triage & Escalation Playbook : Wazuh + Shuffle + TheHive + Discord/Gmail 🛡️

This stack integrates Wazuh (for threat detection and SIEM), Shuffle (for automation and orchestration), and DFIR-IRIS (for case management and investigation), with Discord/Gmail providing real-time notifications. Together, it enables automated alerting, enrichment, incident tracking, and collaborative response to security events.

This playbook demonstrates the **end-to-end workflow** of automatically creating a **TheHive Alert** from Wazuh logs and escalating it into a **Case** when the alert severity is high (≥7).  


## 🛠 Components Used

- **Wazuh**: Security Information & Event Management (SIEM) and XDR solution for threat detection.  

- **Shuffle**: Open-Source Security Orchestration, Automation, and Response  (SOAR) platform for automating workflows  

- **TheHive**: Open-source Security Incident Response Platform (SIRP) case management platform for alert triage, evidence handling, investigation tracking, and team collaboration    

- **Discord/Gmail**: For real-time monitoring and notifications.  

---

## Workflow Steps

### 1. Setup Webhook in Shuffle

- Create a new **Webhook Trigger** in Shuffle.  
- Set the webhook name to: **`Wazuh Alerts`**  
- Copy the endpoint URL of the webhook.

---

### 2. Add Webhook API in Wazuh (`ossec.conf`)

Add the webhook configuration to Wazuh:

```xml
<integration>
  <name>custom-webhook</name>
  <hook_url>http://<your-shuffle-ip>:3001/api/v1/webhook/<webhook-id></hook_url>
  <level>1</level>
  <alert_format>json</alert_format>
</integration>
```

Start the webhook in Shuffle, then restart the Wazuh manager:

```bash
sudo systemctl restart wazuh-manager
```

---

### 3. Log Ingestion to Shuffle

Logs from Wazuh are now sent to Shuffle via the webhook.

---

### 4. Create Repeat Trigger in Shuffle

- Add a **Repeat Back to Me** app in the workflow.  
- Set the name to: **`Repeat Alert`**

---

### 5. Filter Alerts

We are only concerned with alerts where **level > 7**.

- **If alert level < 7**:  
  Send a formatted message to **Discord** for visibility only.

- **If alert level >= 7**:  
  Continue to enrichment and case creation.

---

### 6. Map Severity with Python Script (`severity map`)

**Note:** Wazuh alerts include a severity field by default. Instead of running this script, you can directly use that value. The script below is kept only for knowledge/reference about how severity mapping could be done manually if needed :) 

```python
try:
    rule_level = int("{{ repeat_alert.all_fields.rule.level or 0 }}")
except Exception:
    rule_level = 0

# Map to IRIS severity
if rule_level < 5:
    severity = 2
elif rule_level < 7:
    severity = 3
elif rule_level < 10:
    severity = 4
elif rule_level < 13:
    severity = 5
else:
    severity = 6

# IMPORTANT: Return it as `output` (Shuffle expects this)
output = {
    "severity": severity
}
print(severity)  # This must be returned
```
---

### 7. Create Alert in TheHive

- Drag **TheHive App → Create Alert** into the Shuffle workflow.  
- Authenticate with a **dedicated TheHive API key** (avoid using the global admin account).

Use the following JSON body to create the alert:

  
```json
{
  "description": "Rule ID: $repeat_alert.all_fields.rule.id\nRule Level: $repeat_alert.all_fields.rule.level\nRule Description: $repeat_alert.all_fields.rule.description\nAgent ID: $repeat_alert.all_fields.agent.id\nAgent Name: $repeat_alert.all_fields.agent.name\nLocation: $repeat_alert.all_fields.location",
  "externalLink": "https://192.168.18.159/app/wz-home",
  "flag": true,
  "pap": 2,
  "severity": $repeat_alert.severity,
  "source": "Wazuh",
  "sourceRef": "Incident: $repeat_alert.timestamp",
  "status": "New",
  "summary": "Rule level: $repeat_alert.all_fields.rule.level, Event ID:  $repeat_alert.all_fields.data.win.system.eventID",
  "title": "$repeat_alert.all_fields.rule.description",
  "tlp": 2,
  "type": "Incident",
  "tags": [
    "$repeat_alert.rule_id",
    "$repeat_alert.all_fields.agent.name",
    "$repeat_alert.all_fields.agent.ip"
  ]
```
### 8. Create Ticket/Case in TheHive
- Next, add TheHive App → Create Case in Shuffle.
- Use the output of the alert creation step as input for the case.

Use the following JSON body:

```json
{
  "description": "$create_alert.body.description",
  "flag": true,
  "pap": 2,
  "severity": $create_alert.body.severity,
  "status": "New",
  "summary": "$create_alert.body.summary",
  "title": "$repeat_alert.all_fields.rule.description",
  "tlp": 2,
  "user": "$create_alert.body._createdBy",
  "tags": [
    "$repeat_alert.rule_id",
    "$repeat_alert.all_fields.agent.name",
    "$repeat_alert.all_fields.agent.ip"
  ]
}
```
---

### 9. Notify SOC via Gmail or Discord

Send alert or case summary to the SOC team via email or Discord for real-time notification.

1. **Create a Discord Channel for Informational Alerts**  
   - Open your Discord server  
   - Click **“+”** to create a new text channel (e.g., `#wazuh-Alerts`)  
   - Go to **Channel Settings → Integrations → Webhooks**  
   - Click **“New Webhook”**  
   - Name it (e.g., `Wazuh-SIEM-Alerts`) and copy the **Webhook URL**

2. **Send Message Payload**  
   Use the following JSON payload in Shuffle to post via the Discord Webhook:

```json
{
  "content": "**[🚨 Wazuh Security Alert – Case Created in TheHive]**\n\n**Level:** $repeat_alert.all_fields.rule.level\n**Rule ID:** $repeat_alert.rule_id\n**Title:** $repeat_alert.all_fields.rule.description\n**Source Agent:** $repeat_alert.all_fields.agent.name\n**Detected IP:** $repeat_alert.all_fields.agent.ip\n**Timestamp:** $repeat_alert.timestamp\n\n A new case has been automatically created in **TheHive(SIRP)** for investigation.\nSOC Analysts are advised to review and take action."
}

```
---

## 🔍 Next Step: Enable Enrichment in TheHive

Once alerts and cases are automatically created, you can enrich IOCs using tools like VirusTotal or MISP.
This enrichment can be automated using Cortex analyzers linked with TheHive.

---


> **Arfan Abid**   
> LinkedIn: https://www.linkedin.com/in/arfan-abid-152217270/