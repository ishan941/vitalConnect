# VitalConnect - RFID Integration Solution

## 🎯 Complete Solution Overview

This document explains the complete RFID card scanning solution for VitalConnect, including patient registration flow, real-time card scanning, and ESP32 hardware integration.

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Patient Registration Flow](#patient-registration-flow)
3. [RFID Real-Time Scanning](#rfid-real-time-scanning)
4. [ESP32 Hardware Setup](#esp32-hardware-setup)
5. [Backend Implementation](#backend-implementation)
6. [Frontend Implementation](#frontend-implementation)
7. [Testing Guide](#testing-guide)

---

## 🏗️ Architecture Overview

```
┌─────────────┐         WebSocket          ┌──────────────┐
│   ESP32 +   │◄─────────────────────────►│   Backend    │
│   RDM6300   │      RFID Scan Data        │  (NestJS)    │
└─────────────┘                             └──────┬───────┘
                                                   │
                                                   │ WebSocket
                                                   │ Broadcast
                                                   ▼
                                            ┌──────────────┐
                                            │   Web App    │
                                            │  Dashboard   │
                                            │              │
                                            │ Auto-fills   │
                                            │   UID in     │
                                            │   Form       │
                                            └──────────────┘
```

---

## 👥 Patient Registration Flow

### Problem Identified

The original `/patients` endpoint expected only patient-specific data with a `userId` (UUID), but the web form was sending user data (firstName, lastName, email, phone), causing validation errors.

### Solution Implemented

#### New Endpoint: `POST /api/v1/patients/register`

This endpoint handles **complete patient registration** in a single transaction:

1. **Creates User Account** (Auth System)
   - First name, last name
   - Email (unique)
   - Phone number
   - Auto-generated secure password (or custom)
   - Role: PATIENT

2. **Creates Patient Profile** (linked to User)
   - Date of birth
   - Gender
   - Blood group
   - Address, city, district
   - Emergency contacts
   - Medical info (allergies, conditions)
   - Insurance details

### Who Can Register Patients?

| User Role   | Can Register via Web | Can Self-Register Mobile |
| ----------- | -------------------- | ------------------------ |
| ADMIN       | ✅ Yes               | N/A                      |
| SUPER_ADMIN | ✅ Yes               | N/A                      |
| DOCTOR      | ✅ Yes               | N/A                      |
| STAFF       | ✅ Yes               | N/A                      |
| PATIENT     | ❌ No                | ✅ Yes (Mobile App)      |

### Registration Flow

```
┌──────────────────────────────────────────────────────────┐
│                  Admin/Doctor Dashboard                  │
└───────────────────┬──────────────────────────────────────┘
                    │
                    │ 1. Fills patient form
                    │
                    ▼
┌──────────────────────────────────────────────────────────┐
│          POST /api/v1/patients/register                  │
│                                                           │
│  Request Body:                                            │
│  {                                                        │
│    "firstName": "Ram",                                    │
│    "lastName": "Shrestha",                                │
│    "email": "ram@example.com",                            │
│    "phone": "+9779812345678",                             │
│    "dateOfBirth": "1990-01-15",                           │
│    "gender": "male",                                      │
│    "bloodGroup": "O+",                                    │
│    "address": "Kathmandu",                                │
│    ...                                                    │
│  }                                                        │
└───────────────────┬──────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────┐
│              Transaction Begins (Database)               │
│                                                           │
│  Step 1: Create User                                      │
│  Step 2: Create Patient (linked to User)                 │
│  Step 3: Commit Transaction                               │
└───────────────────┬──────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────────────┐
│                  Response Sent                            │
│                                                           │
│  {                                                        │
│    "user": {                                              │
│      "id": "uuid...",                                     │
│      "email": "ram@example.com",                          │
│      "firstName": "Ram",                                  │
│      "temporaryPassword": "Gen3rated#Pass"  // if not provided│
│    },                                                     │
│    "patient": {                                           │
│      "id": "uuid...",                                     │
│      "userId": "uuid...",                                 │
│      "bloodGroup": "O+",                                  │
│      ...                                                  │
│    }                                                      │
│  }                                                        │
└──────────────────────────────────────────────────────────┘
```

---

## 📡 RFID Real-Time Scanning

### How It Works

1. **ESP32 connects** to backend WebSocket server (`/rfid` namespace)
2. **ESP32 registers** itself with unique device ID and location
3. **When RFID card is scanned**:
   - ESP32 reads UID from RDM6300
   - Sends UID via WebSocket to backend
   - Backend **broadcasts** to all connected web clients
   - Web dashboard **auto-fills** UID in registration form

### WebSocket Events

#### From ESP32 to Backend:

```javascript
// Register device
socket.emit("esp32:register", {
  deviceId: "ESP32_RFID_001",
  location: "Reception Desk",
});

// Send scan data
socket.emit("rfid:scan", {
  uid: "A4:B3:C2:D1:E0",
  deviceId: "ESP32_RFID_001",
  timestamp: "2026-02-12T10:30:00Z",
  location: "Reception Desk",
});
```

#### From Backend to Web Dashboard:

```javascript
// Card scanned notification
socket.on("rfid:scanned", (data) => {
  console.log(data);
  // {
  //   uid: 'A4:B3:C2:D1:E0',
  //   deviceId: 'ESP32_RFID_001',
  //   timestamp: '2026-02-12T10:30:00Z',
  //   location: 'Reception Desk'
  // }

  // Auto-fill UID in form
  setFormUid(data.uid);
  setDialogOpen(true); // Open registration modal
});
```

---

## 🔧 ESP32 Hardware Setup

### Hardware Required

- **ESP32 Development Board** (any variant)
- **RDM6300 125kHz RFID Reader Module**
- **RFID Cards/Tags** (125kHz EM4100 compatible)
- **Jumper Wires**
- **USB Cable** (for programming)

### Wiring Diagram

```
┌─────────────────┐
│     RDM6300     │
│                 │
│  TX   VCC  GND  │
└──┬────┬────┬────┘
   │    │    │
   │    │    └────────────┐
   │    │                 │
   │    └──────────┐      │
   │               │      │
   ▼               ▼      ▼
  RX2             5V    GND
┌─────────────────────────┐
│        ESP32            │
│                         │
│  GPIO 16 (RX2)          │
│  5V                     │
│  GND                    │
└─────────────────────────┘
```

### Pin Connections

| RDM6300 | ESP32         |
| ------- | ------------- |
| TX      | RX2 (GPIO 16) |
| VCC     | 5V            |
| GND     | GND           |

### Programming Steps

1. **Install Arduino IDE**: Download from [arduino.cc](https://www.arduino.cc/en/software)

2. **Install ESP32 Board**:
   - File → Preferences
   - Add to "Additional Board Manager URLs":
     ```
     https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
     ```
   - Tools → Board → Boards Manager
   - Search "ESP32" and install

3. **Install Required Libraries**:
   - Tools → Manage Libraries
   - Install:
     - `ArduinoJson` by Benoit Blanchon
     - `SocketIOClient` by Markus Sattler
     - `WebSockets` by Markus Sattler

4. **Configure the Code**:
   Open `esp32_rfid_scanner.ino` and update:

   ```cpp
   const char* ssid = "YOUR_WIFI_SSID";
   const char* password = "YOUR_WIFI_PASSWORD";
   const char* serverUrl = "192.168.1.100";  // Your server IP
   ```

5. **Upload**:
   - Connect ESP32 via USB
   - Tools → Board → ESP32 Dev Module
   - Tools → Port → Select your ESP32 port
   - Click Upload ▶

6. **Monitor**:
   - Tools → Serial Monitor (115200 baud)
   - Watch connection status and scan events

### LED Status Indicators

| Pattern         | Meaning                       |
| --------------- | ----------------------------- |
| 3 slow blinks   | WiFi connected successfully   |
| 2 quick blinks  | Device registered with server |
| 1 quick blink   | Card scanned and sent         |
| 5 rapid blinks  | Error sending scan data       |
| 10 rapid blinks | WiFi connection failed        |

---

## 🔙 Backend Implementation

### Files Modified/Created:

1. **New DTO**: `dto/admin-register-patient.dto.ts`
   - Combined User + Patient fields
   - Validation for all required fields

2. **WebSocket Gateway**: `rfid/rfid.gateway.ts`
   - Handles ESP32 connections
   - Broadcasts RFID scans to web clients
   - Device registration and management

3. **Patients Controller**: `patients.controller.ts`
   - New endpoint: `POST /patients/register`
   - Restricted to ADMIN/DOCTOR/STAFF

4. **Patients Service**: `patients.service.ts`
   - `adminRegisterPatient()` method
   - Transaction-based user + patient creation
   - Auto-generates secure password if not provided

5. **Users Service**: `users.service.ts`
   - `createUserTransaction()` method
   - Allows user creation within existing transaction

### Key Features:

- **Atomic Transactions**: User and Patient creation in single transaction
- **Auto-Password Generation**: Secure 12-character passwords
- **WebSocket Support**: Real-time RFID scanning
- **Device Management**: Track multiple ESP32 scanners
- **UID Validation**: Ensures proper RFID format

---

## 🎨 Frontend Implementation

### Web Dashboard Changes:

1. **Patients Page** (`patients/page.tsx`):
   - Updated to use `/patients/register` endpoint
   - No more validation errors!

2. **RFID Page** (`rfid/page.tsx`):
   - WebSocket connection to backend
   - "Start Listening" button to toggle scanning mode
   - Real-time UID display when card is scanned
   - Auto-opens registration modal with pre-filled UID
   - Visual indicators for listening status

### UI Components Added:

```tsx
// Listen button with status
<Button onClick={toggleListening}>
  <Radio className={isListening && "animate-pulse"} />
  {isListening ? "Listening..." : "Start Listening"}
</Button>;

// Last scanned card display
{
  lastScanned && (
    <div className="bg-emerald-50 text-emerald-700">Last: {lastScanned}</div>
  );
}
```

---

## 🧪 Testing Guide

### 1. Test Patient Registration (Without RFID)

```bash
curl -X POST http://localhost:3000/api/v1/patients/register \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -d '{
    "firstName": "Test",
    "lastName": "Patient",
    "email": "test@example.com",
    "phone": "+9779812345678",
    "dateOfBirth": "1990-01-15",
    "gender": "male",
    "bloodGroup": "O+"
  }'
```

Expected: Success response with user and patient data

### 2. Test WebSocket Connection

Open browser console on RFID page:

```javascript
// Should see:
// "Connected to RFID WebSocket"
// WebSocket {url: "ws://localhost:3000/rfid", ...}
```

### 3. Test ESP32 RFID Scanning

1. Open Serial Monitor on ESP32
2. You should see:

   ```
   [WiFi] Connected!
   [WebSocket] Connected to server
   [WebSocket] Device registered successfully!
   [System] Ready! Waiting for RFID cards...
   ```

3. Scan an RFID card:

   ```
   [RFID] Card detected!
   [RFID] UID: A4:B3:C2:D1:E0
   [RFID] Scan sent to server: A4:B3:C2:D1:E0
   [WebSocket] Scan acknowledged by server
   ```

4. Check web dashboard - UID should auto-fill!

### 4. Complete Integration Test

1. **Start backend**: `cd backend && npm run start:dev`
2. **Start web app**: `cd web && npm run dev`
3. **Upload ESP32 code** with correct WiFi and server IP
4. **Open web dashboard** → RFID Cards page
5. **Click "Start Listening"**
6. **Scan RFID card** with ESP32
7. **Verify**:
   - Alert pops up with UID
   - Registration modal opens
   - UID field is pre-filled
   - Click "Register Card" button
   - Select patient and submit

---

## 🚀 Deployment Notes

### Backend Environment Variables

```env
# WebSocket Configuration
CORS_ORIGIN=http://localhost:3001,http://your-production-domain.com

# Backend will automatically handle WebSocket on same port as HTTP
PORT=3000
```

### Security Considerations

1. **WebSocket Authentication**: Consider adding JWT auth to WebSocket connections for production
2. **RFID UID Validation**: Ensure only valid UID formats are accepted
3. **Rate Limiting**: Implement rate limiting on scan events to prevent abuse
4. **Device Whitelisting**: Store approved ESP32 device IDs in database

### Production Checklist

- [ ] Update ESP32 code with production server IP
- [ ] Update CORS_ORIGIN in backend
- [ ] Enable WebSocket SSL/TLS (wss://)
- [ ] Set up device registration approval flow
- [ ] Configure firewall to allow WebSocket connections
- [ ] Test with multiple ESP32 devices simultaneously
- [ ] Monitor WebSocket connection stability
- [ ] Set up logging for all RFID scans

---

## 📞 Support

For issues or questions:

1. Check Serial Monitor output from ESP32
2. Check browser console for WebSocket errors
3. Check backend logs for connection issues
4. Verify network connectivity between all components

---

## 🎉 Success!

You now have a complete RFID integration system that:

- ✅ Allows admin/doctors to register patients via web
- ✅ Real-time RFID card scanning from ESP32
- ✅ Auto-fills UID when card is scanned
- ✅ Works with multiple scanning stations
- ✅ Professional and user-friendly interface

Enjoy your smart RFID patient management system! 🏥✨
