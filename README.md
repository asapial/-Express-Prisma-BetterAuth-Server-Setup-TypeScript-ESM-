# ⚡ Express + Prisma + BetterAuth

> A production-ready backend boilerplate with TypeScript, ESM, PostgreSQL, and authentication — configured and ready to ship.

<div align="center">

![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Express](https://img.shields.io/badge/Express-5.x-000000?style=flat-square&logo=express&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma-ORM-2D3748?style=flat-square&logo=prisma&logoColor=white)
![BetterAuth](https://img.shields.io/badge/BetterAuth-Latest-6366F1?style=flat-square)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-336791?style=flat-square&logo=postgresql&logoColor=white)
![ESM](https://img.shields.io/badge/Module-ESM-F7DF1E?style=flat-square)

</div>

---

## ✦ Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20+ |
| Language | TypeScript (ESM) |
| Framework | Express |
| ORM | Prisma + `@prisma/adapter-pg` |
| Auth | BetterAuth |
| Database | PostgreSQL |
| Bundler | tsup |

---

## 1 · Project Setup

```bash
mkdir my-api && cd my-api
npm init -y
npm install typescript tsx @types/node --save-dev
npx tsc --init
```

---

## 2 · Install Dependencies

```bash
# Production
npm install prisma @prisma/client @prisma/adapter-pg \
  dotenv express cors cookie-parser better-auth

# Development
npm install --save-dev @types/express @types/cors @types/cookie-parser tsup
```

---

## 3 · TypeScript Config

**`tsconfig.json`**
```json
{
  "compilerOptions": {
    "outDir": "./dist",
    "module": "ESNext",
    "target": "ES2023",
    "moduleResolution": "bundler",
    "types": [],
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "strict": true,
    "isolatedModules": true,
    "noUncheckedSideEffectImports": true,
    "moduleDetection": "force",
    "skipLibCheck": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "generated/prisma"]
}
```

Also add to **`package.json`**:
```json
{
  "type": "module"
}
```

---

## 4 · Database (Prisma)

### Initialize

```bash
npx prisma init --db --output ../generated/prisma
```

### Client — `src/lib/prisma.ts`

```typescript
import "dotenv/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../generated/prisma/client";

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL! });
const prisma = new PrismaClient({ adapter });

export { prisma };
```

---

## 5 · Authentication (BetterAuth)

### Environment — `.env`

```env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"

# Generate: openssl rand -base64 32
BETTER_AUTH_SECRET=your_secret_here

BETTER_AUTH_URL=http://localhost:3000
```

### Config — `src/lib/auth.ts`

```typescript
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { prisma } from "./prisma";

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  emailAndPassword: { enabled: true },
  session: {
    cookieCache: { enabled: true, maxAge: 5 * 60 }, // cache for 5 minutes
  },
  advanced: {
    cookiePrefix: "better-auth",
    useSecureCookies: process.env.NODE_ENV === "production",
    crossSubDomainCookies: { enabled: false },
    disableCSRFCheck: true,
    defaultCookieAttributes: {
      sameSite: "none",
      secure: true,
      httpOnly: false,
    },
  },
});
```

> **`cookieCache`** — reduces database round-trips by caching session validation for 5 minutes.  
> **`useSecureCookies`** — automatically enforces HTTPS-only cookies in production.  
> **`sameSite: "none"`** — required when your frontend and backend are on separate domains.

---

## 6 · Schema & Migration

### Organize your schema

```
prisma/
└── schema/
    ├── schema.prisma    ← move here
    └── auth.prisma      ← generated below
```

Update `package.json`:
```json
{
  "prisma": { "schema": "prisma/schema" }
}
```

### Generate auth schema

```bash
npx @better-auth/cli@latest generate \
  --output ./prisma/schema/auth.prisma \
  --config ./src/lib/auth.ts
```

### Run migration

```bash
npx prisma migrate dev
```

---

## 7 · Express Application

### App — `src/app.ts`

```typescript
import express, { Application } from "express";
import cors from "cors";
import cookieParser from "cookie-parser";
import { toNodeHandler } from "better-auth/node";
import { auth } from "./lib/auth";

const app: Application = express();

app.use(cookieParser());
app.use(express.json());

const allowedOrigins = ["http://localhost:3000"];

app.use(
  cors({
    origin: (origin, callback) => {
      if (!origin) return callback(null, true);
      const isAllowed =
        allowedOrigins.includes(origin) ||
        /^https:\/\/.*\.vercel\.app$/.test(origin);
      isAllowed
        ? callback(null, true)
        : callback(new Error(`Origin ${origin} not allowed`));
    },
    credentials: true,
    methods: ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allowedHeaders: ["Content-Type", "Authorization", "Cookie"],
    exposedHeaders: ["Set-Cookie"],
  })
);

// Auth routes
app.all("/api/auth/*", toNodeHandler(auth));

// Health check
app.get("/", (_req, res) => {
  res.status(200).json({
    success: true,
    message: "Server is running",
    version: "1.0.0",
    environment: process.env.NODE_ENV ?? "development",
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  });
});

export default app;
```

### Server — `src/server.ts`

```typescript
import app from "./app";
import { prisma } from "./lib/prisma";

const PORT = process.env.PORT || 5000;

async function main() {
  try {
    await prisma.$connect();
    console.log("✓ Database connected");

    app.listen(PORT, () => {
      console.log(`✓ Server running → http://localhost:${PORT}`);
    });
  } catch (error) {
    console.error("✗ Startup failed:", error);
    await prisma.$disconnect();
    process.exit(1);
  }
}

main();
```

---

## 8 · Scripts

**`package.json`**
```json
{
  "scripts": {
    "dev":         "tsx watch src/server.ts",
    "build":       "prisma generate && tsup src/server.ts --format esm --platform node --target node20 --outDir api --external pg-native",
    "migrate":     "prisma migrate dev",
    "generate":    "prisma generate",
    "postinstall": "prisma generate"
  }
}
```

### Start development server

```bash
npm run dev
```

---

## Project Structure

```
.
├── prisma/
│   └── schema/
│       ├── schema.prisma
│       └── auth.prisma
├── src/
│   ├── lib/
│   │   ├── auth.ts
│   │   └── prisma.ts
│   ├── app.ts
│   └── server.ts
├── generated/
│   └── prisma/
├── .env
├── package.json
└── tsconfig.json
```

---

## Quick Start

```bash
# 1. Clone & install
git clone <your-repo> && cd <your-repo>
npm install

# 2. Configure environment
cp .env.example .env
# Edit DATABASE_URL and BETTER_AUTH_SECRET

# 3. Set up database
npm run migrate

# 4. Start dev server
npm run dev
```

---

<div align="center">
  <sub>Built with TypeScript · ESM · Express · Prisma · BetterAuth</sub>
</div>
