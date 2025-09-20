# Docker Swarm

## 🐳 What is Docker Swarm?

Docker Swarm (aka **Swarm mode**) is Docker Engine’s **built-in container orchestrator**.
It turns a group of Docker hosts into a **fault-tolerant cluster**, letting you run containers as **services** with rolling updates, self-healing, service discovery, and secure networking—using the same Docker CLI you already know.

👉 If Docker is how you **run a container**, Swarm is how you **run containers in production** across multiple machines.

---

## 🧐 Why Do We Need Swarm?

Modern apps need **scale, resilience, and zero-downtime updates**:

* Keep N replicas running—**auto-replace** failed ones.
* **Distribute** workloads across nodes.
* **Expose** services behind cluster IP + load balancing.
* **Update** safely (canary/rolling) and **roll back** on failure.
* **Secure** inter-node comms (mTLS) and handle **secrets**.

Swarm delivers these with **simple, Docker-native** workflows.

---

## 🔧 How Swarm Works (Core Ideas)

* **Node** — a Docker host in the cluster.

  * **Manager** nodes maintain **Raft** state & schedule tasks.
  * **Worker** nodes run **tasks** (container instances).
* **Service** — desired state (image, replicas, ports, constraints).

  * **Task** — one running container for a service.
  * **Modes** — `replicated` (N copies) or `global` (1 per node).
* **Overlay networks** — multi-host virtual networks with built-in **service discovery** (`DNS`), VIP load-balancing, optional **encryption**.
* **Routing Mesh** — publishes ports **on every node**, forwards to healthy tasks.
* **Secrets & Configs** — securely distribute small files to services.
* **Stacks** — app bundles defined with **Compose v3** (`docker stack`).

---

### 🔗 Architecture Overview

```text
+-------------------+           +-------------------+           +-------------------+
|   Manager Node    |  mTLS     |   Manager Node    |  mTLS     |   Manager Node    |
|  - Raft consensus |<--------->|  - Scheduling     |<--------->|  - CA/Cert rotate |
+---------+---------+           +---------+---------+           +---------+---------+
          |                                 |                               |
          v                                 v                               v
   +------+-------+                  +------+-------+                +------+-------+
   |  Worker Node |                  |  Worker Node |                |  Worker Node |
   |  Tasks/Pods  |                  |  Tasks/Pods  |                |  Tasks/Pods  |
   +------+-------+                  +------+-------+                +------+-------+
          \___________________ Overlay Networks (VXLAN, optional encryption) ______/
```

---

## 🔄 Control & Data Flow

```mermaid
sequenceDiagram
    participant CLI as docker CLI
    participant MGR as Manager (Raft)
    participant NW as Overlay Net + VIP
    participant T as Service Tasks

    CLI->>MGR: docker service update --image app:v2
    MGR-->>MGR: Plan rolling update (update_config)
    MGR->>T: Replace tasks (start-first/stop-first)
    T-->>MGR: Health OK (or failure)
    MGR->>NW: Update VIP backends
    CLI-->>MGR: docker service ps app
```

---

## 🧰 Quickstart

### 1) Initialize a Swarm (first manager)

```bash
# on node A
docker swarm init --advertise-addr <MANAGER_IP>
```

Grab the printed **join token**.

### 2) Join more nodes

```bash
# on nodes B, C ...
docker swarm join --token <worker-token> <MANAGER_IP>:2377

# (optional) join more managers
docker swarm join --token <manager-token> <MANAGER_IP>:2377
```

### 3) Create a service

```bash
docker service create --name web --replicas 3 -p 80:80 nginx:alpine
docker service ls
docker service ps web
```

### 4) Scale / Update / Roll back

```bash
docker service scale web=6
docker service update --image nginx:1.27 --update-order start-first web
docker service rollback web
```

---

## 🧱 Stacks with Compose (v3+)

### docker-compose.yml

```yaml
version: "3.9"

services:
  api:
    image: ghcr.io/acme/api:1.2.3
    ports:
      - target: 8080
        published: 8080
        protocol: tcp
        mode: ingress   # or host
    networks: [appnet]
    deploy:
      replicas: 4
      update_config:
        parallelism: 1
        order: start-first
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 2
      restart_policy:
        condition: on-failure
        delay: 3s
        max_attempts: 3
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
      placement:
        constraints:
          - node.role == worker
          - node.labels.zone == eu-central
        preferences:
          - spread: node.labels.az

  worker:
    image: ghcr.io/acme/worker:2.0
    deploy:
      mode: global
    networks: [appnet]

networks:
  appnet:
    driver: overlay
    attachable: true
```

Deploy:

```bash
docker stack deploy -c docker-compose.yml acme
docker stack ls
docker stack services acme
docker stack ps acme
docker stack rm acme
```

---

## 🌐 Networking in Swarm

* **Service Discovery**: each service gets DNS (`tasks.<name>`) and a **VIP** (virtual IP).
* **Load Balancing**: VIP distributes L4 traffic across healthy tasks.
* **Routing Mesh (ingress)**:

  * `-p 80:80` exposes on **all nodes**; any node forwards to a task.
  * Use `mode: host` (or CLI `--publish mode=host`) to expose **only on the node** where a task runs.
* **Overlay Networks**:

  * Create: `docker network create -d overlay --attachable appnet`
  * **Encrypt** overlay: `docker network create -d overlay --opt encrypted secure-net`

**Ports to open between nodes**

* `2377/tcp` (cluster management), `7946/tcp+udp` (gossip), `4789/udp` (VXLAN).
* Published service ports (your choice).

---

## 🔒 Security Model

* **Mutual TLS** between nodes by default (built-in **CA**, automatic cert rotation).
* **Raft log** on managers is encrypted; back up the state.
* **Auto-lock** (optional): requires unlock key on daemon restart (`docker swarm update --autolock=true`).
* **Secrets & Configs**:

  * Create: `docker secret create db_pass -` then paste.
  * Mounted in containers at **`/run/secrets/<name>`** (tmpfs).
  * Configs similar, mounted read-only (often under `/`).
  * Keep <small>—ideal for creds, certs, small config files.

---

## 🔁 Updates, Health & Self-Healing

* **Rolling updates** with `update_config` (parallelism, delay, failure\_action).
* **Update order**: `start-first` (zero-downtime) or `stop-first`.
* **Healthchecks** in your Dockerfile/Compose gate promotions.
* **Restart policies** (`on-failure`, `any`, `none`).
* **Automatic rescheduling** of failed tasks to healthy nodes.

---

## 📦 Persistent Storage

Containers are ephemeral; tasks may move. Use:

* **Remote/shared volumes** (e.g., NFS, SMB, Ceph, Portworx, etc.) via volume plugins.
* Or **pin** stateful services with **constraints** to labeled nodes that host the data.
* Avoid default **local** volumes for movable services unless you accept data locality.

---

## 📈 Logs & Metrics

* `docker service logs <svc>` (depends on chosen **logging driver**).
* Drivers: `json-file`, `journald`, `syslog`, `gelf`, `fluentd`, `awslogs`, etc.
* Node/cluster events: `docker events` and `docker inspect` for deep dives.

---

## 🗺️ Placement & Scheduling

* **Constraints** (hard rules): `node.role == worker`, `node.labels.disk == ssd`.

  * Label nodes: `docker node update --label-add disk=ssd <node>`
* **Preferences** (soft rules): spread by `node.labels.az` to balance failure domains.
* **Resources**: set **reservations/limits** so the scheduler makes good decisions.

---

## 🧪 Service Types

* **Replicated** — run N copies across the cluster.
* **Global** — run 1 copy per eligible node (great for agents/daemons).
* **DNS round-robin** — skip VIP with `endpoint_mode: dnsrr` (hand control to app-level LB).

---

## 🧭 Operations Playbook

**Managers & quorum**

* Use **odd number** of managers: 3/5/7.
* Tolerates `⌊(N-1)/2⌋` failures (e.g., 3 managers → tolerate 1 down).
* Avoid running workloads on managers; set `availability drain`.

**Upgrades**

```bash
docker node update --availability drain <node>
# upgrade Docker, reboot, etc.
docker node update --availability active <node>
```

**Backups**

```bash
# On a manager (and when cluster is healthy):
# Save /var/lib/docker/swarm (Raft) while engine is stopped or via documented snapshot method.
```

**Common checks**

```bash
docker node ls
docker node ps <node>
docker service ps <svc> --no-trunc
docker service inspect <svc> --pretty
docker network inspect <net>
```

---

## ⚠️ Limitations & Watch-Outs

* No built-in **horizontal auto-scaling** (need external tooling).
* **Stateful** workloads require deliberate storage design.
* **Routing Mesh** is L4 only; advanced L7 needs a proxy (Traefik, Nginx, HAProxy).
* Smaller ecosystem vs Kubernetes; fewer controllers/operators.
* Cross-cloud WAN clusters can be finicky (latency, firewalls, MTU).

---

## 🥊 Swarm vs Compose vs Kubernetes

| Use Case                | Compose (single host) | **Swarm**               | Kubernetes     |
| ----------------------- | --------------------- | ----------------------- | -------------- |
| Scope                   | Dev, single VM        | Small–mid prod clusters | Mid–large prod |
| HA & Scheduling         | ❌                     | ✅ (simple)              | ✅ (rich)       |
| Rolling updates         | ⚠️ (limited)          | ✅                       | ✅              |
| Auto-scaling            | ❌                     | ❌                       | ✅              |
| Ecosystem/Extensibility | Low                   | Medium                  | High           |
| Complexity              | Low                   | **Low–Med**             | High           |

---

## 🔐 Example: Secrets + Encrypted Overlay

```bash
# network with encryption
docker network create -d overlay --opt encrypted prod-net

# secret
printf 's3cr3tP@ss' | docker secret create db_password -

# compose snippet
cat > stack.yml <<'YAML'
version: "3.9"
services:
  db:
    image: postgres:16
    networks: [prod-net]
    secrets: [db_password]
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    deploy:
      placement:
        constraints: [ "node.labels.role == db" ]
      restart_policy: { condition: on-failure }
networks:
  prod-net: { driver: overlay }
secrets:
  db_password: { external: true }
YAML

docker stack deploy -c stack.yml data
```

---

## 🧾 Swarm Cheat Sheet

### Cluster

```bash
docker swarm init | join | leave --force
docker node ls | inspect | update --label-add team=payments <node>
docker node update --availability drain|active <node>
```

### Services

```bash
docker service create --name api --replicas 3 -p 8080:8080 ghcr.io/acme/api:1.0
docker service ls | ps api | logs -f api
docker service scale api=8
docker service update --image ghcr.io/acme/api:1.1 --update-order start-first api
docker service rollback api
```

### Stacks

```bash
docker stack deploy -c docker-compose.yml app
docker stack services app
docker stack ps app
docker stack rm app
```

### Networks & Secrets

```bash
docker network create -d overlay --attachable appnet
docker secret create tls_key key.pem
docker config create app_conf app.ini
```

---

## 🧪 Common Patterns

* **Blue-green**: run `app_v1` and `app_v2` with different published ports; swap LB.
* **Canary**: temporarily scale a `:next` version to 5–10% of replicas via constraints.
* **Node pinning**: place GPU jobs with `node.labels.gpu == true`.
* **Sidecar**: run log shippers or metrics collectors as global services.

---

## 🧯 Troubleshooting Tips

* Task flapping? Check **healthchecks** & `restart_policy`.
* No connectivity? Verify **firewall ports** `2377`, `7946`, `4789` and MTU.
* VIP oddities? Try `endpoint_mode: dnsrr` or `--publish mode=host`.
* Overlay issues? Ensure consistent **subnet** ranges and `iptables` not blocking VXLAN.
* Quorum lost? **Do not** force-new-cluster casually; restore from backup.

---

## 🎯 Final Takeaway

Docker Swarm gives you **simple, secure, Docker-native orchestration**:

* ✅ Easy to learn (Docker CLI/Compose).
* ✅ Built-in **mTLS**, **service discovery**, **rolling updates**, **self-healing**.
* ✅ Great fit for small–medium clusters and teams prioritizing **simplicity**.

Design storage thoughtfully, keep **odd manager counts**, and use **stacks** + **overlay networks** to ship reliable apps with minimal overhead.

# 🐳 Cheat Sheet

```bash
# Initialize Swarm on the first node
docker swarm init

# Get join token for workers
docker swarm join-token worker

# Get join token for managers
docker swarm join-token manager

# Join a worker/manager node to the cluster
docker swarm join --token <TOKEN> <MANAGER-IP>:2377
```

---

# 📦 Stack Deployment

```bash
# Deploy the stack (monitoring here is the stack name)
docker stack deploy -c docker-compose.yml monitoring

# List stacks
docker stack ls

# List services in the stack
docker stack services monitoring

# List running tasks (replicas) for a service
docker service ps monitoring_prometheus
```

---

# ⚙️ Service Management

```bash
# Scale a service (e.g., run 5 replicas of nginx-app)
docker service scale monitoring_nginx-app=5

# Update a service (e.g., rolling restart)
docker service update --force monitoring_prometheus

# Remove a service
docker service rm monitoring_nginx-app
```

---

# 🐳 Container & Node Debugging

```bash
# List all swarm nodes
docker node ls

# Inspect details about a node
docker node inspect <NODE-ID> --pretty

# Show running containers on this node
docker ps

# Exec into a running container
docker exec -it <CONTAINER-ID> sh

# Show logs from a service (aggregates all tasks)
docker service logs monitoring_prometheus

# Show logs from one container
docker logs <CONTAINER-ID>
```

---

# 📊 Prometheus & Grafana

```bash
# Access Prometheus UI
http://localhost:9090

# Access Grafana UI
http://localhost:3000  (default admin/admin if not changed)

# Prometheus test query
up

# Check which targets are being scraped
http://localhost:9090/targets
```

---

# 🛠 Prometheus Config Reload

Prometheus doesn’t auto-reload config in Swarm unless restarted:

```bash
# Force service to reload config (rolling update)
docker service update --force monitoring_prometheus
```

---

# 🧹 Cleanup

```bash
# Remove the whole stack
docker stack rm monitoring

# Leave the swarm (on workers)
docker swarm leave

# Leave the swarm (on manager, forcefully)
docker swarm leave --force
```

---

⚡ With just these commands you can:

* Stand up your monitoring stack
* Scale apps
* Debug targets in Prometheus
* Import Grafana dashboards

---