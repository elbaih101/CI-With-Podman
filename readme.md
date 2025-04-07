# Running Jenkins with Podman: A Comprehensive Guide

## Table of Contents
1. General Podman Commands
2. Running Any Container with Podman
3. Podman Pod Options and Usage
4. Setting Up Jenkins with Podman
5. Using a Pod for Jenkins
6. Automating Jenkins Pod Startup with systemd

---

## 1. General Podman Commands
### Basic Podman Commands
| Command | Description |
|---------|-------------|
| `podman pull <image>` | Pull an image from a registry. |
| `podman images` | List downloaded images. |
| `podman run <options> <image>` | Run a container from an image. |
| `podman ps -a` | List all running and stopped containers. |
| `podman stop <container>` | Stop a running container. |
| `podman start <container>` | Start a stopped container. |
| `podman rm <container>` | Remove a stopped container. |
| `podman rmi <image>` | Remove an image. |
| `podman volume create <name>` | Create a persistent volume. |
| `podman volume ls` | List all volumes. |

### Managing Pods
| Command | Description |
|---------|-------------|
| `podman pod create --name <pod>` | Create a new pod. |
| `podman pod list` | List all pods. |
| `podman pod inspect <pod>` | Inspect pod details. |
| `podman pod stop <pod>` | Stop a pod. |
| `podman pod start <pod>` | Start a pod. |
| `podman pod rm <pod>` | Remove a pod. |
| `podman pod pause <pod>` | Pause all containers in a pod. |
| `podman pod unpause <pod>` | Unpause all containers in a pod. |

---

## 2. Running Any Container with Podman
To run a container using Podman:
```sh
podman run -d --name <container_name> -p <host_port>:<container_port> <image>
```
Example:
```sh
podman run -d --name nginx -p 8080:80 nginx
```

To persist container data, use a volume:
```sh
podman volume create <volume_name>
```
Then, mount it:
```sh
podman run -d --name <container_name> -v <volume_name>:/container/path <image>
```

---

## 3. Podman Pod Options and Usage
### Commonly Used `podman pod create` Options
| Option | Description |
|--------|-------------|
| `--name <name>` | Assigns a custom name to the pod. |
| `-p <host_port>:<container_port>` | Maps host ports to the pod's network. |
| `--network <type>` | Specifies the network mode (`bridge`, `host`, `none`). |
| `--hostname <name>` | Sets a hostname for the pod. |
| `--infra` | Creates an infra container to manage namespaces. |
| `--infra-image <image>` | Uses a custom infra container image. |
| `--share <namespace>` | Specifies which namespaces to share (`cgroup`, `ipc`, `net`, `pid`, `uts`). |
| `--memory <size>` | Limits memory usage for the pod. |
| `--cpu-shares <number>` | Assigns CPU shares for the pod. |

Example:
```sh
podman pod create --name jenkins-pod -p 8080:8080 -p 50000:50000 --memory 2G --cpu-shares 512
```

---

## 4. Setting Up Jenkins with Podman
### 1. Create a Named Volume for Jenkins Data
```sh
podman volume create jenkins-data
```

### 2. Run Jenkins as a Standalone Container
```sh
podman run -d --name jenkins \
    --privileged \
    -p 8080:8080 \
    -p 50000:50000 \
    -v jenkins-data:/var/jenkins_home \
    jenkins/jenkins:lts
```

### 3. Retrieve Initial Admin Password
```sh
podman logs jenkins | grep "Admin password"
```

### 4. Access Jenkins
Open a browser and go to:  
**http://localhost:8080**

---

## 5. Using a Pod for Jenkins
### 1. Create a Pod
```sh
podman pod create --name jenkins-pod -p 8080:8080 -p 50000:50000
```

### 2. Run Jenkins Inside the Pod
```sh
podman run -d --name jenkins \
    --pod jenkins-pod \
    --user root \
    --privileged \
    -v jenkins-data:/var/jenkins_home \
    jenkins/jenkins:lts
```

### 3. Add a Reverse Proxy (Example: Nginx)
```sh
podman run -d --name nginx \
    --pod jenkins-pod \
    nginx
```

### 4. Retrieve Jenkins Password
```sh
podman logs jenkins | grep "Admin password"
```

### 5. Start and Stop the Pod
```sh
podman pod start jenkins-pod
podman pod stop jenkins-pod
```

---

## 6. Automating Jenkins Pod Startup with systemd
### 1. Generate a systemd Service for the Pod
```sh
podman generate systemd --name jenkins-pod > ~/.config/systemd/user/jenkins-pod.service
```

### 2. Enable and Start the Service
```sh
systemctl --user enable jenkins-pod
systemctl --user start jenkins-pod
```

### 3. Ensure the User Service Starts on Boot
```sh
loginctl enable-linger $USER
```

Now, Jenkins will start automatically after a reboot.

---

## Conclusion
This guide provides both a **standalone** and **pod-based** setup for Jenkins using Podman. If you plan to expand your Jenkins environment later (e.g., add agents, reverse proxies), the **pod-based** approach is recommended. The guide also covers essential Podman commands, pod creation options, and systemd automation to maintain a persistent Jenkins setup.

