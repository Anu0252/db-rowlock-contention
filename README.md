# Database Row-Level Lock Contention: The "Thundering Herd"

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=Prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)

A hands-on SRE lab demonstrating how high-density traffic on a single database record (like an E-Commerce Flash Sale) cascades into row-level lock contention, connection pool exhaustion, and site-wide outages.

👉 **[View the Interactive Architecture Diagram Here](https://[your-github-username].github.io/your-repo-name/)**

*(Note: Enable GitHub Pages in your repo settings pointing to the `main` branch to view the diagram).*

---

## 📖 The Problem Statement

During an e-commerce flash sale (e.g., a limited sneaker drop), thousands of users attempt to purchase the exact same item at the same millisecond. 

To maintain ACID compliance, relational databases (PostgreSQL/MySQL) assign an `ExclusiveLock` to the first query to reach the row. The remaining 9,999 queries are placed in a waiting queue. Because these queries cannot complete, they hold their database connection open. 

Within seconds, the database **connection pool is completely exhausted**. Users attempting to load the homepage or buy entirely unrelated items receive an `HTTP 500 / 504 Gateway Timeout` error, bringing down the entire platform, even though the database CPU is sitting idle at 10%.

---

## 🔬 Lab Simulation Overview

This lab simulates this exact scenario inside a Kubernetes cluster to observe the system degradation through Prometheus and Grafana.

### 1. The Setup
- **PostgreSQL StatefulSet** deployed on Kubernetes with a Persistent Volume.
- **pgbench** used to simulate a concurrent traffic spike (the shoppers).
- **Postgres Exporter** exposing metrics to Prometheus.

### 2. Executing the Simulation
We trigger the failure using two terminal windows:

**Window 1 (The Bottleneck):**
We manually acquire an exclusive lock and intentionally delay the commit.
```sql
BEGIN;
SELECT * FROM items WHERE id = 1 FOR UPDATE;
SELECT pg_sleep(20);
UPDATE items SET payload = 'locked-checkout' WHERE id = 1;
COMMIT;