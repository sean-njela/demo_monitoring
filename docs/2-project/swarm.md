Got it ✅ — here’s a **Docker Swarm + Prometheus/Grafana Monitoring Cheat Sheet** with the **commands you’ll actually use**, from setup to debugging.

---

# 🐳 Swarm Setup

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

👉 Do you want me to also give you a **cheat sheet of Grafana dashboards IDs & PromQL queries** that are most useful for infra/app monitoring?
