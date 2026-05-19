# Chapter 3: Project Structure & Dockerfile

In this chapter, you will create the project directory structure and write a multi-stage Dockerfile for your NextJS application.

---

## Step 1: Create a NextJS Application (If You Do Not Have One)

If you already have a NextJS project on GitHub, clone it onto your server and skip to Step 2.

To create a new NextJS app from scratch, run this on your **local machine** (not the server):

```bash
npx create-next-app@latest my-project
cd my-project
```

Push the project to a new GitHub repository:

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git push -u origin main
```

Then on your server, clone the repository:

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git ~/my-project
cd ~/my-project
```

Replace `YOUR_USERNAME` and `YOUR_REPO` with your GitHub details.

---

## Step 2: Create the Project Directory Layout

Inside the project root (`~/my-project`), create the required directories:

```bash
mkdir -p nginx
```

Your project should look like this:

```
my-project/
├── app/                    # NextJS pages and components (already exists if using app router)
├── pages/                  # NextJS pages (if using pages router)
├── public/                 # Static assets
├── nginx/
│   └── default.conf        # (will create next)
├── package.json            # Already exists from create-next-app
├── next.config.js          # Already exists
├── Dockerfile              # (will create next)
├── docker-compose.yml      # (will create in chapter 4)
├── .env                    # (will create in chapter 4)
└── Jenkinsfile             # (will create in chapter 7)
```

---

## Step 3: Create the Dockerfile

Create a file called `Dockerfile` in the project root:

```bash
nano Dockerfile
```

Paste the following content:

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

RUN npm run build

FROM node:20-alpine

WORKDIR /app

COPY --from=builder /app .

EXPOSE 3000

CMD ["npm", "start"]
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

---

## Understanding the Multi-Stage Build

This Dockerfile uses **multi-stage builds** — a pattern that produces smaller, more secure images.

### Stage 1: `builder`

```
FROM node:20-alpine AS builder
```

- Installs all dependencies (`npm install`)
- Builds the application (`npm run build`)
- Contains compilers and dev dependencies that the final image does not need

### Stage 2: production runtime

```
FROM node:20-alpine
```

- Starts from a fresh, minimal Node.js image
- Copies only the built output from `builder`
- Does **not** include `node_modules` from the build stage (only production dependencies needed)

### Why This Matters

| Single-stage build | Multi-stage build |
|---|---|
| Image size: ~1.2 GB | Image size: ~250 MB |
| Contains build tools (compilers, TypeScript, etc.) | Contains only runtime files |
| Larger attack surface | Smaller attack surface |
| Slower deployment | Faster deployment |

> **Pro tip:** You can add `--omit=dev` or use `npm ci --only=production` in the second stage to install only production dependencies if your app needs them at runtime. However, NextJS's standalone output mode (`output: "standalone"` in `next.config.js`) already bundles most of what is needed.

---

## Step 4: Optimize the Dockerfile (Optional but Recommended)

If your NextJS project uses the standalone output mode, update `next.config.js` on your local machine first:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: "standalone",
};
module.exports = nextConfig;
```

Then use this optimized Dockerfile:

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

RUN npm run build

FROM node:20-alpine

WORKDIR /app

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3000

CMD ["node", "server.js"]
```

This uses NextJS's `output: "standalone"` feature, which traces only the files your app needs at runtime, producing an even smaller image.

---

## Verification Checklist

Before moving on, confirm:

- [x] The project directory exists at `~/my-project`
- [x] The `nginx/` subdirectory exists
- [x] `Dockerfile` exists with the multi-stage build content
- [x] `package.json` exists (from NextJS setup or clone)

---

**Next:** [Chapter 4 — Docker Compose & Networking](04-docker-compose-and-networking.md)
