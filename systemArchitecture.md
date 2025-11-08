System Architecture - Plantix Nigeria

System Overview

Plantix Nigeria presents a voice-first, offline-capable mobile application designed specifically for Nigerian farmers. This innovative app diagnose crop diseases and offers solutions in English, Yoruba, Igbo, and Hausa, ensuring that local farmers receive support in their native languages

Core Technical Requirements:
- Offline disease diagnosis without internet
- Voice input/output in four Nigerian languages
- Real-time geolocation for agro-dealer network
- Community disease outbreak tracking
- Scalable to support multiple concurrent users

Technology Stack

Frontend - Mobile Application

Platform: React Native  
Why: Cross-platform development (iOS & Android from single codebase), native performance, large community support

Core Libraries:
react-native-paper - Material Design UI components

 Backend - Server Infrastructure

Runtime Environment:Node.js v20 LTS  
Framework:Express.js v4.18+

Core Packages:
- express - Web framework
- jsonwebtoken - Authentication tokens
- bcrypt - Password hashing
- helmet - Security middleware
- morgan - Request logging
- express-validator - Input validation
- multer - File upload handling (crop images)
- socket.io - Real-time outbreak alerts
- node-cron - Scheduled tasks (price updates)

External Integrations:
- Google Cloud Speech-to-Text: Voice recognition with custom Nigerian language models
- Google Cloud Text-to-Speech: Voice output
- Google Maps Geocoding API: Location services
- Twilio SMS Gateway: Backup alert system for offline farmers
- Paystack Payment API: Premium subscription processing






