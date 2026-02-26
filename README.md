# Deploying a Production Ready n8n Stack with Pulumi

Deploying n8n manually is tedious. If your server crashes, you have to do it all over again. Pulumi changes this by letting you define your entire setup in a single TypeScript file.

This guide sets up a fully automated, secure n8n environment.

## What We Are Building

This stack includes everything you need for a stable environment:

* **Caddy:** Automatically handles your domain routing and provides free SSL certificates (HTTPS).
* **n8n Main:** The UI and webhook handler.
* **Postgres:** The database for workflows and credentials.
* **Redis:** The message queue.
* **n8n Runner:** A dedicated sidecar container for heavy code execution.
* **Docker Volumes:** Permanent storage on your VPS disk so you never lose data.

## Step 1: Configure Your Domain

Before writing any code, you need to point your domain name to your VPS.

* Go to your domain registrar (like Namecheap, GoDaddy, or Cloudflare).
* Open your DNS settings.
* Create a new **A Record**.
* Set the Host to `@` (for a root domain) or a subdomain name like `n8n`.
* Set the Value to your exact VPS IP address.
* Save the record and wait a few minutes for it to propagate.

## Step 2: The SSH Key Checkpoint

Pulumi needs to talk to your VPS over SSH.

If your SSH key is in the default folder (`~/.ssh/` on Mac/Linux or `C:\Users\YourName\.ssh\` on Windows), skip to Step 3.

If you use PuTTY or store your key on a separate drive, follow these steps:

1. Open PuTTYgen, load your `.ppk` key, click **Conversions**, and select **Export OpenSSH key**. Save this to your preferred drive.
2. Go to your computer's default `~/.ssh` or `C:\Users\YourName\.ssh` folder.
3. Create a file named exactly `config` (no file extension).
4. Open the file and add this text, replacing it with your actual details:

```text
Host n8n-server
    HostName YOUR_VPS_IP
    User root
    IdentityFile D:\path\to\your\converted_openssh_key

```

## Step 3: The Pulumi Code

Create a new Pulumi TypeScript project and replace the contents of `index.ts` with this code.

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as docker from "@pulumi/docker";

const config = new pulumi.Config();
const dbPassword = config.requireSecret("dbPassword");
const encryptionKey = config.requireSecret("encryptionKey");
const runnerToken = config.requireSecret("runnerToken");
const domainName = config.require("domainName");

// Persistent Storage
const n8nData = new docker.Volume("n8n-data", { name: "n8n_data" });
const dbData = new docker.Volume("db-data", { name: "postgres_data" });
const caddyData = new docker.Volume("caddy-data", { name: "caddy_data" });

const network = new docker.Network("n8n-network", { name: "n8n_network" });

// Reverse Proxy (Caddy for automatic HTTPS)
const caddy = new docker.Container("caddy", {
    image: "caddy:2-alpine",
    name: "caddy",
    restart: "always",
    ports: [
        { internal: 80, external: 80 },
        { internal: 443, external: 443 }
    ],
    volumes: [{ volumeName: caddyData.name, containerPath: "/data" }],
    // Caddy will automatically fetch SSL and route traffic to the main n8n container
    command: ["caddy", "reverse-proxy", "--from", domainName, "--to", "http://n8n-main:5678"],
    networksAdvanced: [{ name: network.name }],
});

// Database
const postgres = new docker.Container("postgres", {
    image: "postgres:16-alpine",
    name: "n8n-postgres",
    restart: "always",
    volumes: [{ volumeName: dbData.name, containerPath: "/var/lib/postgresql/data" }],
    envs: [
        `POSTGRES_USER=postgres`,
        pulumi.interpolate`POSTGRES_PASSWORD=${dbPassword}`,
        `POSTGRES_DB=n8n`,
    ],
    networksAdvanced: [{ name: network.name }],
});

// Queue
const redis = new docker.Container("redis", {
    image: "redis:7-alpine",
    name: "n8n-redis",
    restart: "always",
    networksAdvanced: [{ name: network.name }],
});

// Main Instance
const n8nMain = new docker.Container("n8n-main", {
    image: "n8nio/n8n:latest",
    name: "n8n-main",
    restart: "always",
    // We removed exposed ports here because Caddy handles all external access
    volumes: [{ volumeName: n8nData.name, containerPath: "/home/node/.n8n" }],
    envs: [
        pulumi.interpolate`WEBHOOK_URL=https://${domainName}`,
        "N8N_PROTOCOL=https",
        pulumi.interpolate`N8N_HOST=${domainName}`,
        "GENERIC_TIMEZONE=Asia/Kolkata",
        "TZ=Asia/Kolkata",
        pulumi.interpolate`N8N_ENCRYPTION_KEY=${encryptionKey}`,
        "DB_TYPE=postgresdb",
        "DB_POSTGRESDB_HOST=n8n-postgres",
        "DB_POSTGRESDB_USER=postgres",
        pulumi.interpolate`DB_POSTGRESDB_PASSWORD=${dbPassword}`,
        "EXECUTIONS_MODE=queue",
        "QUEUE_BULL_REDIS_HOST=n8n-redis",
        "N8N_RUNNERS_ENABLED=true",
        "N8N_RUNNERS_MODE=external",
        pulumi.interpolate`N8N_RUNNERS_AUTH_TOKEN=${runnerToken}`,
        "N8N_RUNNERS_BROKER_LISTEN_ADDRESS=0.0.0.0",
    ],
    networksAdvanced: [{ name: network.name }],
}, { dependsOn: [postgres, redis] });

// Runner Sidecar
const n8nRunner = new docker.Container("n8n-runner", {
    image: "n8nio/runners:latest",
    name: "n8n-runner",
    restart: "always",
    envs: [
        pulumi.interpolate`N8N_RUNNERS_AUTH_TOKEN=${runnerToken}`,
        "N8N_RUNNERS_TASK_BROKER_URI=http://n8n-main:5679", 
        "N8N_RUNNERS_AUTO_SHUTDOWN_TIMEOUT=15",
    ],
    networksAdvanced: [{ name: network.name }],
}, { dependsOn: [n8nMain] });

```

## Step 4: Deployment

Run these commands in your terminal to deploy your stack.

**1. Point Pulumi to your server:**

* If using the default SSH location: `pulumi config set docker:host ssh://root@YOUR_VPS_IP`
* If using the custom config file from Step 2: `pulumi config set docker:host ssh://n8n-server`

**2. Set your variables:**

```bash
pulumi config set domainName "n8n.yourdomain.com"
pulumi config set --secret dbPassword "YourDatabasePassword123"
pulumi config set --secret encryptionKey "YourLongRandomStringXYZ"
pulumi config set --secret runnerToken "YourRandomStringForRunners"

```

**3. Launch:**

```bash
pulumi up

```

## Step 5: Backup and Restore Strategy

Because your data lives in Docker Volumes, it is completely separated from the application code. Run these commands directly on your VPS terminal.

**To Backup Everything:**

```bash
docker run --rm -v n8n_data:/n8n -v postgres_data:/db -v $(pwd):/backup alpine tar czf /backup/n8n_full_backup.tar.gz /n8n /db

```

**To Restore (If your server dies):**

1. Run `pulumi up` on your new server to create the empty volumes.
2. Copy your `n8n_full_backup.tar.gz` file to the new server.
3. Run this command to inject your data:

```bash
docker run --rm -v n8n_data:/n8n -v postgres_data:/db -v $(pwd):/backup alpine sh -c "tar xzf /backup/n8n_full_backup.tar.gz -C /"

```

## Troubleshooting Common Issues

**1. Permission Denied Errors After Restore**

* Because n8n runs as a secure, non-root user (UID 1000), restoring backup files as the root user can cause permission conflicts.
* If your n8n container crashes on startup with an `EACCES: permission denied` error, run this command on your VPS to fix the volume ownership:

```bash
docker run --rm -v n8n_data:/n8n alpine chown -R 1000:1000 /n8n

```

**2. Pulumi Cannot Connect via SSH**

* If the `pulumi up` command hangs or throws a connection error, your SSH configuration might be slightly off.
* Always test your connection manually first. Open your terminal and type `ssh root@YOUR_VPS_IP` or `ssh n8n-server` to see if it connects.
* If you are on Mac or Linux and get a "permissions are too open" warning on your key file, run `chmod 600 /path/to/your/key` to secure it.

**3. n8n Runners Are Offline**

* If your n8n UI shows that no runners are connected, the sidecar container cannot talk to the main broker.
* Ensure the `runnerToken` secret you set in Pulumi is exactly the same for both the main and runner containers.
* Make sure the `N8N_RUNNERS_TASK_BROKER_URI` in the runner setup is pointing specifically to `http://n8n-main:5679`. It must use `http` (not https) and reference the exact Docker container name.

**4. SSL Certificate Not Working**

* If your site does not load over HTTPS, Caddy might not be able to verify your domain.
* Check that your DNS A Record is correctly pointing to your VPS IP.
* Ensure you do not have any firewall rules blocking ports 80 and 443 on your VPS. Caddy needs port 80 open to verify the Let's Encrypt certificate.
