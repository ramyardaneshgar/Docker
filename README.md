# Docker
Leverage Docker to deploy a vulnerable target (DVWA) and scan it using OWASP ZAP from a separate, containerized environment for DevSecOps automation.

By Ramyar Daneshgar 


---

## Step 1: Installing Docker and Preparing the Environment

Docker acts as a **containerization platform**, allowing me to execute applications within **isolated user-space instances** (containers), which are more resource-efficient than traditional VMs because they use the host’s kernel rather than emulating a full OS.

### On Ubuntu:
```bash
sudo apt update
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

### On macOS (using Homebrew):
```bash
brew install --cask docker
open /Applications/Docker.app
```

I validated the install:
```bash
docker --version
```

This setup is foundational. It ensures I can build and run containers that simulate isolated assets—ideal for **threat modeling** and **controlled exploit testing**.

---

## Step 2: Pulling Base Images for Target and Scanner

I chose two Docker images for this assessment:

- **Target**: `vulnerables/web-dvwa` (Damn Vulnerable Web Application)
- **Scanner**: `owasp/zap2docker-stable` (Zed Attack Proxy in headless mode)

These represent an **attack surface** (DVWA) and an **active reconnaissance tool** (ZAP) respectively.

I pulled the official images:
```bash
docker pull vulnerables/web-dvwa
docker pull owasp/zap2docker-stable
```

This ensures image authenticity and avoids any tampering—important for maintaining **supply chain trust** in security tooling.

---

## Step 3: Deploying the Vulnerable Web Application (Target Surface)

I instantiated DVWA in a detached Docker container, mapping it to host port 8080:

```bash
docker run -d -p 8080:80 --name dvwa vulnerables/web-dvwa
```

**Explanation**:
- `-d`: runs in detached mode (background).
- `-p 8080:80`: maps port 80 inside the container to port 8080 on the host.
- `--name dvwa`: assigns a human-readable container name.

By containerizing the target, I limit its **network access**, create a predictable **exploit environment**, and avoid risk to my host system. The container acts as a **sandbox** for vulnerability testing.

---

## Step 4: Scanning with OWASP ZAP in Headless Mode

I now launched a second container to run **ZAP’s passive scanning mode** against DVWA. The goal here was to simulate a **black-box reconnaissance scan** against a live web target using a separate **network-bound attacker container**.

```bash
docker run -v $(pwd):/zap/wrk/:rw --network="host" \
  owasp/zap2docker-stable zap-baseline.py \
  -t http://localhost:8080 -r zap_report.html
```

### Breakdown:
- `-v $(pwd):/zap/wrk`: mounts the host’s current directory to ZAP’s working directory so I can retrieve reports.
- `--network="host"`: ensures ZAP can route directly to `localhost:8080` without needing Docker’s default bridge network. (Note: This is secure on Linux. On macOS/Windows, I would use container names instead.)
- `zap-baseline.py`: performs passive vulnerability discovery without active attacks.
- `-t http://localhost:8080`: the target URL to be scanned.
- `-r zap_report.html`: writes the results to a static HTML report.

This command represents a minimal, **non-intrusive vulnerability scan** suitable for **automated security baselining** in a CI/CD pipeline.

---

## Step 5: Reviewing Scan Output

Once the scan was complete, I opened the generated report:

```bash
open zap_report.html
```

Inside the report, ZAP identified numerous security weaknesses typical of a vulnerable application:
- Reflected **Cross-Site Scripting (XSS)**
- **SQL injection vectors**
- Misconfigured **cookie flags** (HttpOnly, Secure)
- Use of **cleartext communication** (unencrypted HTTP)

Each finding corresponds to a **CWE (Common Weakness Enumeration)** identifier, which aids in **triage** and **remediation prioritization**.

---

## Step 6: Automating the Environment with Docker Compose

For repeatability and automation, I defined both containers in a `docker-compose.yml` file. This enables **infrastructure-as-code** deployment, reducing manual setup and standardizing the environment.

```yaml
version: "3"
services:
  dvwa:
    image: vulnerables/web-dvwa
    ports:
      - "8080:80"

  zap:
    image: owasp/zap2docker-stable
    volumes:
      - ./zap-reports:/zap/wrk:rw
    network_mode: "host"
    command: >
      zap-baseline.py -t http://localhost:8080 -r zap_report.html
```

I launched both services with a single command:
```bash
docker-compose up
```

This created a fully reproducible testing pipeline—ideal for **continuous security testing** or for setting up a **Red Team/Blue Team exercise**.

---

## Step 7: Cleanup 

Once the assessment was complete, I cleaned up all running containers and resources:

```bash
docker stop dvwa
docker rm dvwa
docker system prune -f
```

Removing unused containers and images is good **container hygiene**, preventing resource bloat and reducing the **attack surface** of my host system.

---

## Lessons Learned

1. **Isolation by Design**  
   Docker provides strong **namespace isolation** (PID, NET, IPC, and MNT) which keeps the DVWA and ZAP environments separate. This is useful for red teamers running untrusted code or blue teamers recreating known exploits.

2. **Safe Vulnerability Testing**  
   I ran an intentionally vulnerable service without risking my host system or external exposure. The use of `--network=host` is a convenience during local scanning, but in real-world CI/CD scenarios, a **bridge network with explicit IP routing** would be more secure.

3. **Automation-Ready**  
   By using Docker Compose and headless ZAP, I now have a portable security test rig that can be plugged into a **DevSecOps pipeline** for early vulnerability detection.

4. **Scalable Threat Emulation**  
   This containerized approach can easily scale to multiple target containers (Juice Shop, bWAPP) and multiple scanners (Nikto, Burp CLI, Nmap), all coordinated through automation.
