[README_Hybrid_Homelab_Networking.md](https://github.com/user-attachments/files/33140115/README_Hybrid_Homelab_Networking.md)
# DIY Hybrid Homelab: Two-Node Networking & Distributed Worker Project

A hands-on networking and distributed-systems homelab built from an old
Linux laptop and an Android phone running Termux.

This project demonstrates how to connect heterogeneous devices on a home
LAN, assign stable addresses, configure SSH key authentication, run
containerized infrastructure, expose services between nodes, and build a
Redis-backed distributed job queue.

> **Privacy note:** All usernames, MAC addresses, passwords,
> host-specific identifiers, and real LAN addresses from the original
> build have been replaced with examples/placeholders.

## Architecture

``` text
                         Home Router
                         192.168.1.1
                              |
                 +------------+------------+
                 |                         |
                 |                         |
        Linux Server                  Android Worker
        <SERVER_IP>                   <WORKER_IP>
        hostname: therock             Termux / ARM64
                 |                         |
          +------+-------+                 |
          |              |                 |
        Docker        Portainer            |
          |                                |
     +----+----+                           |
     |         |                           |
   Nginx     Redis <-----------------------+
              |
           Job Queue
```

The Linux laptop is the primary infrastructure node. The Android/Termux
device is an unprivileged worker node. Because standard Docker Engine
requires kernel/cgroup capabilities that were unavailable inside the
unprivileged Termux environment, Docker runs only on the Linux server
while the Android device participates at the application/network layer.

## What You Learn

-   LAN addressing and routing
-   DHCP reservations
-   Linux network interfaces
-   SSH aliases and public-key authentication
-   Remote command execution
-   Docker bridge networking
-   Portainer
-   Nginx
-   Redis
-   TCP service connectivity
-   Producer/consumer queues
-   Distributed job processing
-   Heterogeneous x86_64 + ARM64 systems
-   Foundations for CI/CD and hybrid-cloud integration

## Requirements

### Primary server

-   Linux laptop/server
-   Docker Engine
-   Docker Compose
-   SSH client
-   LAN connectivity

### Worker

-   Android device
-   Termux
-   OpenSSH
-   Python 3
-   Same LAN as the primary server

This guide uses these placeholders:

``` text
Router:       <ROUTER_IP>
Linux server: <SERVER_IP>
Worker:       <WORKER_IP>
SSH user:     <WORKER_USER>
Worker SSH:   8022
Redis secret: <REDIS_PASSWORD>
```

Example private addresses might be `192.168.1.1`, `192.168.1.20`, and
`192.168.1.21`. Use the addresses assigned by your own network.

------------------------------------------------------------------------

## 1. Verify LAN Connectivity

From the Linux server, test whether the worker is reachable:

``` bash
ping <WORKER_IP>
```

A healthy result should show received packets with little or no packet
loss.

Connect to Termux over SSH:

``` bash
ssh -p 8022 <WORKER_USER>@<WORKER_IP>
```

On Termux, the SSH daemon can be started with:

``` bash
sshd
```

Check the Termux username:

``` bash
whoami
```

If necessary, set or change the Termux password:

``` bash
passwd
```

------------------------------------------------------------------------

## 2. Create a Convenient SSH Alias

On the Linux server:

``` bash
nano ~/.ssh/config
```

Add:

``` text
Host samsung
    HostName <WORKER_IP>
    User <WORKER_USER>
    Port 8022
```

Then connect using:

``` bash
ssh samsung
```

Instead of:

``` bash
ssh -p 8022 <WORKER_USER>@<WORKER_IP>
```

------------------------------------------------------------------------

## 3. Configure SSH Key Authentication

Generate an Ed25519 key on the Linux server:

``` bash
ssh-keygen -t ed25519
```

Accept the default location unless you have a reason to use another key.

Copy the public key to the worker:

``` bash
ssh-copy-id samsung
```

Test passwordless login:

``` bash
ssh samsung
```

Test remote command execution:

``` bash
ssh samsung 'echo "Hello from $(whoami)@$(hostname)"'
```

Check the worker architecture remotely:

``` bash
ssh samsung 'uname -m'
```

An ARM64 Android device will commonly report:

``` text
aarch64
```

------------------------------------------------------------------------

## 4. Inspect Both Nodes

### Linux server

Check its addresses:

``` bash
hostname -I
```

Find the default network interface:

``` bash
ip route
```

Inspect that interface:

``` bash
ip link show <INTERFACE>
```

For example:

``` bash
ip link show wlo1
```

The `192.168.x.x` address is normally the LAN address. Addresses such as
`172.17.0.1` and `172.18.0.1` are commonly Docker bridge networks and
should not be confused with the host's LAN address.

### Android/Termux worker

Inspect the platform:

``` bash
uname -a
nproc
free -h
df -h $HOME
python --version
id
```

Optional Docker/cgroup capability checks:

``` bash
docker --version
cat /proc/cgroups
ls -l /sys/fs/cgroup
which su
```

On an unrooted Android/Termux installation, cgroup access may be denied.
In that case, do not force Docker or Docker Swarm onto the phone. Use it
as a native worker instead.

------------------------------------------------------------------------

## 5. Reserve Stable DHCP Addresses

Do this in the router administration interface rather than hard-coding
static addresses on each device.

Look for a setting named:

-   DHCP Reservation
-   Address Reservation
-   Static DHCP
-   Static Lease
-   Reserved IP Address

Reserve one LAN address for each node based on its MAC address:

``` text
Linux server:
MAC: <SERVER_MAC>
IP:  <SERVER_IP>

Android worker:
MAC: <WORKER_MAC>
IP:  <WORKER_IP>
```

Keep the router's DHCP server enabled.

On Android, use a stable MAC mode for the home network if your
device/router requires it for reliable reservations.

Do not publish real MAC addresses, public IPs, router credentials, or
passwords in a public repository.

------------------------------------------------------------------------

## 6. Verify Docker on the Primary Server

``` bash
docker --version
docker info
docker ps
```

Test Docker if needed:

``` bash
docker run --rm hello-world
```

If your account does not have Docker permissions, diagnose that
separately rather than permanently using root for every operation.

------------------------------------------------------------------------

## 7. Create a Docker Network

Create a dedicated bridge network for the project:

``` bash
docker network create distributed-net
```

Verify it:

``` bash
docker network ls
```

You should see:

``` text
distributed-net    bridge
```

Containers attached to this network can communicate using Docker's
internal DNS.

------------------------------------------------------------------------

## 8. Optional: Run Portainer

Portainer provides a web UI for inspecting containers, images, volumes,
networks, and logs.

Example:

``` bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Open:

``` text
https://<SERVER_IP>:9443
```

A self-signed certificate warning can be expected for a private LAN
installation.

------------------------------------------------------------------------

## 9. Optional: Run Nginx

A basic Nginx container can be exposed on a non-privileged test port:

``` bash
docker run -d \
  --name nginx \
  --restart unless-stopped \
  -p 8080:80 \
  nginx:alpine
```

Test from another machine:

``` bash
curl http://<SERVER_IP>:8080
```

Later, Nginx can become the reverse proxy for frontend and backend
services.

------------------------------------------------------------------------

## 10. Create a Simple HTTP Worker

Before introducing a queue, test application-level communication between
the machines.

On Termux:

``` bash
mkdir -p ~/worker
cd ~/worker
python -m venv venv
source venv/bin/activate
```

Create:

``` bash
nano worker.py
```

Example standard-library HTTP worker:

``` python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json
import socket

class WorkerHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/health":
            response = {
                "node": "android-worker",
                "hostname": socket.gethostname(),
                "status": "healthy"
            }
        else:
            response = {
                "node": "android-worker",
                "status": "online"
            }

        data = json.dumps(response).encode()

        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)

server = HTTPServer(("0.0.0.0", 8000), WorkerHandler)

print("Worker listening on port 8000...")
server.serve_forever()
```

Run:

``` bash
python worker.py
```

From the Linux server:

``` bash
curl http://<WORKER_IP>:8000/health
```

This verifies that a service running on one physical node can be
consumed by another.

------------------------------------------------------------------------

## 11. Deploy Redis on the Linux Server

Generate a strong password:

``` bash
openssl rand -base64 32
```

Do **not** commit the generated value.

Temporarily export it:

``` bash
export REDIS_PASSWORD='<REDIS_PASSWORD>'
```

Start Redis on the dedicated Docker network and bind it to the server's
LAN address:

``` bash
docker run -d \
  --name redis \
  --network distributed-net \
  --restart unless-stopped \
  -p <SERVER_IP>:6379:6379 \
  redis:alpine \
  redis-server --requirepass "$REDIS_PASSWORD"
```

Verify the container:

``` bash
docker ps
```

Test Redis locally:

``` bash
docker exec redis redis-cli -a "$REDIS_PASSWORD" ping
```

Expected:

``` text
PONG
```

For a more permanent deployment, store secrets outside source control
and consider Redis ACLs/firewall rules instead of relying only on
`requirepass`.

------------------------------------------------------------------------

## 12. Connect the Android Worker to Redis

SSH into the worker:

``` bash
ssh samsung
```

Activate the Python environment:

``` bash
cd ~/worker
source venv/bin/activate
```

Install the Redis Python client:

``` bash
pip install redis
```

Temporarily set the Redis credential:

``` bash
export REDIS_PASSWORD='<REDIS_PASSWORD>'
```

Test connectivity:

``` bash
python - <<'PY'
import os
import redis

r = redis.Redis(
    host="<SERVER_IP>",
    port=6379,
    password=os.environ["REDIS_PASSWORD"],
    decode_responses=True
)

print(r.ping())
PY
```

Expected:

``` text
True
```

At this point the Android worker can communicate directly with Redis
running inside Docker on the Linux server.

------------------------------------------------------------------------

## 13. Build the Queue Worker

Create or replace `~/worker/worker.py`:

``` python
import redis
import json
import os
import time
import hashlib
import socket

REDIS_HOST = "<SERVER_IP>"
REDIS_PORT = 6379
NODE_NAME = "android-worker"

password = os.environ.get("REDIS_PASSWORD")

if not password:
    raise RuntimeError("REDIS_PASSWORD is not set")

r = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    password=password,
    decode_responses=True
)

print(f"{NODE_NAME} starting...")
print(f"Hostname: {socket.gethostname()}")
print(f"Redis: {REDIS_HOST}:{REDIS_PORT}")

r.ping()

print("Connected to Redis.")
print("Waiting for jobs...")

while True:
    _, message = r.brpop("jobs")

    try:
        job = json.loads(message)

        job_id = job["id"]
        task = job["task"]
        value = job.get("value")

        print(f"\nReceived job {job_id}")
        print(f"Task: {task}")

        start = time.time()

        if task == "square":
            result = value * value

        elif task == "hash":
            result = hashlib.sha256(
                str(value).encode()
            ).hexdigest()

        else:
            raise ValueError(f"Unknown task: {task}")

        duration = time.time() - start

        response = {
            "id": job_id,
            "worker": NODE_NAME,
            "task": task,
            "result": result,
            "execution_seconds": duration
        }

        r.set(
            f"result:{job_id}",
            json.dumps(response),
            ex=3600
        )

        print(f"Completed job {job_id}")
        print(f"Result: {result}")

    except Exception as e:
        print(f"Job failed: {e}")
```

Start it:

``` bash
python worker.py
```

The worker blocks on the Redis `jobs` queue until work arrives.

------------------------------------------------------------------------

## 14. Submit a Distributed Job

On the Linux server, make sure the Redis password exists in the shell:

``` bash
export REDIS_PASSWORD='<REDIS_PASSWORD>'
```

Submit a job:

``` bash
docker exec redis redis-cli -a "$REDIS_PASSWORD" LPUSH jobs \
'{"id":"job-001","task":"square","value":25}'
```

The worker should receive and execute it.

Retrieve the result:

``` bash
docker exec redis redis-cli -a "$REDIS_PASSWORD" GET result:job-001
```

Example result:

``` json
{
  "id": "job-001",
  "worker": "android-worker",
  "task": "square",
  "result": 625,
  "execution_seconds": 0.000001
}
```

The computation happened on a different physical machine from the queue.

------------------------------------------------------------------------

## How the Distributed Queue Works

``` text
                 Primary Linux Server
                 +-------------------+
                 |     Producer      |
                 +---------+---------+
                           |
                         LPUSH
                           |
                           v
                 +-------------------+
                 |       Redis       |
                 |    jobs queue     |
                 +---------+---------+
                           |
                         BRPOP
                           |
                           v
                 +-------------------+
                 |  Android Worker   |
                 |                   |
                 | receive -> run    |
                 |        -> result  |
                 +---------+---------+
                           |
                           | SET result:<id>
                           v
                 +-------------------+
                 |       Redis       |
                 +---------+---------+
                           |
                           v
                       Producer
```

The producer does not have to call the Android worker directly. It
publishes a job to the queue. An available worker consumes the job.

This design can later support multiple workers:

``` text
                         Redis
                       jobs queue
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
          Worker 1      Worker 2      Worker 3
```

------------------------------------------------------------------------

## 15. Frontend + Backend Deployment Pipeline

A natural extension is to turn the Linux server into a deployment target
for web applications.

Recommended repository layout:

``` text
myapp/
├── frontend/
│   ├── src/
│   ├── package.json
│   └── Dockerfile
├── backend/
│   ├── Controllers/
│   ├── Program.cs
│   ├── MyApp.csproj
│   └── Dockerfile
├── docker-compose.yml
└── .github/
    └── workflows/
        └── deploy.yml
```

Conceptual pipeline:

``` text
git push
   |
   v
GitHub Actions
   |
   +--> frontend tests/build
   |
   +--> backend tests/build
   |
   +--> Docker image builds
   |
   v
Container Registry
   |
   v
Primary Linux Server
   |
   +--> docker compose pull
   |
   +--> docker compose up -d
   |
   v
Nginx -> Frontend / Backend
```

A production-style flow should use immutable image tags such as Git
commit SHAs rather than relying exclusively on `latest`.

------------------------------------------------------------------------

## 16. Optional AWS Hybrid-Cloud Extension

The homelab can later be connected to AWS without replacing the
on-premises machines.

Possible progression:

``` text
GitHub
   |
   v
CI/CD
   |
   v
Amazon ECR
   |
   +------------------+
   |                  |
   v                  v
On-Prem Server      AWS Compute
   |                  |
Docker             ECS / EC2
   |
Android Worker
```

Useful AWS integrations include:

-   Amazon ECR for Docker images
-   Amazon S3 for selected backups/artifacts
-   Amazon CloudWatch for logs and metrics
-   AWS Systems Manager hybrid/multicloud managed nodes
-   Route 53 for DNS
-   VPC networking
-   Site-to-Site VPN where the network equipment and use case support it
-   ECS/EC2 for cloud compute
-   RDS for managed databases

Start small. ECR is a useful first cloud integration because it adds
IAM, registry, and deployment experience without requiring an
always-running EC2 instance.

------------------------------------------------------------------------

## Security Notes

This project is intended primarily for a trusted home LAN.

Never commit:

``` text
.env
*.pem
*.key
id_ed25519
id_rsa
credentials
secrets
```

Example `.gitignore`:

``` gitignore
.env
.env.*
*.pem
*.key
id_ed25519
id_ed25519.pub
credentials/
secrets/
```

Additional recommendations:

-   Never publish router administrator credentials.
-   Never publish Redis passwords.
-   Never publish private SSH keys.
-   Do not expose Redis port `6379` directly to the public Internet.
-   Do not expose Portainer directly to the public Internet without
    appropriate protection.
-   Prefer SSH keys over password authentication.
-   Use firewall rules to limit service access.
-   Keep Docker images and operating systems patched.
-   Keep databases on private/container networks unless external access
    is explicitly required.
-   Do not root an Android device solely to force it into being a Docker
    node.

------------------------------------------------------------------------

## Troubleshooting

### `ssh: Could not resolve hostname samsung`

Make sure `~/.ssh/config` contains the alias:

``` text
Host samsung
    HostName <WORKER_IP>
    User <WORKER_USER>
    Port 8022
```

### SSH still asks for a password

Copy the key:

``` bash
ssh-copy-id samsung
```

Then test:

``` bash
ssh samsung
```

### Termux SSH is not reachable

Start the daemon:

``` bash
sshd
```

Confirm the phone is still using `<WORKER_IP>`.

### Redis worker cannot connect

From the worker, verify:

``` bash
ping <SERVER_IP>
```

Then check Redis on the server:

``` bash
docker ps
docker exec redis redis-cli -a "$REDIS_PASSWORD" ping
```

Verify that Redis is bound to the expected LAN address and that local
firewall rules permit the connection.

### Docker addresses look confusing

`hostname -I` may show several addresses:

``` text
<SERVER_IP> 172.17.0.1 172.18.0.1
```

The private LAN address is the one assigned by the home router. The
`172.x.x.x` addresses are commonly Docker bridges.

### FastAPI/Pydantic fails to install on Termux

Some combinations of Android, ARM64, and newer Python versions may not
have compatible prebuilt wheels and may attempt native/Rust compilation.

For a lightweight networking experiment, Python's standard-library
`http.server` avoids that dependency. The Redis Python client can then
be used for queue communication.

------------------------------------------------------------------------

## Future Improvements

-   Run the worker persistently using Termux:Boot or an appropriate
    process supervisor.
-   Build a REST API on the Linux server for job submission/status.
-   Add generated UUID job IDs.
-   Add failed-job and retry queues.
-   Add worker heartbeats.
-   Add worker registration/discovery.
-   Add multiple worker nodes.
-   Add Prometheus/Grafana monitoring.
-   Add structured application logs.
-   Add Docker Compose for infrastructure-as-code.
-   Add CI/CD through GitHub Actions.
-   Add container image scanning.
-   Add automated health checks and rollback.
-   Push images to Amazon ECR.
-   Send selected metrics/logs to CloudWatch.
-   Add S3 backup jobs.
-   Experiment with a private AWS VPC/hybrid networking lab.

------------------------------------------------------------------------

## Project Milestones

``` text
[x] Connect two heterogeneous devices over a LAN
[x] Configure SSH aliases
[x] Configure SSH public-key authentication
[x] Reserve stable DHCP addresses
[x] Verify remote command execution
[x] Run Docker on the primary Linux server
[x] Create an isolated Docker bridge network
[x] Run Portainer/Nginx
[x] Expose a service from the Android worker
[x] Run authenticated Redis
[x] Connect the worker to Redis
[x] Build a producer/consumer job queue
[x] Execute computation on a remote physical worker
[ ] Persistent worker
[ ] REST job API
[ ] CI/CD
[ ] Monitoring
[ ] AWS hybrid-cloud integration
```

## Why This Is a Networking Project

Although the final demo includes distributed computation, the foundation
is networking:

1.  Devices discover and reach each other through a private IPv4 LAN.
2.  DHCP reservations provide stable endpoint addressing.
3.  SSH provides authenticated remote administration.
4.  TCP ports expose specific application services.
5.  Docker bridge networks isolate container traffic.
6.  Host port publishing connects Docker services to the physical LAN.
7.  Redis acts as a network-accessible message broker between different
    operating environments.
8.  The design separates infrastructure networking from
    application-level workload distribution.

It is a compact DIY lab for understanding how addressing, routing,
ports, service discovery, authentication, containers, queues, and
distributed applications fit together.

## License

Use this project as a personal learning lab, portfolio project, or
starting point for your own homelab experiments.
