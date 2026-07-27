# SETUKA Study Material

## Overview
SETUKA is an AI-powered hybrid emergency communication system designed for areas without reliable cellular connectivity.

### Tech Stack
- ESP32
- SX1262 LoRa Mesh
- FastAPI
- PostgreSQL
- GenAI
- Isolation Forest

## Problem
Conventional emergency systems fail in remote regions because they depend on mobile networks or the Internet.

## Solution
SETUKA uses multiple communication methods in priority:
1. Internet
2. GSM
3. LoRa Mesh
4. Satellite (future)

## System Architecture
User → Smart Device → ESP32 → Decision Engine → LoRa/GSM/Internet → FastAPI → PostgreSQL → GenAI → Dashboard

## Hardware
### ESP32
- Reads sensors
- Reads GPS
- Controls communication
- Sends SOS packets

### SX1262 LoRa
- Long-range communication
- Low power
- Multi-hop mesh support

### GPS
- Latitude
- Longitude
- Altitude
- Time

### Sensors
- Heart rate
- SpO₂
- Motion/Fall
- Temperature

## LoRa Mesh
Each node forwards packets until they reach a gateway.

### Packet Fields
- Packet ID
- Source ID
- Destination ID
- Timestamp
- GPS
- Sensor Data
- TTL
- CRC

### Reliability
- Duplicate filtering
- ACKs
- Retransmission
- CRC validation

Approximate reliability: **95%**.

## Backend (FastAPI)
Modules:
- Authentication
- Packet Parser
- REST APIs
- Database
- AI Report Generator
- Notification Engine

## Database
Tables:
- Users
- Devices
- Sensor Data
- SOS Events
- Incident Reports
- Mesh Logs

## Isolation Forest
Used for anomaly detection from sensor readings.

Workflow:
Sensor → Features → Isolation Forest → Normal / Alert

Approximate precision: **88%**.

## GenAI
Converts raw sensor data into readable emergency reports.

Example:
- Raw: HR=42, SpO₂=83, Fall Detected
- AI: Possible medical emergency. Immediate assistance recommended.

## API Examples
- POST /api/sos
- POST /api/ack
- GET /api/events
- POST /api/report

## Functional Workflow
Emergency → ESP32 → Sensors → GPS → Isolation Forest → LoRa Mesh → Gateway → FastAPI → Database → GenAI → Dashboard

## Interview Justification
- Designed LoRa mesh routing with ESP32 nodes.
- Built FastAPI backend for IoT devices.
- Used Isolation Forest because emergencies are anomalies.
- Used GenAI to automate incident summaries.
- Contributed to product strategy, Hult Prize finalist journey, and provisional patent.

## Common Interview Questions
1. Why LoRa?
2. Why FastAPI?
3. Why Isolation Forest?
4. How does mesh routing work?
5. How are duplicate packets prevented?
6. How is security implemented?
7. How would the system scale?
