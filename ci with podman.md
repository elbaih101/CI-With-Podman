Awesome—let’s turn that repo into a complete, production-ish setup where **Jenkins** and a **Selenium Grid** (plus **Appium**) all run on **Podman**, and your Jenkins pipelines use the Grid to run both web and mobile tests.

Below is a pragmatic, step-by-step path you can follow right now, plus best-practice notes where it matters.

---

# 0) Host prep (once)

```bash
# Install Podman & compose helper (names vary by distro)
sudo apt-get update && sudo apt-get install -y podman podman-compose jq curl || true
sudo dnf install -y podman podman-compose jq curl || true

# Allow user services to start at boot (for rootless containers)
loginctl enable-linger "$USER"

# Create a dedicated network for CI
podman network create ci
```

> Why this way: Podman works rootless, plays well with systemd, and can run multi-container apps via `podman-compose`. ([Podman Documentation][1])

---

# 1) Bring up Jenkins on Podman

You already have a good starting point in your repo. Here’s a minimal, clean version (no extra privileges) that persists data and binds ports 8080/50000:

```bash
podman volume create jenkins-data

podman run -d --name jenkins \
  --hostname jenkins \
  --network ci \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins-data:/var/jenkins_home \
  jenkins/jenkins:lts

# Get the initial admin password
podman logs -f jenkins | grep -m1 'Please use the following password'
```

(Your repo walks through variants—standalone or in a Pod—and shows how to autostart with systemd. Use it as reference.) ([GitHub][2])

**Best practice (prod):** manage long-running services with **systemd/Quadlet** so they start after reboots and get clean lifecycle management. ([Red Hat][3], [Podman Documentation][4])

---

# 2) Deploy Selenium Grid on Podman (web testing)

Create a folder and a `compose.yaml` for the Grid:

```bash
mkdir -p ~/grid && cd ~/grid
cat > compose.yaml <<'YAML'
services:
  selenium-hub:
    image: docker.io/selenium/hub:4.23.0
    container_name: selenium-hub
    networks: [ci]
    ports:
      - "4444:4444"
    healthcheck:
      test: ["CMD", "curl", "-fsSL", "http://localhost:4444/status"]
      interval: 10s
      timeout: 5s
      retries: 30

  chrome:
    image: docker.io/selenium/node-chrome:4.23.0
    container_name: node-chrome
    networks: [ci]
    shm_size: "2g"
    depends_on:
      selenium-hub:
        condition: service_healthy
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443

  firefox:
    image: docker.io/selenium/node-firefox:4.23.0
    container_name: node-firefox
    networks: [ci]
    shm_size: "2g"
    depends_on:
      selenium-hub:
        condition: service_healthy
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443

networks:
  ci:
    external: true
YAML

# start the grid
podman-compose up -d
```

Notes:

* `shm_size: "2g"` avoids Chrome/Firefox crashes due to small `/dev/shm`. ([GitHub][2])
* Podman happily pulls Docker Hub images (`docker.io/...`).
* Healthcheck waits until Grid is READY.

Check it:

```bash
curl -fsSL http://localhost:4444/status | jq .
```

---

# 3) Add Appium into the Grid (mobile testing)

**Approach:** Run one (or more) **Appium servers** alongside the Grid, then attach **Grid relay nodes** that forward mobile sessions to Appium. (This is the recommended pattern in Selenium Grid 4.) ([Appium][5])

### 3.1 Appium servers (example: two servers)

```bash
mkdir -p ~/grid/appium
cat > ~/grid/appium/appium1.yml <<'YAML'
server:
  port: 4723
  use-drivers:
    - uiautomator2
YAML

cat > ~/grid/appium/appium2.yml <<'YAML'
server:
  port: 4733
  use-drivers:
    - uiautomator2
YAML

cat >> ~/grid/compose.yaml <<'YAML'

  appium1:
    image: docker.io/appium/appium:latest
    container_name: appium1
    networks: [ci]
    command: ["appium","--config","/appium/appium1.yml"]
    volumes:
      - ./appium/appium1.yml:/appium/appium1.yml:Z
    ports: ["4723:4723"]

  appium2:
    image: docker.io/appium/appium:latest
    container_name: appium2
    networks: [ci]
    command: ["appium","--config","/appium/appium2.yml"]
    volumes:
      - ./appium/appium2.yml:/appium/appium2.yml:Z
    ports: ["4733:4733"]
YAML

podman-compose up -d appium1 appium2
```

> Real Android devices: grant USB access (host-dependent) by adding `--device /dev/bus/usb:/dev/bus/usb` (and possibly `--privileged`) to the Appium service, and ensure `adb` on the host can see the devices. iOS requires macOS hardware (use a remote Mac/Appium server). ([Appium][5])

### 3.2 Grid relay nodes that point to Appium

Create TOML configs for two relay nodes:

```bash
mkdir -p ~/grid/relay
cat > ~/grid/relay/node1.toml <<'TOML'
[server]
port = 5555

[node]
detect-drivers = false

[relay]
url = "http://appium1:4723"
status-endpoint = "/status"
# "1" means one concurrent session; adjust as needed
configs = [
  "1", "{\"platformName\":\"Android\",\"appium:automationName\":\"UiAutomator2\"}"
]
TOML

cat > ~/grid/relay/node2.toml <<'TOML'
[server]
port = 5565

[node]
detect-drivers = false

[relay]
url = "http://appium2:4733"
status-endpoint = "/status"
configs = [
  "1", "{\"platformName\":\"Android\",\"appium:automationName\":\"UiAutomator2\"}"
]
TOML
```

Attach them to your Grid using a lightweight Selenium node image:

```bash
cat >> ~/grid/compose.yaml <<'YAML'

  appium-relay-1:
    image: docker.io/selenium/node-base:4.23.0
    container_name: appium-relay-1
    networks: [ci]
    depends_on:
      selenium-hub:
        condition: service_healthy
      appium1:
        condition: service_started
    volumes:
      - ./relay/node1.toml:/opt/selenium/node.toml:Z
    command: ["selenium-node","--config","/opt/selenium/node.toml","--bind-host","0.0.0.0","--publish-events","tcp://selenium-hub:4442","--subscribe-events","tcp://selenium-hub:4443"]

  appium-relay-2:
    image: docker.io/selenium/node-base:4.23.0
    container_name: appium-relay-2
    networks: [ci]
    depends_on:
      selenium-hub:
        condition: service_healthy
      appium2:
        condition: service_started
    volumes:
      - ./relay/node2.toml:/opt/selenium/node.toml:Z
    command: ["selenium-node","--config","/opt/selenium/node.toml","--bind-host","0.0.0.0","--publish-events","tcp://selenium-hub:4442","--subscribe-events","tcp://selenium-hub:4443"]
YAML

podman-compose up -d appium-relay-1 appium-relay-2
```

Now all web & mobile sessions go to **[http://localhost:4444](http://localhost:4444)** (Grid), and Grid routes Android sessions to Appium via the relay. ([Appium][5])

---

# 4) Wire Jenkins to the Grid

Inside Jenkins (Manage Jenkins → Plugins), install at minimum: **Pipeline**, **Git**, **JUnit**, and whatever test framework publisher you use (e.g., Allure).

### Option A (recommended): keep the Grid always-on

* Jenkins simply targets `http://selenium-hub:4444` from any container on the `ci` network, or `http://localhost:4444` from the host.
* Your tests should use the **RemoteWebDriver URL = `http://<grid-host>:4444`** (no `/wd/hub` needed for W3C clients; both work). ([GitHub][2])

### Option B (ephemeral Grid per build)

* Have the pipeline run `podman-compose up -d` before tests and `podman-compose down -v` after.
* This requires the Jenkins **controller or agent** to have the `podman` CLI and access to your compose files (either on the host, or in a workspace mounted from the host).

> Tip: If you want Jenkins to call Podman’s Docker-compatible API (for Docker-Plugin style steps), run `podman system service --time=0` for your Jenkins user and set `DOCKER_HOST=unix:///run/user/UID/podman/podman.sock`. Many teams still prefer shelling out to `podman` directly for simplicity. ([Jenkins][6])

---

# 5) Example Jenkinsfile (works for both web + mobile)

This assumes your Grid is already up (Option A). Adjust the build container to your stack (e.g., Maven, Gradle, npm, pytest).

```groovy
pipeline {
  agent any

  environment {
    GRID_URL = 'http://localhost:4444'   // Or http://selenium-hub:4444 if running on the same Podman network
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Wait for Grid ready') {
      steps {
        sh '''
          echo "Waiting for Selenium Grid..."
          for i in $(seq 1 60); do
            READY=$(curl -fsSL ${GRID_URL}/status | jq -r .value.ready 2>/dev/null || echo false)
            [ "$READY" = "true" ] && break
            sleep 2
          done
          [ "$READY" = "true" ] || { echo "Grid not ready"; exit 1; }
        '''
      }
    }

    stage('Run Web UI tests') {
      steps {
        // Example: Maven tests reading GRID_URL from env and using RemoteWebDriver
        sh 'mvn -B -Dgrid.url=${GRID_URL} -Dheadless=true test || true'
      }
      post {
        always { junit 'target/surefire-reports/*.xml' }
      }
    }

    stage('Run Mobile (Appium) tests') {
      steps {
        // Example: Gradle/pytest/etc. Ensure capabilities request e.g. platformName=Android, appium:automationName=UiAutomator2
        sh './gradlew connectedAndroidTest -PgridUrl=${GRID_URL} || true'
      }
      post {
        always { junit '**/build/test-results/test/*.xml' }
      }
    }
  }

  post {
    always {
      archiveArtifacts allowEmptyArchive: true, artifacts: '**/screenshots/**'
    }
  }
}
```

---

# 6) Systemd (Quadlet) autostart (optional but great in prod)

Make the Grid (and Jenkins) restart on boot via Quadlet (preferred over the old `podman generate systemd`):

1. Create `~/.config/containers/systemd/selenium-hub.container` and `*.container` files for nodes; or one `*.kube` referencing a Compose-like stack if you export to kube YAML.
2. `systemctl --user daemon-reload && systemctl --user enable --now selenium-hub.container`

This gives you clean lifecycle management, restarts, logs via `journalctl`, etc. ([Red Hat][3], [Podman Documentation][4])

---

# 7) Best practices you’ll want

* **Pin image versions** (e.g., `selenium/hub:4.23.0`) to avoid surprises on upgrades. ([GitHub][2])
* **Health checks & waits** before running tests (as in `compose.yaml` and pipeline).
* **Enough `/dev/shm`** for browsers (`shm_size: "2g"`), or set Chrome flag `--disable-dev-shm-usage` if you can’t. ([GitHub][2])
* **Rootless Podman** and least-privilege containers; only add `--privileged` for Appium real-device USB if necessary.
* **Separate networks** (we used `ci`) and avoid publishing more ports than needed.
* **Resource limits** (`cpus`, `memory`) to prevent test flakiness from contention.
* **Logs & artifacts**: stream container logs to Jenkins (e.g., `podman logs --since=...`), archive screenshots/videos.
* **Parallelism**: scale nodes (`podman-compose up --scale chrome=3`) and Appium relays to match your desired concurrency. ([GitHub][2])
* **Keep Jenkins simple**: Let Jenkins talk HTTP to the Grid; avoid making Jenkins “control” containers unless you really need it. (If you do, wire the Podman socket for API compatibility.) ([Jenkins][6])

---

## Quick “smoke test” checklist

1. `podman ps` shows `jenkins`, `selenium-hub`, `node-chrome`, `node-firefox`, `appium1`, `appium2`, and the two relays.
2. `curl http://localhost:4444/status` → `value.ready=true`.
3. Open Jenkins at `http://<host>:8080`, run a pipeline using `GRID_URL=http://localhost:4444`.
4. In tests, request:

   * **Web**: standard W3C caps (no `legacy` JSONWP).
   * **Mobile**: `platformName=Android`, `appium:automationName=UiAutomator2`. Sessions should appear on the Grid UI.

---

If you want, I can tailor the `compose.yaml` and a `Jenkinsfile` to your repo layout and test stack (Java/JS/Python/…); just tell me what you’re running your tests with and if you prefer always-on or ephemeral Grid.

[1]: https://docs.podman.io/en/v5.3.1/markdown/podman-compose.1.html?utm_source=chatgpt.com "podman-compose"
[2]: https://github.com/elbaih101/jenkins-with-podman/tree/main "GitHub - elbaih101/jenkins-with-podman"
[3]: https://www.redhat.com/en/blog/quadlet-podman?utm_source=chatgpt.com "Make systemd better for Podman with Quadlet - Red Hat"
[4]: https://docs.podman.io/en/v5.2.5/markdown/podman-generate-systemd.1.html?utm_source=chatgpt.com "podman-generate-systemd"
[5]: https://appium.io/docs/en/2.0/guides/grid/ "Appium and Selenium Grid - Appium Documentation"
[6]: https://www.jenkins.io/doc/book/installing/docker/?utm_source=chatgpt.com "Docker - Jenkins"
