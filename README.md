
# 🧩 **Task-14: Image Size Reduction of Docker Images**

## **1. Introduction**

Docker images are the backbone of containerized applications. Each image contains all the dependencies, libraries, and binaries required to run the application.  
However, **larger images** increase:

- Build time
    
- Deployment time
    
- Network transfer time
    
- Storage costs
    

Therefore, optimizing and **reducing the Docker image size** is a key practice for efficient, fast, and cost-effective deployments.

---

## **2. Why Image Size Matters**

|Problem|Impact|
|---|---|
|**Longer image build times**|Slower CI/CD pipelines|
|**Slower deployments**|Increased downtime during rolling or blue/green updates|
|**More bandwidth usage**|Longer image pulls across clusters or nodes|
|**Higher storage costs**|Increased cost on AWS ECR, Docker Hub, etc.|
|**Security risks**|Larger images may contain unnecessary packages or vulnerabilities|

---

## **3. Techniques to Reduce Docker Image Size**

### **A. Use Minimal Base Images**

- Instead of using a full OS like `ubuntu` or `debian`, use lightweight images such as:
    
    `FROM alpine:latest`
    
- Alpine images are typically **5 MB** compared to Ubuntu images which are **~70–100 MB**.

# 🧩 Practical: Image Size Reduction using Minimal Base Image (Node.js)

## 🎯 **Goal**

Build two Docker images for a simple Node.js app:

1. Using **`node:20`** (default base image)
    
2. Using **`node:20-alpine`** (minimal base image)
    
Then, compare the image sizes and build times.

## 🧰 **1. Prerequisites**

You need:

- Docker installed
    
- Basic Node.js app (example below)
    
- Internet connection (for pulling images)

## 🧱 **3. Create Two Dockerfiles**

### **A. Dockerfile 1 — Using Full Node Image**

Save as `Dockerfile.full`
```
# Stage 1: Full Node image
FROM node:20

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```
### **B. Dockerfile 2 — Using Minimal Alpine Image**

Save as `Dockerfile.alpine`
```
# Stage 2: Lightweight Alpine image
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install --only=production
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```
## ⚙️ **4. Build Both Images**

Open a terminal inside the folder and run:
```
# Build full image
docker build -t node-full -f Dockerfile.full .

# Build minimal image
docker build -t node-alpine -f Dockerfile.alpine .
```
## 📏 **5. Compare Image Sizes**

After building, check both image sizes:
```
docker images
```
**Example Output:**

<img width="837" height="161" alt="Screenshot 2025-10-19 191509" src="https://github.com/user-attachments/assets/4cfc2237-fbf2-4e8a-b82f-2693e21a4463" />

✅ **Result:**

- Full image = **1.10 GB**
- Alpine image = **134 MB**  
    🎯 **~85% size reduction!**

## 📊 **7. Observations**

| Parameter                          | Full Node | Node Alpine |
| ---------------------------------- | --------- | ----------- |
| Image Size                         | ~1.10 GB  | ~134 MB     |
| Build Time                         | ~40 sec   | ~12 sec     |
| Container Start Time               | 2–3 sec   | <1 sec      |
| Deployment Pull Time (ECR/Fargate) | Slow      | Fast        |
| Storage Cost (ECR)                 | High      | Low         |

---
# 🧩 Practical: Image Size Reduction using Multi-Stage Builds (Node.js)

## 🎯 **Goal**

We’ll build two Docker images for the same Node.js app:

1. **Without multi-stage build** (standard build)
    
2. **With multi-stage build** (optimized)
    

Then, we’ll compare:

- Image size
    
- Build performance
    
- Deployment efficiency

## ⚡ **2. Dockerfile With Multi-Stage Build**

**`Dockerfile.multistage`**

```
# Stage 1: Builder
FROM node:20 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Runtime
FROM node:20-alpine AS runner

WORKDIR /app

# Copy only built output and production dependencies
COPY --from=builder /app/package*.json ./
RUN npm install --only=production

COPY --from=builder /app/app.js ./

EXPOSE 3000
CMD ["npm", "start"]
```
### ✅ Benefits of Multi-Stage Build:

- The **first stage** handles compilation/build (e.g., TypeScript, React).
    
- The **final stage** contains only the runtime files needed to execute the app.
    
- All intermediate layers (build tools, caches) are excluded.

## ⚙️ **6. Build Both Images**

Run these commands:
```
docker build -t node-multistage -f Dockerfile.multistage .
```
## 📏 **7. Compare Image Sizes**

Run:
```
docker images
```
<img width="819" height="75" alt="Screenshot 2025-10-19 192625" src="https://github.com/user-attachments/assets/3d64327a-0e21-45fc-a586-402233b30e27" />

✅ **Result:**  
**~85% size reduction** due to removal of build dependencies.

---

## 🧩 **Technique C: Remove Unnecessary Files**

### 🎯 **Goal**

Reduce image size and attack surface by **removing build-time and unused files** (e.g., test folders, `.git`, `README.md`, logs, caches, etc.) before packaging the final runtime image.

This step is applied either:

- At the **end of a single-stage Dockerfile**, or
    
- **Inside a multi-stage build**, so only essential files go into the final stage.

## 🛠️ **Example — Strapi Multi-Stage Dockerfile with Technique C**

Here’s a real-world optimized `Dockerfile` using the **“remove unnecessary files”** approach:
```
# Stage 1: Install dependencies
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm run build
COPY . .

# Remove dev/test files
RUN rm -rf tests .git .github Dockerfile* README.md

# Stage 2: Runtime
FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app ./
EXPOSE 1337
CMD ["npm", "start"]
```
## 💡 **Why It Helps in Deployment**

- **Faster deployments** → smaller images push/pull quickly.
    
- **Reduced ECR/S3 storage cost** → smaller image sizes = less storage.
    
- **Less bandwidth use** → faster CI/CD pipeline.
    
- **Smaller attack surface** → removes files not needed in production.

---
## 🧩 **Technique D: Combine and Minimize Layers**

### 🎯 **Goal**

Each instruction in a Dockerfile (like `RUN`, `COPY`, `ADD`) creates a **new layer**.  
More layers = more size + slower build and push times.

To optimize:  
✅ Combine multiple `RUN` commands into one.  
✅ Clean caches and temp files within the same `RUN`.  
✅ Use logical grouping to reduce layer count.


### 🧹 **Example of Combining RUN Layers**

❌ **Before (Inefficient):**
```
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean
```

✅ **After (Efficient):**
```
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*`
```
> 👉 Combines 3 layers into 1 and cleans up cache immediately — smaller and faster image.

### 💡 **How This Helps in Deployment**

- **Faster image build** → fewer layers = less caching overhead.
    
- **Reduced ECR and CI/CD cost** → smaller image, less bandwidth.
    
- **Simpler debugging** → consolidated commands = easier maintenance.
    
- **Faster container start time** → less metadata overhead in final image.

---

## 🧩 **Technique E – Use `.dockerignore` File**

### 🎯 **Goal**

Prevent unnecessary files from being **sent to the Docker daemon** during `docker build`.

When you run a build, Docker sends your **entire build context** (the folder contents) to the daemon.  
If you don’t exclude large or irrelevant files, they:

- Increase **build time**
    
- Increase **image size**
    
- Slow down **CI/CD pipeline**

## ⚙️ **Example – `.dockerignore` for Node.js / Strapi**

Create a file named **`.dockerignore`** in the same directory as your `Dockerfile`:
```
# Node and build artifacts
node_modules
npm-debug.log
yarn-error.log
*.log
*.tmp
build/
dist/
.cache/

# Git and version control
.git
.gitignore
.github/

# IDE / editor files
.vscode/
.idea/
.DS_Store

# Docker
Dockerfile*
docker-compose.yml

# OS / environment files
.env
.env.local
.env.development
.env.production

# Tests and docs
tests/
docs/
README.md
```

### 🧠 **How It Works**

When Docker builds your image:

- Only files **not ignored** are included in the “build context”.
    
- This reduces the data sent to the daemon, making builds **faster**.
    
- It also ensures sensitive files (like `.env`) or large ones (`node_modules`) never end up inside the image.

### 💡 **Best Practices**

✅ Always include `.env`, `.git`, `node_modules`, and logs.  
✅ Keep `.dockerignore` updated as your repo grows.  
✅ Treat it like `.gitignore` — review regularly.  
✅ Combine with other techniques (A–D) for **maximum image optimization**.

---
## 🧩 **Technique F – Use Compressed or Distroless Images**

### 🎯 **Goal**

Reduce image size and attack surface by using **lightweight or distroless base images** instead of full OS distributions (like Ubuntu or Debian).

## ⚙️ **2️⃣ Distroless Images (Google’s Secure Minimal Images)**

Distroless images contain **only the application and its runtime** —  
No shell, no package manager, no unnecessary libraries.  
This makes them **ultra-small** and **very secure**.

### Example — Node.js (Distroless):
```
# Stage 1: Builder
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Distroless Runtime
FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app

# Copy only the final app
COPY --from=builder /app ./

EXPOSE 1337
CMD ["build/server.js"]
```
🧠 Note: You must specify the **entry point file** directly since `bash` or `sh` doesn’t exist in distroless images.

## 🛡️ **Security Benefits**

✅ No package manager or shell → attack surface reduced drastically.  
✅ No way to run unauthorized commands inside the container.  
✅ Better compliance and vulnerability scores (CVE-free base).

## 🧠 **Best Practice Combo**

For maximum efficiency in production:

1. Use **multi-stage builds**
    
2. Use **Alpine or Distroless** final stage
    
3. Use **`.dockerignore`**, and
    
4. **Remove unnecessary files + combine layers**
    

👉 Result: A **secure ~50 MB image** that builds fast, deploys instantly, and costs less to store or transfer.

---

## **Conclusion**

Reducing Docker image size is not just a best practice — it’s a cost-saving, security-improving, and performance-enhancing strategy.  
In real-world deployments (e.g., AWS ECS Fargate, Kubernetes, Docker Swarm), it directly translates to **faster deployment times**, **less downtime**, and **lower cloud costs**.
