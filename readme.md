# Comprehensive Project Documentation: Azure Identity Demo Ingestion

This documentation provides a complete end-to-end guide for building a modern identity demo. It covers the front-end interface, the Azure integration bridge, and the backend security analytics configuration.

---

## 1. Project Overview
This project simulates a secure login portal to generate telemetry. Unlike legacy systems, it uses a **Data Collection Rule (DCR)** ingestion path, ensuring that logs are natively indexed by **Microsoft Defender XDR** and visible in **Advanced Hunting**.

---

## 2. Infrastructure Setup (The "How-To")

### A. Data Collection Endpoint (DCE)
The DCE is the physical gateway for your logs.
1.  **Search**: Search for 'Data Collection Endpoints' in the Azure portal.
2.  **Create**: Deploy a new DCE in the **same region** as your Log Analytics Workspace.
3.  **Logs Ingestion URL**: Copy the URL from the **Overview** blade (e.g., `https://...ingest.monitor.azure.com`).

### B. Custom Table (DCR-based)
1.  **Navigate**: Log Analytics Workspace > Settings > Tables.
2.  **Create**: Select **Create > New custom log (DCR-based)**.
3.  **Name**: `DemoSignInLogs` (the `_CL` suffix is added automatically).
4.  **DCR**: Select **Create new** for the Data Collection Rule during this process.
5.  **Schema**: Upload the sample JSON (Section 5) to automatically define columns.

### C. Data Collection Rule (DCR) Configuration
1.  **Search**: Search for 'Data Collection Rules' in the portal.
2.  **Immutable ID**: Open the **JSON View** of your DCR and copy the `immutableId`.
3.  **Transformation**: Go to **Data Sources > Inbound Stream > Transformation Editor**.
4.  **KQL**: Paste the following to map the mandatory `TimeGenerated` field:
    ```kusto
    source | extend TimeGenerated = todatetime(timestamp)
    ```

### D. IAM Permissions
The Logic App requires authorization to write to the DCR.
1.  **Resource**: Go to your **Data Collection Rule**.
2.  **Role**: Assign **Monitoring Metrics Publisher**.
3.  **Assignee**: Select the **System-assigned Managed Identity** of your Logic App.

---

## 3. The Bridge: Azure Logic App
The Logic App securely handles data from the web and delivers it to the ingestion API.

### Configuration Details
* **Method**: `POST`
* **URI**: `https://{DCE-URL}/dataCollectionRules/{DCR-ID}/streams/Custom-DemoSignInLogs_CL?api-version=2023-01-01`
* **Headers**: `Content-Type: application/json`
* **Body Expression**: `[ @triggerBody() ]` (Data must be sent as a JSON array).
* **Authentication**: Managed Identity with Audience `https://monitor.azure.com/`.

### Logic App Code View (JSON)
```json
{
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "contentVersion": "1.0.0.0",
        "triggers": {
            "When_an_HTTP_request_is_received": {
                "type": "Request",
                "kind": "Http",
                "inputs": {
                    "schema": {
                        "type": "object",
                        "properties": {
                            "username": { "type": "string" },
                            "pin": { "type": "string" },
                            "timestamp": { "type": "string" },
                            "clientIP": { "type": "string" },
                            "action": { "type": "string" }
                        }
                    }
                }
            }
        },
        "actions": {
            "HTTP": {
                "runAfter": {},
                "type": "Http",
                "inputs": {
                    "uri": "https://{DCE-URL}/dataCollectionRules/{DCR-ID}/streams/Custom-DemoSignInLogs_CL?api-version=2023-01-01",
                    "method": "POST",
                    "headers": { "Content-Type": "application/json" },
                    "body": [ "@triggerBody()" ],
                    "authentication": {
                        "type": "ManagedServiceIdentity",
                        "audience": "https://monitor.azure.com/"
                    }
                }
            }
        }
    }
}
```

---

## 4. Front-End: Modern Identity Portal
The website features a **Glassmorphism UI** and captures the user's **real public IP**. It also enforces a **4-digit numeric PIN**.

### Features
* **IP Discovery**: Calls `api.ipify.org` to fetch the actual public IP before ingestion.
* **PIN Enforcement**: Regex prevents non-numeric input; `maxlength="4"` limits length.
* **Auto-Reset**: Clears the PIN field upon success while keeping the Username for repeated testing.

### Full HTML Code
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Azure Identity Demo</title>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
    <style>
        :root { --primary: #0078d4; --bg: #0a0a0a; --glass: rgba(255, 255, 255, 0.05); }
        body { font-family: 'Inter', sans-serif; background: radial-gradient(circle at top right, #1a1a2e, var(--bg)); color: white; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
        .login-card { background: var(--glass); backdrop-filter: blur(15px); border: 1px solid rgba(255, 255, 255, 0.1); padding: 40px; border-radius: 20px; box-shadow: 0 25px 50px rgba(0,0,0,0.5); width: 320px; text-align: center; }
        h3 { font-weight: 300; margin-bottom: 30px; letter-spacing: 1px; color: #ccc; }
        .input-group { position: relative; margin-bottom: 20px; }
        .input-group i { position: absolute; left: 15px; top: 50%; transform: translateY(-50%); color: #666; }
        input { width: 100%; padding: 12px 15px 12px 45px; background: rgba(0, 0, 0, 0.2); border: 1px solid rgba(255, 255, 255, 0.1); border-radius: 8px; color: white; outline: none; box-sizing: border-box; }
        button { width: 100%; padding: 12px; background: var(--primary); border: none; border-radius: 8px; color: white; font-weight: 600; cursor: pointer; transition: all 0.3s; margin-top: 10px; }
        button:hover { background: #005a9e; box-shadow: 0 0 15px rgba(0, 120, 212, 0.4); }
        #status { margin-top: 20px; font-size: 12px; color: #aaa; }
    </style>
</head>
<body>
<div class="login-card">
    <i class="fa-solid fa-shield-halved" style="font-size: 40px; color: var(--primary); margin-bottom: 15px;"></i>
    <h3>SECURE PORTAL</h3>
    <div class="input-group">
        <i class="fa-regular fa-user"></i>
        <input type="text" id="username" placeholder="Username" required autocomplete="off">
    </div>
    <div class="input-group">
        <i class="fa-solid fa-key"></i>
        <input type="password" id="pin" placeholder="4-digit PIN" maxlength="4" inputmode="numeric" oninput="this.value = this.value.replace(/[^0-9]/g, '')" required>
    </div>
    <button onclick="sendData()" id="btnText">Sign In</button>
    <p id="status"></p>
</div>
<script>
    async function sendData() {
        const userEl = document.getElementById('username');
        const pinEl = document.getElementById('pin');
        const statusEl = document.getElementById('status');
        const btnText = document.getElementById('btnText');
        const logicAppUrl = "YOUR_LOGIC_APP_URL";
        if (!userEl.value || pinEl.value.length !== 4) { statusEl.innerHTML = '<span style="color: #ff4d4d;">Invalid credentials.</span>'; return; }
        btnText.disabled = true; btnText.innerText = "Processing...";
        try {
            const ipRes = await fetch('https://api.ipify.org?format=json');
            const ipData = await ipRes.json();
            const payload = { username: userEl.value, pin: pinEl.value, timestamp: new Date().toISOString(), clientIP: ipData.ip, action: "LoginAttempt" };
            const response = await fetch(logicAppUrl, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) });
            if (response.ok) { statusEl.innerHTML = 'Log ingested.'; pinEl.value = ""; } else { statusEl.innerText = "Failed."; }
        } catch (e) { statusEl.innerText = "Connection error."; } finally { btnText.disabled = false; btnText.innerText = "Sign In"; }
    }
</script>
</body>
</html>
```

---

## 5. Sample JSON Payload
Use this to define your DCR schema:
```json
{
  "username": "Clippy",
  "pin": "0987",
  "timestamp": "2026-02-23T09:24:22.547Z",
  "clientIP": "1.2.3.4",
  "action": "LoginAttempt"
}
```

---

## 6. Verification (Advanced Hunting)
```kusto
DemoSignInLogs_CL
| project TimeGenerated, username, pin, clientIP, action
| order by TimeGenerated desc
```
