# 🧩 Wallarm Solutions Engineer Technical Evaluation  
**Author:** Ed Maunders  
**Date:** October 2025  
**Deployment Method:** Docker (Host Networking)  
**Backend:** Python API / TicketBox (booking_api)  
**Attack Simulation:** GoTestWAF  

---

## 🧭 Overview

This project demonstrates the deployment, configuration, and validation of a **Wallarm Filtering Node** using **Docker**.

I chose Docker for its simplicity, portability, and reproducibility — ideal for proof-of-concept and lab testing environments.  

The Wallarm node was configured to inspect incoming traffic, forward clean requests to a backend API, and block malicious requests.  
I followed the official documentation here: [Wallarm Docker Deployment Guide](https://docs.wallarm.com/installation/inline/compute-instances/docker/nginx-based/)

 Example Deployment Command

```bash
docker run -d --name wallarm-node --network host \
  -e WALLARM_API_TOKEN='{API_TOKEN}' \
  -e WALLARM_LABELS='group=ticketbox' \
  -e NGINX_BACKEND='http://127.0.0.1:8080' \
  -e WALLARM_API_HOST='audit.api.wallarm.com' \
  -e WALLARM_MODE='block' \
  wallarm/node:6.6.0
```

This setup allowed me to validate both detection and blocking capabilities using the GoTestWAF tool, simulating real-world attack traffic against the protected backend.

⚙️ Prerequisites

Before deployment, ensure the following components are available:

- Host Environment: Ubuntu 22.04 or similar Linux environment

- Docker: Version 20.x or higher installed and running

- Wallarm API Token: Active token from the Wallarm Cloud Console

- Backend Application: TicketBox API or httpbin.org reachable at http://127.0.0.1:8080

- GoTestWAF: Installed to generate and simulate attack traffic

- Internet Access: Required for Wallarm node registration via audit.api.wallarm.com

**Deploy Test Backend Application (TicketBox API)**

To validate that the Wallarm Node correctly forwards and filters traffic, a simple backend API was used — the TicketBox (booking_api) service.
This Python-based API simulates a real-world backend with endpoints for booking and session handling.
The application should be deployed locally on port 8080, so the Wallarm Node can forward clean traffic directly to it.

🪄 Step 1: Clone and Run the Backend Application

Clone the API from GitHub and start it locally using Docker:

```bash
git clone https://github.com/edmaunders/booking_api.git
cd booking_api
docker build -t booking_api .
docker run -d --name booking_api -p 8080:8080 booking_api
```
⚙️ Step 2: Verify Backend Functionality

Confirm that the backend is reachable before deploying the Wallarm Node:
```bash
curl http://127.0.0.1:8080/get


Expected response (example):

{
  "status": "ok",
  "service": "TicketBox Booking API",
  "message": "Backend running and reachable"
}
```
🚀 Deploy Wallarm Filtering Node

The Wallarm Node was deployed using Docker to inspect incoming HTTP traffic, forward clean requests to the backend API, and block malicious ones.

🪄 Step 1: Create a Node in the Wallarm Console

Navigate to Configuration → Nodes in the Wallarm Console.

Click Create Node, enter a name, and select Deployment / Node Usage Type.

Copy the generated API Token — you’ll need this in the next step.

⚙️ Step 2: Deploy the Wallarm Node Container

Run the Wallarm Node container in host networking mode to intercept requests directly.
Replace {API_TOKEN} with your own from the Wallarm Console.

docker run -d --name wallarm-node --network host \
  -e WALLARM_API_TOKEN='{API_TOKEN}' \
  -e WALLARM_LABELS='group=ticketbox' \
  -e NGINX_BACKEND='http://127.0.0.1:8080' \
  -e WALLARM_API_HOST='audit.api.wallarm.com' \
  -e WALLARM_MODE='block' \
  wallarm/node:6.6.0

### 🧩 Command Breakdown

| Parameter | Description |
|------------|-------------|
| `-d` | Runs the container in detached mode (background). |
| `--name wallarm-node` | Assigns a friendly name to the container. |
| `--network host` | Uses the host’s network stack so Wallarm can inspect traffic directly. |
| `-e WALLARM_API_TOKEN='{API_TOKEN}'` | Authenticates the node with Wallarm Cloud. |
| `-e WALLARM_LABELS='group=ticketbox'` | Adds metadata tags for node grouping. |
| `-e NGINX_BACKEND='http://127.0.0.1:8080'` | Defines the backend server for clean traffic. |
| `-e WALLARM_API_HOST='audit.api.wallarm.com'` | Specifies the API endpoint for your region. |
| `-e WALLARM_MODE='block'` | Enables blocking mode to deny malicious traffic. |
| `wallarm/node:6.6.0` | The Wallarm Docker image and version. |

✅ Step 3: Verify Deployment

Check that the container is running:
```bash
docker ps
```

View logs to confirm successful registration:
```bash
docker logs wallarm-node | grep "Wallarm node started"
```

Once registered, you’ll see the node in the Wallarm Console under Configuration → Nodes.

<p align="center">
  <img src="screenshots/Screenshot%202025-10-15%20at%2012.48.00.png" alt="Wallarm Node Verification" width="700"/>
  <br>

</p>


🧪 Run GoTestWAF to Validate Wallarm Detection (Docker)

Once both the backend API and Wallarm Node are running, use GoTestWAF to simulate legitimate and malicious requests.
This verifies that the Wallarm Node is detecting and blocking common web attacks such as SQL injection, XSS, and RCE.

Running GoTestWAF in a Docker container keeps the environment clean and avoids the need to install Go locally.

🪄 Step 1: Download the GoTestWAF Docker Image

Pull the latest GoTestWAF image from Docker Hub:
```bash
docker pull wallarm/gotestwaf
```
⚙️ Step 2: Run GoTestWAF

Execute the following command to start the test and generate reports:
```bash
docker run --rm --network host -u 0:0 \
  -v ~/gotestwaf_reports:/app/reports \
  wallarm/gotestwaf \
  --url=http://127.0.0.1 \
  --noEmailReport \
  --blockStatusCodes=403 \
  --testSet owasp
```

### 🧩 Command Breakdown

| Option | Description |
|---------|-------------|
| `docker run --rm` | Runs the container and removes it automatically after the test completes. |
| `--network host` | Sends all requests through the Wallarm Node at `127.0.0.1`. |
| `-u 0:0` | Runs as root to ensure permissions for writing report files. |
| `-v ~/gotestwaf_reports:/app/reports` | Mounts a local folder on the host to store generated reports. |
| `wallarm/gotestwaf` | The official Wallarm GoTestWAF Docker image. |
| `--url=http://127.0.0.1` | Sets the target URL for testing (the Wallarm Node in blocking mode). |
| `--noEmailReport` | Disables email delivery of the report (useful for local testing). |
| `--blockStatusCodes=403` | Treats HTTP 403 responses as blocked attacks. |
| `--testSet owasp` | Runs the OWASP Core test suite (SQLi, XSS, RCE, etc.). |

✅ Step 3: Review Results

Check the reports generated on your host:

```bash
ls -lh ~/gotestwaf_reports
waf-evaluation-report-2025-October-13-12-42-47.pdf
waf-evaluation-report-2025-October-13-12-42-47.csv
gotestwaf.log
```

Open the PDF report to review detection metrics such as:

Total Requests Sent

Blocked Attacks

Detection Coverage

False Negatives / Passed Attacks

<p align="center">
  <img src="screenshots/report.png" alt="Wallarm Node Verification" width="700"/>
  <br>

</p>

## 🔍 Check API Sessions and Attacks in the Wallarm Console

After running GoTestWAF, you can verify that the simulated traffic and detected attacks were correctly processed by your Wallarm Node and appear in the **Wallarm Console**.

---

### 🧩 Step 1: View API Sessions

1. Log in to your Wallarm Console:  
   - [Audit Cloud](https://my.audit.wallarm.com)  


2. Navigate to **Traffic Analysis → API Discovery → Endpoints**.  
3. Confirm that your backend endpoints (e.g., `/get`, `/login`, `/book`) are visible.  
   - These were automatically discovered from the traffic sent by GoTestWAF.  
   - Each endpoint should show HTTP methods, request counts, and average response codes.  

This confirms that the **Wallarm Node** successfully forwarded clean requests and recorded legitimate API traffic for analysis.

---

### ⚔️ Step 2: Review Detected Attacks

1. In the Console, open **Events → Attacks**.  
2. Filter by **Source: your Node name or tag (`group=ticketbox`)**.  
3. You should see multiple attack entries corresponding to the OWASP tests executed by GoTestWAF, such as:  
   - SQL Injection (`sqli`)  
   - Cross-Site Scripting (`xss`)  
   - Remote Code Execution (`rce`)  
   - Path Traversal (`pthtrv`)  
   - Command Injection (`cmdi`)
  
<p align="center">
  <img src="screenshots/attacks.png" alt="Wallarm Node Verification" width="700"/>
  <br>

</p>

4. Click on any attack entry to view:  
   - **Full request and response details**  
   - **Attack type and risk score**  
   - **Detection point** (parameter, header, or URI)  
   - **Blocking status (403)**
  
<p align="center">
  <img src="screenshots/details.png" alt="Wallarm Node Verification" width="700"/>
  <br>

</p>

---

### 🧠 Validation Outcome

If the deployment was successful:
- Legitimate traffic from GoTestWAF appears in **API sessions**.
 <p align="center">
  <img src="screenshots/API-Session.png" alt="Wallarm Node Verification" width="700"/>
  <br>

</p>
- Malicious requests appear in **Events / Attacks** with action `Blocked`.  
- All events are tagged with your node label (`group=ticketbox`).  




This demonstrates that your Wallarm Node is correctly integrated with the Console, registering both **legitimate API sessions** and **blocked malicious attacks**.








This confirmed that the Wallarm Node was successfully operating in blocking mode, intercepting and denying the majority of simulated OWASP attacks.

🧠 Troubleshooting Tips

If no reports appear, verify directory permissions:
sudo chmod 777 ~/gotestwaf_reports

Ensure both the backend and Wallarm Node are reachable at 127.0.0.1.

Note: --network host works on Linux; macOS/Windows require alternate networking.






📖 References

Wallarm Docker Deployment Docs

Wallarm API Tokens Guide

GoTestWAF Repository

Booking API GitHub Repo
