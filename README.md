Here's a cleaned-up, professional-looking version of your guide suitable for a **README.md** file on GitHub.

```markdown
# Express + Prisma + Better Auth Starter

Modern TypeScript backend template with:

- Express.js (ESM)
- Prisma ORM (PostgreSQL)
- Better Auth (authentication)
- TypeScript strict mode
- CORS + cookie handling

## Project Structure (recommended)

```
hello-prisma/
├── prisma/
│   ├── schema/
│   │   ├── schema.prisma           ← base schema
│   │   └── auth.prisma             ← Better Auth generated models
│   └── migrations/                 ← auto-generated
├── src/
│   ├── lib/
│   │   ├── prisma.ts
│   │   └── auth.ts
│   ├── server.ts                   ← entry point
│   └── app.ts                      ← Express app setup
├── .env
├── package.json
└── tsconfig.json
```

## Setup Instructions

### 1. Initialize Project

```bash
mkdir hello-prisma
cd hello-prisma

npm init -y
npm install typescript tsx @types/node --save-dev
npx tsc --init
```

### 2. Install Core Dependencies

```bash
npm install prisma --save-dev
npm install @prisma/client @prisma/adapter-pg dotenv
npm install express cors cookie-parser
npm install better-auth

# TypeScript types
npm install --save-dev @types/express @types/cors @types/cookie-parser @types/node
```

### 3. Configure TypeScript (ESM + modern strict settings)

Replace content of **`tsconfig.json`** with:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "outDir": "./dist",
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
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

Add to **`package.json`**:

```json
{
  "type": "module"
}
```

### 4. Initialize Prisma

```bash
npx prisma init --datasource-provider postgresql --output ./prisma
```

**Important:** Move the generated `schema.prisma` file:

```bash
mkdir -p prisma/schema
mv prisma/schema.prisma prisma/schema/
```

Update **`prisma/schema.prisma`** later when adding models.

First migration:

```bash
npx prisma migrate dev --name init
npx prisma generate
```

### 5. Create Prisma Client with pg adapter

**`src/lib/prisma.ts`**

```ts
import "dotenv/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../../prisma/client"; // adjust path if needed

const connectionString = process.env.DATABASE_URL!;

const adapter = new PrismaPg({ connectionString });
export const prisma = new PrismaClient({ adapter });
```

### 6. Set up Better Auth

**`src/lib/auth.ts`**

```ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";

import { prisma } from "./prisma";

export const auth = betterAuth({
  database: prismaAdapter(prisma, {
    provider: "postgresql",
  }),

  emailAndPassword: {
    enabled: true,
  },

  session: {
    cookieCache: {
      enabled: true,
      maxAge: 5 * 60, // 5 minutes
    },
  },

  advanced: {
    cookiePrefix: "better-auth",
    useSecureCookies: process.env.NODE_ENV === "production",
    crossSubDomainCookies: {
      enabled: false,
    },
    disableCSRFCheck: true,
    defaultCookieAttributes: {
      sameSite: "none",
      secure: true,
      httpOnly: false,
    },
  },
});
```

### 7. Generate Better Auth schema

```bash
npx @better-auth/cli@latest generate \
  --output ./prisma/schema/auth.prisma \
  --config ./src/lib/auth.ts
```

Then migrate:

```bash
npx prisma migrate dev --name better-auth-init
```

### 8. Create Express Application

**`src/app.ts`**

```ts
import express, { Application } from "express";
import cors from "cors";
import cookieParser from "cookie-parser";
import { toNodeHandler } from "better-auth/node";
import { auth } from "./lib/auth";

const app: Application = express();

app.use(cookieParser());
app.use(express.json());

// ── CORS (must come early) ────────────────────────────────────────
const allowedOrigins = ["http://localhost:3000"].filter(Boolean);

app.use(
  cors({
    origin: (origin, callback) => {
      if (!origin) return callback(null, true);

      const isAllowed =
        allowedOrigins.includes(origin) ||
        /^https:\/\/next-blog-client.*\.vercel\.app$/.test(origin) ||
        /^https:\/\/.*\.vercel\.app$/.test(origin);

      if (isAllowed) {
        callback(null, true);
      } else {
        callback(new Error(`Origin ${origin} not allowed by CORS`));
      }
    },
    credentials: true,
    methods: ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allowedHeaders: ["Content-Type", "Authorization", "Cookie"],
    exposedHeaders: ["Set-Cookie"],
  })
);

// ── Better Auth routes ─────────────────────────────────────────────
app.all("/api/auth/*", toNodeHandler(auth));

// ── Health check ───────────────────────────────────────────────────
app.get("/", (_req, res) => {
  res.status(200).json({
    success: true,
    message: "Server is running successfully",
    service: "Backend API",
    version: "1.0.0",
    environment: process.env.NODE_ENV ?? "development",
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  });
});

export default app;
```

**`src/server.ts`**

```ts
import app from "./app";
import { prisma } from "./lib/prisma";

const PORT = process.env.PORT || 5000;

async function main() {
  try {
    await prisma.$connect();
    console.log("Connected to the database successfully.");

    app.listen(PORT, () => {
      console.log(`Server is running on http://localhost:${PORT}`);
    });
  } catch (error) {
    console.error("An error occurred:", error);
    await prisma.$disconnect();
    process.exit(1);
  }
}

main();
```

### 9. Recommended `package.json` scripts

```json
{
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "migrate": "prisma migrate dev",
    "generate": "prisma generate",
    "postinstall": "prisma generate",
    "build": "prisma generate && tsup src/server.ts --format esm --platform node --target node20 --outDir api --external pg-native"
  }
}
```

### 10. Environment Variables (.env)

```env
DATABASE_URL="postgresql://user:password@localhost:5432/dbname?schema=public"
BETTER_AUTH_SECRET=your-very-long-random-secret-min-32-chars
BETTER_AUTH_URL=http://localhost:5000
PORT=5000
NODE_ENV=development
```

### Quick Start

```bash
# Install dependencies
npm install

# Create & migrate database (first time)
npm run migrate

# Development server with auto-reload
npm run dev
```

Open http://localhost:5000/

Better Auth endpoints will be available under `/api/auth/*`

Good luck with your project! 🚀

```

Feel free to adjust paths, ports, allowed origins, or add sections like **Features**, **Security Notes**, **Deployment** etc. as needed.

Let me know if you'd like to add API documentation examples, authentication flow explanation, or folder structure diagram using mermaid.
