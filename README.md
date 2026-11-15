# ⚡ NammaElectricity

**Real-time power outage mapping, regional grid reliability analytics, and smart appliance energy monitoring for Bengaluru.**

🌐 **Live Application:** [https://nammaelectricity.onrender.com/#home](https://nammaelectricity.onrender.com/#home)

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Render-46E3B7.svg?logo=render&logoColor=white)](https://nammaelectricity.onrender.com/#home)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Backend: Node.js + Express](https://img.shields.io/badge/Backend-Node.js%20%7C%20Express-green.svg)](https://expressjs.com/)
[![Database: PostgreSQL + Prisma](https://img.shields.io/badge/Database-PostgreSQL%20%7C%20Prisma-blue.svg)](https://www.prisma.io/)
[![Realtime: Socket.io](https://img.shields.io/badge/Realtime-Socket.io-black.svg)](https://socket.io/)

---

## 📖 Overview

**NammaElectricity** bridges the information gap between power distribution updates and everyday consumers across Bengaluru neighborhoods. Instead of waiting blindly during unexpected blackouts or getting surprised by high monthly bills, users get live neighborhood outage maps, historical feeder reliability metrics, and appliance-level energy consumption alerts.

* **Live Web App:** [https://nammaelectricity.onrender.com/#home](https://nammaelectricity.onrender.com/#home)

---

## ✨ Key Features

* **🗺️ Real-Time Power Outage Map:** Live geospatial view of active scheduled maintenance and unscheduled outages across Bengaluru zones and substations, powered by WebSockets.
* **📣 Crowdsourced Outage Verification:** Residents can report local power cuts and verify restoration status in their locality to surface unannounced feeder trips faster.
* **📊 Regional Historical Frequency Tracking:** Tracks outage frequency, average restoration duration, and peak disruption hours by area/pincode over 30-, 90-, and 365-day windows.
* **🔌 Appliance Energy & Spike Alerts:** Log household appliances with wattage ratings and daily runtime to estimate kWh usage, project slab-based monthly bills, and trigger threshold alerts before crossing higher tariff tiers.
* **🔔 Automated Outage & Bill Notifications:** Push and SMS/email alerts for upcoming scheduled maintenance in a user's saved locality and abnormal daily consumption spikes.

---

## 🏗️ Tech Stack & Backend Architecture

* **Frontend / Mobile Client:** React.js + Tailwind CSS (Web Dashboard) & React Native / Flutter (Mobile App)
* **Backend Framework:** **Node.js (v20+)** with **Express.js** (REST API)
* **Real-Time Engine:** **Socket.io** for live map updates and room-based locality outage broadcasts
* **Database & ORM:** **PostgreSQL** with **PostGIS** (geospatial queries) & **Prisma ORM**
* **Caching & Background Workers:** **Redis** + **BullMQ / node-cron** for scheduled maintenance scraping, outage aggregation, and consumption threshold alerts
* **Authentication:** **JWT** (Access + Refresh tokens) with role-based access (`CONSUMER`, `MODERATOR`, `ADMIN`)
* **Hosting & Deployment:** **Render** ([nammaelectricity.onrender.com](https://nammaelectricity.onrender.com/#home))

---

## 🗄️ Database Schema

The relational schema models users, Bengaluru sub-zones/feeders, outage events, crowdsourced reports, and household appliance logs.

### Entity Overview

* **`users`**: Stores consumer profiles, saved locality/pincode, notification preferences, and monthly kWh budget targets.
* **`zones`**: Represents Bengaluru subdivisions, feeder codes, and bounding coordinates.
* **`outage_events`**: Tracks verified scheduled and unscheduled power cuts, cause, start time, estimated restoration time (ETR), and actual restoration time.
* **`outage_reports`**: Stores user-submitted outage reports with geo-coordinates and upvote confirmations used to auto-verify emerging blackouts.
* **`appliances`**: Tracks user-registered household devices, rated wattage, average daily hours, and calculated daily kWh.
* **`consumption_logs`**: Records daily/monthly meter or estimated kWh readings and alert triggers.

### Prisma Schema (`prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  CONSUMER
  MODERATOR
  ADMIN
}

enum OutageType {
  SCHEDULED
  UNSCHEDULED
  EMERGENCY
}

enum OutageStatus {
  ACTIVE
  RESTORED
  CANCELLED
}

model User {
  id               String           @id @default(uuid())
  name             String
  email            String           @unique
  phone            String?          @unique
  passwordHash     String
  role             Role             @default(CONSUMER)
  zoneId           String?
  zone             Zone?            @relation(fields: [zoneId], references: [id])
  monthlyKwhTarget Float            @default(200.0)
  reports          OutageReport[]
  appliances       Appliance[]
  consumptionLogs  ConsumptionLog[]
  createdAt        DateTime         @default(now())
  updatedAt        DateTime         @updatedAt
}

model Zone {
  id           String         @id @default(uuid())
  name         String         // e.g., "Indiranagar", "Electronic City Phase 1"
  pincode      String
  subdivision  String         // e.g., "E-4 Subdivision"
  latitude     Float
  longitude    Float
  users        User[]
  outageEvents OutageEvent[]
  reports      OutageReport[]

  @@index([pincode])
}

model OutageEvent {
  id                String       @id @default(uuid())
  zoneId            String
  zone              Zone         @relation(fields: [zoneId], references: [id])
  type              OutageType
  status            OutageStatus @default(ACTIVE)
  title             String
  reason            String?      // e.g., "Transformer maintenance", "Feeder trip"
  affectedAreas     String[]
  startedAt         DateTime
  estimatedRestore  DateTime?
  restoredAt        DateTime?
  verificationCount Int          @default(1)
  createdAt         DateTime     @default(now())
  updatedAt         DateTime     @updatedAt

  @@index([zoneId, status])
  @@index([startedAt])
}

model OutageReport {
  id          String   @id @default(uuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  zoneId      String
  zone        Zone     @relation(fields: [zoneId], references: [id])
  latitude    Float
  longitude   Float
  description String?
  isResolved  Boolean  @default(false)
  createdAt   DateTime @default(now())
}

model Appliance {
  id              String   @id @default(uuid())
  userId          String
  user            User     @relation(fields: [userId], references: [id])
  name            String   // e.g., "Split AC (1.5 Ton)", "Water Heater"
  category        String   // e.g., "Cooling", "Heating", "Kitchen"
  wattage         Int      // Rated power in Watts
  dailyUsageHours Float    // Average hours used per day
  isAlertEnabled  Boolean  @default(true)
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}

model ConsumptionLog {
  id             String   @id @default(uuid())
  userId         String
  user           User     @relation(fields: [userId], references: [id])
  date           DateTime @db.Date
  consumedKwh    Float
  estimatedCost  Float
  alertTriggered Boolean  @default(false)
  createdAt      DateTime @default(now())

  @@unique([userId, date])
}
