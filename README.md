# Cloud-Based ANPR Parking Management System Development Guide

## System Architecture Overview

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  ANPR Camera    │────│  Edge Device     │────│  Cloud Backend  │
│  (Entry/Exit)   │    │  (Local Process) │    │  (Main System)  │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                        │
                       ┌─────────────────┐             │
                       │  Gate Barrier   │─────────────┘
                       │  Controller     │
                       └─────────────────┘
                                │
                       ┌─────────────────┐
                       │  Admin Panel    │
                       │  Mobile App     │
                       └─────────────────┘
```

## Step 1: Technology Stack Selection

### Backend (Cloud)
- **Framework**: Node.js with Express.js or Python with FastAPI
- **Database**: PostgreSQL (primary) + Redis (caching/sessions)
- **Cloud Provider**: AWS/Google Cloud/Azure
- **Container**: Docker + Kubernetes
- **Message Queue**: Redis/RabbitMQ for real-time communication

### Frontend
- **Admin Panel**: React.js/Vue.js
- **Mobile App**: React Native/Flutter
- **Payment UI**: Embedded payment widgets

### ANPR & Hardware Integration
- **ANPR Engine**: OpenALPR (open source)
- **Edge Computing**: Raspberry Pi 4 or similar
- **Communication**: MQTT/HTTP APIs

## Step 2: Open Source Components to Leverage

### 1. ANPR Recognition
```bash
# OpenALPR - License Plate Recognition
git clone https://github.com/openalpr/openalpr
# Alternative: EasyOCR for license plates
pip install easyocr
```

### 2. Payment Gateway Integration
```bash
# Stripe SDK (most comprehensive)
npm install stripe
# PayPal SDK
npm install @paypal/checkout-server-sdk
# Razorpay (for India/Asia)
npm install razorpay
```

### 3. QR Code Generation
```bash
# QR Code generation
npm install qrcode
pip install qrcode
```

### 4. Database & ORM
```bash
# For Node.js
npm install prisma postgresql
# For Python
pip install sqlalchemy psycopg2-binary
```

### 5. Real-time Communication
```bash
# Socket.io for real-time updates
npm install socket.io
# MQTT for IoT communication
npm install mqtt
```

## Step 3: Database Schema Design

### Core Tables
```sql
-- Vehicles table
CREATE TABLE vehicles (
    id SERIAL PRIMARY KEY,
    license_plate VARCHAR(20) UNIQUE NOT NULL,
    owner_name VARCHAR(100),
    phone_number VARCHAR(15),
    email VARCHAR(100),
    is_registered BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Parking sessions
CREATE TABLE parking_sessions (
    id SERIAL PRIMARY KEY,
    vehicle_id INTEGER REFERENCES vehicles(id),
    license_plate VARCHAR(20) NOT NULL,
    entry_time TIMESTAMP DEFAULT NOW(),
    exit_time TIMESTAMP,
    entry_image_url VARCHAR(255),
    exit_image_url VARCHAR(255),
    parking_fee DECIMAL(10,2),
    payment_status VARCHAR(20) DEFAULT 'pending',
    payment_id VARCHAR(100),
    qr_code_data TEXT,
    is_active BOOLEAN DEFAULT TRUE
);

-- Parking rates configuration
CREATE TABLE parking_rates (
    id SERIAL PRIMARY KEY,
    rate_name VARCHAR(50),
    hourly_rate DECIMAL(10,2),
    free_minutes INTEGER DEFAULT 15,
    max_daily_rate DECIMAL(10,2),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Payment transactions
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    session_id INTEGER REFERENCES parking_sessions(id),
    amount DECIMAL(10,2),
    payment_method VARCHAR(50),
    payment_gateway_id VARCHAR(100),
    status VARCHAR(20),
    qr_code_url VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);
```

## Step 4: Core Modules Development

### Module 1: ANPR Service (Edge Device)
```python
# anpr_service.py using OpenALPR
import openalpr
import requests
import json
from datetime import datetime

class ANPRService:
    def __init__(self):
        self.alpr = openalpr.Alpr("us", "/path/to/openalpr.conf", "/path/to/runtime_data")
        self.cloud_api_url = "https://your-cloud-api.com"
    
    def process_image(self, image_path, gate_type="entry"):
        results = self.alpr.recognize_file(image_path)
        
        if results['results']:
            license_plate = results['results'][0]['plate']
            confidence = results['results'][0]['confidence']
            
            if confidence > 85:  # Confidence threshold
                self.send_to_cloud(license_plate, image_path, gate_type)
                return license_plate
        return None
    
    def send_to_cloud(self, plate, image_path, gate_type):
        data = {
            'license_plate': plate,
            'timestamp': datetime.now().isoformat(),
            'gate_type': gate_type,
            'image_path': image_path
        }
        requests.post(f"{self.cloud_api_url}/api/vehicle/{gate_type}", json=data)
```

### Module 2: Cloud Backend API
```javascript
// Node.js Express API
const express = require('express');
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
const QRCode = require('qrcode');

const app = express();

// Vehicle entry endpoint
app.post('/api/vehicle/entry', async (req, res) => {
    const { license_plate, timestamp, image_path } = req.body;
    
    try {
        // Check if vehicle already has active session
        const activeSession = await db.query(
            'SELECT * FROM parking_sessions WHERE license_plate = ? AND is_active = true',
            [license_plate]
        );
        
        if (activeSession.length > 0) {
            return res.status(400).json({ error: 'Vehicle already has active session' });
        }
        
        // Create new parking session
        const session = await db.query(
            'INSERT INTO parking_sessions (license_plate, entry_time, entry_image_url) VALUES (?, ?, ?)',
            [license_plate, timestamp, image_path]
        );
        
        // Open gate barrier
        await controlGateBarrier('entry', 'open');
        
        res.json({ success: true, session_id: session.insertId });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Vehicle exit endpoint
app.post('/api/vehicle/exit', async (req, res) => {
    const { license_plate, timestamp } = req.body;
    
    try {
        const session = await getActiveSession(license_plate);
        
        if (!session) {
            return res.status(404).json({ error: 'No active session found' });
        }
        
        // Calculate parking fee
        const fee = calculateParkingFee(session.entry_time, timestamp);
        
        if (fee <= 0) {
            // Free parking - open gate immediately
            await completeSession(session.id, fee, 'free');
            await controlGateBarrier('exit', 'open');
            return res.json({ success: true, fee: 0, status: 'free' });
        }
        
        // Generate QR code for payment
        const paymentUrl = `https://your-app.com/pay/${session.id}`;
        const qrCodeUrl = await QRCode.toDataURL(paymentUrl);
        
        // Update session with fee and QR code
        await db.query(
            'UPDATE parking_sessions SET parking_fee = ?, qr_code_data = ? WHERE id = ?',
            [fee, qrCodeUrl, session.id]
        );
        
        res.json({
            success: true,
            fee: fee,
            qr_code: qrCodeUrl,
            payment_url: paymentUrl,
            session_id: session.id
        });
        
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Payment processing
app.post('/api/payment/process', async (req, res) => {
    const { session_id, payment_method_id } = req.body;
    
    try {
        const session = await getSession(session_id);
        
        // Create Stripe payment intent
        const paymentIntent = await stripe.paymentIntents.create({
            amount: Math.round(session.parking_fee * 100), // Convert to cents
            currency: 'usd',
            payment_method: payment_method_id,
            confirmation_method: 'manual',
            confirm: true,
        });
        
        if (paymentIntent.status === 'succeeded') {
            // Update session as paid
            await completeSession(session_id, session.parking_fee, 'paid');
            
            // Open exit gate
            await controlGateBarrier('exit', 'open');
            
            res.json({ success: true, payment_id: paymentIntent.id });
        }
        
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Helper functions
async function calculateParkingFee(entryTime, exitTime) {
    const rates = await db.query('SELECT * FROM parking_rates WHERE is_active = true LIMIT 1');
    const rate = rates[0];
    
    const durationMs = new Date(exitTime) - new Date(entryTime);
    const durationMinutes = Math.floor(durationMs / (1000 * 60));
    
    if (durationMinutes <= rate.free_minutes) {
        return 0;
    }
    
    const billableHours = Math.ceil((durationMinutes - rate.free_minutes) / 60);
    return Math.min(billableHours * rate.hourly_rate, rate.max_daily_rate);
}

async function controlGateBarrier(gate, action) {
    // Send MQTT command to gate controller
    const mqtt = require('mqtt');
    const client = mqtt.connect('mqtt://your-mqtt-broker');
    
    client.publish(`gates/${gate}/control`, JSON.stringify({
        action: action,
        timestamp: new Date().toISOString()
    }));
}
```

### Module 3: Admin Panel Features
```javascript
// Admin configuration API
app.get('/api/admin/rates', async (req, res) => {
    const rates = await db.query('SELECT * FROM parking_rates WHERE is_active = true');
    res.json(rates);
});

app.post('/api/admin/rates', async (req, res) => {
    const { hourly_rate, free_minutes, max_daily_rate } = req.body;
    
    // Deactivate old rates
    await db.query('UPDATE parking_rates SET is_active = false');
    
    // Create new rate
    const result = await db.query(
        'INSERT INTO parking_rates (hourly_rate, free_minutes, max_daily_rate) VALUES (?, ?, ?)',
        [hourly_rate, free_minutes, max_daily_rate]
    );
    
    res.json({ success: true, rate_id: result.insertId });
});

// Extend free time for specific session
app.post('/api/admin/extend-free-time/:sessionId', async (req, res) => {
    const { sessionId } = req.params;
    const { additional_minutes } = req.body;
    
    // Add free time extension to session
    await db.query(
        'UPDATE parking_sessions SET free_time_extension = ? WHERE id = ?',
        [additional_minutes, sessionId]
    );
    
    res.json({ success: true });
});
```

## Step 5: Deployment Architecture

### Cloud Infrastructure (AWS Example)
```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: ./backend
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/parking
      - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:14
    environment:
      - POSTGRES_DB=parking
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=secure_password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    
  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - api

volumes:
  postgres_data:
```

### Edge Device Setup (Raspberry Pi)
```bash
# Install OpenALPR on Raspberry Pi
sudo apt-get update
sudo apt-get install openalpr openalpr-daemon openalpr-Utils libopenalpr-dev

# Install Python dependencies
pip3 install openalpr-python requests opencv-python

# Set up as systemd service
sudo systemctl enable anpr-service
sudo systemctl start anpr-service
```

## Step 6: Integration Steps

### 1. Hardware Setup
- Install ANPR cameras at entry/exit points
- Connect gate barrier controllers
- Set up Raspberry Pi as edge device
- Configure network connectivity

### 2. Software Deployment
- Deploy cloud backend to AWS/GCP/Azure
- Configure database and Redis
- Set up payment gateway accounts (Stripe/PayPal)
- Deploy frontend applications

### 3. Configuration
- Set initial parking rates
- Configure camera positions and recognition zones
- Test gate barrier integration
- Set up monitoring and logging

### 4. Testing Protocol
- Test ANPR accuracy with various license plates
- Verify payment gateway integration
- Test gate barrier automation
- Load testing for concurrent users
- Security testing

## Step 7: Monitoring & Maintenance

### Key Metrics to Monitor
- ANPR recognition accuracy
- Payment success rates
- Gate barrier response times
- System uptime
- Revenue tracking

### Logging Strategy
```javascript
// Structured logging example
const winston = require('winston');

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: 'parking-system.log' }),
        new winston.transports.Console()
    ]
});

// Log all vehicle entries/exits
logger.info('Vehicle entry', {
    license_plate: 'ABC123',
    timestamp: new Date(),
    gate: 'entry_1',
    confidence: 92.5
});
```

## Estimated Development Timeline

- **Week 1-2**: Setup infrastructure, basic ANPR integration
- **Week 3-4**: Core backend API development
- **Week 5-6**: Payment gateway integration
- **Week 7-8**: Admin panel and mobile app
- **Week 9-10**: Hardware integration and testing
- **Week 11-12**: Deployment and fine-tuning

## Budget Considerations

### Monthly Operating Costs
- Cloud hosting: $100-500/month
- Payment processing: 2.9% + $0.30 per transaction
- ANPR software licensing: $0-200/month (if using commercial)
- Hardware maintenance: $50-100/month

### Initial Setup Costs
- Development: $15,000-30,000
- Hardware (cameras, barriers, edge devices): $5,000-15,000
- Cloud setup and configuration: $2,000-5,000

This comprehensive system will provide a scalable, cloud-based parking management solution with all the features you requested. The modular approach allows you to implement features incrementally and scale as needed.
