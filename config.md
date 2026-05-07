# ═══════════════════════════════════════════════════════════
# functions/package.json
# ═══════════════════════════════════════════════════════════
{
  "name": "sql-connect-functions",
  "version": "1.0.0",
  "main": "dist/index.js",
  "scripts": {
    "build":     "tsc",
    "dev":       "nodemon --exec ts-node src/index.ts",
    "start":     "node dist/index.js",
    "test":      "vitest run",
    "test:watch": "vitest",
    "lint":      "eslint src/**/*.ts"
  },
  "dependencies": {
    "@google-cloud/cloud-sql-connector": "^1.2.0",
    "@google-cloud/functions-framework": "^3.3.0",
    "@google-cloud/pubsub":              "^4.3.0",
    "express":                           "^4.18.2",
    "firebase-admin":                    "^12.0.0",
    "helmet":                            "^7.1.0",
    "pg":                                "^8.11.3",
    "uuid":                              "^9.0.0",
    "ws":                                "^8.16.0"
  },
  "devDependencies": {
    "@testcontainers/postgresql": "^10.6.0",
    "@types/express":             "^4.17.21",
    "@types/node":                "^20.11.0",
    "@types/pg":                  "^8.10.9",
    "@types/uuid":                "^9.0.7",
    "@types/ws":                  "^8.5.10",
    "nodemon":                    "^3.0.3",
    "testcontainers":             "^10.6.0",
    "ts-node":                    "^10.9.2",
    "typescript":                 "^5.3.3",
    "vitest":                     "^1.2.1"
  }
}

# ═══════════════════════════════════════════════════════════
# functions/tsconfig.json
# ═══════════════════════════════════════════════════════════
{
  "compilerOptions": {
    "target":            "ES2022",
    "module":            "CommonJS",
    "lib":               ["ES2022"],
    "outDir":            "./dist",
    "rootDir":           "./src",
    "strict":            true,
    "esModuleInterop":   true,
    "skipLibCheck":      true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration":       true,
    "declarationMap":    true,
    "sourceMap":         true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}

# ═══════════════════════════════════════════════════════════
# functions/.env.example
# ═══════════════════════════════════════════════════════════
NODE_ENV=development

# Local Postgres (docker-compose)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=sql_connect_db
DB_USER=sql_connect_user
DB_PASS=sql_connect_pass_local

# Cloud SQL (production — leave empty for local)
CLOUD_SQL_CONNECTION_NAME=YOUR_PROJECT:us-central1:sql-connect-sql-imna
DB_IAM_USER=sql-connect-app@YOUR_PROJECT.iam.gserviceaccount.com

# GCP
GCP_PROJECT_ID=YOUR_PROJECT_ID
PUBSUB_REGION=us-central1

# Firebase Admin (use ADC locally: gcloud auth application-default login)
# FIREBASE_SERVICE_ACCOUNT_PATH=./service-account.json

# ═══════════════════════════════════════════════════════════
# functions/Dockerfile.dev
# ═══════════════════════════════════════════════════════════
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ENV NODE_ENV=development
CMD ["npm", "run", "dev"]

# ═══════════════════════════════════════════════════════════
# functions/Dockerfile (production)
# ═══════════════════════════════════════════════════════════
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm ci
COPY src ./src
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=builder /app/dist ./dist
USER node
EXPOSE 8080
CMD ["node", "dist/index.js"]

# ═══════════════════════════════════════════════════════════
# frontend/.env.example
# ═══════════════════════════════════════════════════════════
VITE_FIREBASE_API_KEY=your-api-key
VITE_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=your-project-id
VITE_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=your-sender-id
VITE_FIREBASE_APP_ID=your-app-id
VITE_API_URL=http://localhost:8080
VITE_WS_GATEWAY_URL=ws://localhost:8081

# ═══════════════════════════════════════════════════════════
# infra/main.tf — Terraform IaC
# ═══════════════════════════════════════════════════════════
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

variable "project_id" { type = string }
variable "region"     { default = "us-central1" }
variable "db_tier"    { default = "db-custom-4-26624" }

# ─── Cloud SQL ────────────────────────────────────────────
resource "google_sql_database_instance" "sql_connect_imna" {
  name             = "sql-connect-sql-imna"
  database_version = "POSTGRES_15"
  region           = var.region

  settings {
    tier              = var.db_tier
    availability_type = "REGIONAL"

    backup_configuration {
      enabled                        = true
      start_time                     = "02:00"
      point_in_time_recovery_enabled = true
      retained_backups               = 30
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.vpc.id
    }

    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = true
      record_client_address   = false
    }

    database_flags {
      name  = "max_connections"
      value = "500"
    }
    database_flags {
      name  = "log_min_duration_statement"
      value = "1000"  # log queries > 1 second
    }
  }

  deletion_protection = true
}

resource "google_sql_database" "sql_connect_db" {
  name     = "sql_connect_db"
  instance = google_sql_database_instance.sql_connect_imna.name
}

# ─── Pub/Sub Topics ──────────────────────────────────────
locals {
  regions = ["imna", "ime", "apac"]
}

resource "google_pubsub_topic" "events" {
  for_each = toset(local.regions)
  name     = "${each.key}-events"

  message_retention_duration = "604800s"  # 7 days
}

resource "google_pubsub_topic" "events_dlq" {
  for_each = toset(local.regions)
  name     = "${each.key}-events-dlq"
}

resource "google_pubsub_subscription" "events_sub" {
  for_each = toset(local.regions)
  name     = "${each.key}-events-sub"
  topic    = google_pubsub_topic.events[each.key].name

  ack_deadline_seconds       = 60
  message_retention_duration = "604800s"

  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.events_dlq[each.key].id
    max_delivery_attempts = 5
  }

  retry_policy {
    minimum_backoff = "10s"
    maximum_backoff = "600s"
  }
}

# ─── VPC ─────────────────────────────────────────────────
resource "google_compute_network" "vpc" {
  name                    = "sql-connect-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_global_address" "private_ip" {
  name          = "sql-connect-sql-private-ip"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.vpc.id
}

resource "google_service_networking_connection" "private_vpc" {
  network                 = google_compute_network.vpc.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip.name]
}

# ═══════════════════════════════════════════════════════════
# firebase/firebase.json
# ═══════════════════════════════════════════════════════════
{
  "dataconnect": {
    "source": "firebase/dataconnect"
  },
  "emulators": {
    "dataconnect": {
      "port": 9399,
      "dataDir": ".dataconnect"
    },
    "auth": {
      "port": 9099
    },
    "ui": {
      "enabled": true,
      "port": 4000
    }
  }
}
