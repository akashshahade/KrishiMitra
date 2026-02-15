# KrishiMitra - Technical Design Document

## Table of Contents
1. [System Architecture](#system-architecture)
2. [Component Design](#component-design)
3. [Data Models](#data-models)
4. [API Specifications](#api-specifications)
5. [User Interface Design](#user-interface-design)
6. [Security Architecture](#security-architecture)
7. [Deployment Strategy](#deployment-strategy)
8. [Testing Strategy](#testing-strategy)

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│  Web App (PWA)  │  Mobile App  │  WhatsApp Bot  │  SMS Gateway  │
└────────┬────────┴──────┬───────┴───────┬────────┴───────┬────────┘
         │               │               │                │
         └───────────────┴───────────────┴────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   API Gateway     │
                    │   (Load Balancer) │
                    └─────────┬─────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼─────┐      ┌──────▼──────┐      ┌─────▼──────┐
    │  Web API │      │  ML Service │      │  Bot Service│
    │ (Node.js)│      │  (FastAPI)  │      │  (Node.js) │
    └────┬─────┘      └──────┬──────┘      └─────┬──────┘
         │                   │                    │
         └───────────────────┼────────────────────┘
                             │
            ┌────────────────┼─────────────────┐
            │                │                 │
       ┌────▼────┐    ┌─────▼──────┐    ┌────▼──────┐
       │PostgreSQL│    │   Redis    │    │  MongoDB  │
       │ (Primary)│    │  (Cache)   │    │ (Images)  │
       └──────────┘    └────────────┘    └───────────┘

                    EXTERNAL SERVICES
       ┌──────────────────────────────────────────┐
       │  Claude API  │  Agmarknet  │  IMD/Weather│
       │  Twilio SMS  │  WhatsApp   │  Google Maps│
       └──────────────────────────────────────────┘
```

### Architecture Principles

1. **Microservices Pattern:** Independent services for core functionalities
2. **API-First Design:** All features accessible via RESTful APIs
3. **Cloud-Native:** Designed for AWS/Azure with containerization
4. **Scalability:** Horizontal scaling for each service
5. **Resilience:** Circuit breakers, retries, graceful degradation
6. **Security:** Zero-trust model with encryption at rest and in transit

---

## Component Design

### 1. Web API Service (Node.js/Express)

**Responsibilities:**
- User authentication and authorization
- User profile management
- Query routing to appropriate services
- Response aggregation
- Rate limiting and throttling

**Key Modules:**

```javascript
// Authentication Module
class AuthService {
  async register(userData)
  async login(credentials)
  async verifyToken(token)
  async refreshToken(refreshToken)
}

// User Profile Module
class UserProfileService {
  async createProfile(userId, profileData)
  async updateProfile(userId, updates)
  async getProfile(userId)
  async getUserCrops(userId)
  async updatePreferences(userId, preferences)
}

// Query Router Module
class QueryRouterService {
  async routeQuery(query, userId)
  async processTextQuery(query, context)
  async processImageQuery(image, userId)
  async getHistory(userId, limit)
}
```

**Technology Stack:**
- Node.js v18+
- Express.js v4.18+
- JWT for authentication
- Joi for validation
- Winston for logging

---

### 2. ML Service (Python/FastAPI)

**Responsibilities:**
- Crop disease detection from images
- Price prediction using time-series models
- Claude API integration for advisory
- Language translation
- Image preprocessing

**Key Modules:**

```python
# Disease Detection Module
class DiseaseDetectionService:
    def __init__(self):
        self.model = load_model('disease_classifier_v2.h5')
        
    async def predict_disease(self, image_bytes: bytes) -> dict:
        """
        Returns: {
            'disease': str,
            'confidence': float,
            'treatment': str,
            'severity': str
        }
        """
        
    def preprocess_image(self, image: bytes) -> np.array:
        # Resize to 224x224, normalize
        pass

# Price Prediction Module
class PricePredictionService:
    def __init__(self):
        self.model = load_model('price_predictor.pkl')
        
    async def predict_price(self, commodity: str, 
                           market: str, 
                           days_ahead: int = 7) -> dict:
        """
        Returns: {
            'predicted_prices': List[float],
            'trend': str,
            'recommendation': str,
            'confidence': float
        }
        """

# Claude Advisory Module
class AdvisoryService:
    def __init__(self):
        self.client = Anthropic(api_key=os.getenv('CLAUDE_API_KEY'))
        
    async def get_advisory(self, query: str, 
                          context: dict,
                          language: str = 'en') -> str:
        """
        Context includes: user_crops, location, season, history
        Returns personalized advisory in specified language
        """
        system_prompt = self._build_system_prompt(context)
        response = await self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1000,
            system=system_prompt,
            messages=[{"role": "user", "content": query}]
        )
        return self._translate_if_needed(response.content, language)
```

**Technology Stack:**
- Python 3.10+
- FastAPI 0.109+
- TensorFlow 2.15+ / PyTorch 2.1+
- Anthropic Claude API
- OpenCV for image processing
- scikit-learn for price models

**ML Models:**

1. **Disease Detection Model**
   - Architecture: MobileNetV2 (transfer learning)
   - Input: 224x224 RGB images
   - Output: 25 disease classes + confidence scores
   - Training data: 50,000 images from PlantVillage + custom dataset
   - Accuracy target: 85%+

2. **Price Prediction Model**
   - Algorithm: LSTM + ARIMA ensemble
   - Features: Historical prices, season, weather, demand indicators
   - Output: 7-day price forecast + trend
   - Accuracy target: 80%+ (MAPE < 20%)

---

### 3. Bot Service (Node.js)

**Responsibilities:**
- WhatsApp message handling
- SMS message processing
- Conversational context management
- Message formatting for different channels
- Rate limiting per user

**Key Modules:**

```javascript
// WhatsApp Bot Module
class WhatsAppBotService {
  constructor() {
    this.client = new WhatsAppClient(process.env.WHATSAPP_TOKEN);
    this.sessionManager = new SessionManager();
  }

  async handleIncomingMessage(message) {
    const session = await this.sessionManager.getSession(message.from);
    const context = this._buildContext(session, message);
    
    // Route to appropriate handler
    if (message.type === 'text') {
      return await this.handleTextMessage(message, context);
    } else if (message.type === 'image') {
      return await this.handleImageMessage(message, context);
    }
  }

  async handleTextMessage(message, context) {
    // Parse intent (price, weather, advisory)
    const intent = await this.parseIntent(message.text);
    
    switch(intent.type) {
      case 'PRICE_QUERY':
        return await this.handlePriceQuery(intent, context);
      case 'WEATHER_QUERY':
        return await this.handleWeatherQuery(intent, context);
      case 'ADVISORY_QUERY':
        return await this.handleAdvisoryQuery(intent, context);
      default:
        return await this.handleGeneralQuery(message.text, context);
    }
  }

  formatResponse(data, channel) {
    // Format based on channel (WhatsApp/SMS)
    // WhatsApp: Rich formatting, emojis, images
    // SMS: Plain text, concise
  }
}

// SMS Handler Module
class SMSHandlerService {
  async processKeywordQuery(smsText, phoneNumber) {
    // Parse keywords: "PRICE WHEAT NAGPUR"
    const parsed = this.parseKeywords(smsText);
    
    if (parsed.command === 'PRICE') {
      return await this.getPriceInfo(parsed.commodity, parsed.location);
    } else if (parsed.command === 'WEATHER') {
      return await this.getWeatherInfo(parsed.location);
    }
  }

  parseKeywords(text) {
    // Extract command, commodity, location
    const regex = /^(PRICE|WEATHER|HELP)\s+(\w+)?\s*(\w+)?/i;
    // Returns { command, commodity, location }
  }
}
```

**Technology Stack:**
- Node.js v18+
- WhatsApp Business API
- Twilio SDK for SMS
- Redis for session management

---

### 4. Data Integration Services

**Agmarknet Integration:**
```javascript
class AgmarknetService {
  async fetchMarketPrices(commodity, state, district) {
    // API endpoint: https://api.data.gov.in/resource/...
    const response = await this.httpClient.get(
      'https://api.data.gov.in/resource/9ef84268-d588-465a-a308-a864a43d0070',
      {
        params: {
          'api-key': process.env.AGMARKNET_API_KEY,
          'format': 'json',
          'filters[commodity]': commodity,
          'filters[state]': state
        }
      }
    );
    
    return this.transformPriceData(response.data);
  }

  transformPriceData(rawData) {
    return rawData.records.map(record => ({
      market: record.market,
      price: parseFloat(record.modal_price),
      date: record.arrival_date,
      quantity: record.arrivals,
      unit: 'quintal'
    }));
  }

  async getCachedPrices(commodity, location) {
    const cacheKey = `prices:${commodity}:${location}`;
    let prices = await redis.get(cacheKey);
    
    if (!prices) {
      prices = await this.fetchMarketPrices(commodity, location);
      await redis.setex(cacheKey, 3600, JSON.stringify(prices)); // 1 hour cache
    }
    
    return JSON.parse(prices);
  }
}
```

**Weather Integration:**
```javascript
class WeatherService {
  async getWeatherForecast(latitude, longitude) {
    const response = await axios.get(
      'https://api.openweathermap.org/data/2.5/onecall',
      {
        params: {
          lat: latitude,
          lon: longitude,
          appid: process.env.OPENWEATHER_API_KEY,
          units: 'metric',
          lang: 'en'
        }
      }
    );
    
    return {
      current: this.formatCurrentWeather(response.data.current),
      daily: this.formatDailyForecast(response.data.daily),
      alerts: this.formatAlerts(response.data.alerts)
    };
  }

  async sendWeatherAlert(userId, alertData) {
    const user = await User.findById(userId);
    
    // Send via user's preferred channel
    if (user.preferences.alertChannel === 'sms') {
      await this.sendSMSAlert(user.phone, alertData);
    } else if (user.preferences.alertChannel === 'whatsapp') {
      await this.sendWhatsAppAlert(user.phone, alertData);
    } else {
      await this.sendPushNotification(userId, alertData);
    }
  }
}
```

---

## Data Models

### User Schema (PostgreSQL)

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone VARCHAR(15) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    language VARCHAR(10) DEFAULT 'hi',
    state VARCHAR(50),
    district VARCHAR(50),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE user_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    land_size_acres DECIMAL(6, 2),
    farming_experience_years INTEGER,
    irrigation_type VARCHAR(50),
    soil_type VARCHAR(50),
    primary_crops TEXT[], -- Array of crop names
    preferences JSONB, -- { alertChannel: 'sms', units: 'metric' }
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_location ON users(latitude, longitude);
```

### Crop Schema

```sql
CREATE TABLE user_crops (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    crop_name VARCHAR(100) NOT NULL,
    variety VARCHAR(100),
    area_acres DECIMAL(6, 2),
    sowing_date DATE,
    expected_harvest_date DATE,
    status VARCHAR(20), -- 'sowing', 'growing', 'harvested'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE crop_master (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name_en VARCHAR(100) NOT NULL,
    name_hi VARCHAR(100),
    name_mr VARCHAR(100),
    category VARCHAR(50), -- 'cereal', 'pulse', 'vegetable', 'fruit'
    season VARCHAR(20), -- 'kharif', 'rabi', 'zaid'
    avg_duration_days INTEGER,
    common_diseases TEXT[]
);
```

### Query History Schema

```sql
CREATE TABLE query_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    query_type VARCHAR(50), -- 'advisory', 'price', 'weather', 'disease'
    query_text TEXT,
    response TEXT,
    metadata JSONB, -- { confidence: 0.92, disease_detected: 'rust' }
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_query_history_user ON query_history(user_id, created_at DESC);
```

### Disease Detection Schema (MongoDB)

```javascript
// MongoDB Collection: disease_detections
{
  _id: ObjectId,
  userId: String,
  imageUrl: String,
  imageHash: String, // For deduplication
  detectedDisease: {
    name: String,
    nameLo: String, // Localized name
    confidence: Number,
    severity: String, // 'low', 'medium', 'high'
  },
  treatment: {
    chemical: [String],
    organic: [String],
    preventive: [String]
  },
  cropType: String,
  location: {
    type: "Point",
    coordinates: [Number, Number] // [longitude, latitude]
  },
  metadata: {
    modelVersion: String,
    processingTime: Number,
    imageQuality: String
  },
  createdAt: Date,
  expiresAt: Date // Auto-delete after 30 days
}
```

### Market Prices Cache (Redis)

```
Key Pattern: prices:{commodity}:{state}:{district}
Value: JSON string
TTL: 3600 seconds (1 hour)

Example:
Key: "prices:wheat:maharashtra:nagpur"
Value: {
  "commodity": "wheat",
  "markets": [
    {
      "name": "Kalamna Mandi",
      "price": 2150,
      "unit": "quintal",
      "date": "2026-02-15",
      "trend": "up",
      "change": 50
    }
  ],
  "cachedAt": "2026-02-15T10:30:00Z"
}
```

---

## API Specifications

### Authentication APIs

**POST /api/v1/auth/register**
```json
Request:
{
  "phone": "+919876543210",
  "name": "Ramesh Kumar",
  "language": "hi",
  "state": "Maharashtra",
  "district": "Nagpur"
}

Response: 200 OK
{
  "userId": "uuid",
  "token": "jwt_token",
  "refreshToken": "refresh_token",
  "expiresIn": 3600
}
```

**POST /api/v1/auth/verify-otp**
```json
Request:
{
  "phone": "+919876543210",
  "otp": "123456"
}

Response: 200 OK
{
  "verified": true,
  "token": "jwt_token"
}
```

---

### Advisory APIs

**POST /api/v1/advisory/query**
```json
Request:
Headers: { "Authorization": "Bearer <token>" }
{
  "query": "मेरे गेहूं की फसल पर पीले धब्बे हैं, क्या करूं?",
  "language": "hi",
  "context": {
    "cropType": "wheat",
    "stage": "flowering"
  }
}

Response: 200 OK
{
  "response": "आपके गेहूं की फसल पर येलो रस्ट (पीली रतुआ) रोग के लक्षण दिख रहे हैं...",
  "language": "hi",
  "confidence": 0.89,
  "relatedResources": [
    {
      "title": "Yellow Rust Treatment Guide",
      "url": "https://..."
    }
  ],
  "queryId": "uuid"
}
```

**POST /api/v1/advisory/detect-disease**
```json
Request:
Headers: { "Authorization": "Bearer <token>" }
Content-Type: multipart/form-data
{
  "image": <file>,
  "cropType": "wheat",
  "location": {
    "latitude": 21.1458,
    "longitude": 79.0882
  }
}

Response: 200 OK
{
  "disease": {
    "name": "Yellow Rust",
    "nameHi": "पीली रतुआ",
    "confidence": 0.92,
    "severity": "medium"
  },
  "treatment": {
    "immediate": [
      "Apply Propiconazole 25% EC @ 500ml per hectare",
      "Spray early morning or evening"
    ],
    "organic": [
      "Neem oil spray (5ml per liter water)",
      "Remove infected leaves"
    ],
    "preventive": [
      "Use resistant varieties",
      "Proper crop rotation"
    ]
  },
  "estimatedLoss": "15-20% if untreated",
  "detectionId": "uuid"
}
```

---

### Market Price APIs

**GET /api/v1/markets/prices**
```
Request:
GET /api/v1/markets/prices?commodity=wheat&location=nagpur&radius=50
Headers: { "Authorization": "Bearer <token>" }

Response: 200 OK
{
  "commodity": "wheat",
  "location": "Nagpur, Maharashtra",
  "markets": [
    {
      "name": "Kalamna Mandi",
      "distance": 5.2,
      "price": 2150,
      "unit": "quintal",
      "date": "2026-02-15",
      "trend": "up",
      "changeAmount": 50,
      "changePercent": 2.38,
      "arrivals": 250,
      "contact": "+91-712-1234567"
    },
    {
      "name": "Kalmeshwar APMC",
      "distance": 32.8,
      "price": 2100,
      "unit": "quintal",
      "date": "2026-02-15",
      "trend": "stable",
      "changeAmount": 0
    }
  ],
  "recommendation": {
    "bestMarket": "Kalamna Mandi",
    "bestTime": "Wait 3-5 days for higher prices",
    "confidence": 0.76,
    "reasoning": "Price trend indicates upward movement based on seasonal demand"
  }
}
```

**GET /api/v1/markets/forecast**
```
Request:
GET /api/v1/markets/forecast?commodity=wheat&market=nagpur&days=7

Response: 200 OK
{
  "commodity": "wheat",
  "market": "Nagpur",
  "currentPrice": 2150,
  "forecast": [
    { "date": "2026-02-16", "predictedPrice": 2160, "confidence": 0.82 },
    { "date": "2026-02-17", "predictedPrice": 2180, "confidence": 0.78 },
    { "date": "2026-02-18", "predictedPrice": 2200, "confidence": 0.74 }
  ],
  "trend": "upward",
  "recommendation": "Consider selling in 5-7 days for optimal returns",
  "factors": [
    "Seasonal demand increase",
    "Reduced supply from neighboring states",
    "Government procurement expected to start"
  ]
}
```

---

### Weather APIs

**GET /api/v1/weather/forecast**
```
Request:
GET /api/v1/weather/forecast?latitude=21.1458&longitude=79.0882&days=7
Headers: { "Authorization": "Bearer <token>" }

Response: 200 OK
{
  "location": {
    "name": "Nagpur, Maharashtra",
    "latitude": 21.1458,
    "longitude": 79.0882
  },
  "current": {
    "temperature": 32.5,
    "feelsLike": 35.2,
    "humidity": 45,
    "windSpeed": 12,
    "condition": "Partly Cloudy",
    "timestamp": "2026-02-15T14:30:00Z"
  },
  "daily": [
    {
      "date": "2026-02-15",
      "tempMax": 34,
      "tempMin": 18,
      "rainfall": 0,
      "humidity": 45,
      "condition": "Sunny",
      "cropAdvisory": "Good day for pesticide application"
    }
  ],
  "alerts": [
    {
      "type": "heat_wave",
      "severity": "moderate",
      "message": "Heat wave conditions likely for next 3 days",
      "startTime": "2026-02-16T12:00:00Z",
      "endTime": "2026-02-18T18:00:00Z",
      "advisory": "Ensure adequate irrigation and avoid midday farm work"
    }
  ]
}
```

---

## User Interface Design

### Mobile App (PWA) - Key Screens

**1. Home Dashboard**
```
┌─────────────────────────────────┐
│ 🌾 KrishiMitra        [Settings]│
├─────────────────────────────────┤
│ Welcome, Ramesh! 👋              │
│ Nagpur, Maharashtra             │
│ ☀️ 32°C | Partly Cloudy          │
├─────────────────────────────────┤
│ ┌─────────────────────────────┐ │
│ │  💬 Ask Anything             │ │
│ │  "कैसे बढ़ाएं गेहूं की पैदावार" │ │
│ └─────────────────────────────┘ │
├─────────────────────────────────┤
│ Quick Actions                   │
│ ┌────────┐ ┌────────┐ ┌───────┐│
│ │📸 Crop │ │💰 Price│ │🌦️ Alert││
│ │  Check │ │  Info  │ │  Setup ││
│ └────────┘ └────────┘ └───────┘│
├─────────────────────────────────┤
│ Your Crops (2 active)           │
│ ┌─────────────────────────────┐ │
│ │ 🌾 Wheat (2 acres)           │ │
│ │ Stage: Flowering             │ │
│ │ Days to harvest: 45          │ │
│ │ [View Details]               │ │
│ └─────────────────────────────┘ │
│                                 │
│ Latest Market Prices            │
│ Wheat: ₹2,150 ↗️ (+50)          │
│ Soybean: ₹4,800 ↘️ (-100)       │
└─────────────────────────────────┘
```

**2. Disease Detection Screen**
```
┌─────────────────────────────────┐
│ ← Crop Disease Detection        │
├─────────────────────────────────┤
│                                 │
│  ┌─────────────────────────┐   │
│  │                         │   │
│  │   [Camera Preview]      │   │
│  │                         │   │
│  │   📸 Tap to capture     │   │
│  │   or                    │   │
│  │   📁 Upload from gallery│   │
│  └─────────────────────────┘   │
│                                 │
│  Select Crop Type:              │
│  ⦿ Wheat   ○ Rice   ○ Cotton   │
│  ○ Soybean ○ Other             │
│                                 │
│  [Analyze Image]                │
│                                 │
│  Recent Detections:             │
│  • Yellow Rust (2 days ago)     │
│  • Aphid Infestation (5 days)  │
└─────────────────────────────────┘
```

**3. Results Screen (After Detection)**
```
┌─────────────────────────────────┐
│ ← Detection Results              │
├─────────────────────────────────┤
│  ┌─────────────────────────┐   │
│  │ [Analyzed Crop Image]   │   │
│  └─────────────────────────┘   │
│                                 │
│  ⚠️ Yellow Rust Detected         │
│  Confidence: 92%                │
│  Severity: Medium               │
│                                 │
├─────────────────────────────────┤
│  💊 Treatment                    │
│  ┌─────────────────────────┐   │
│  │ Immediate Action:        │   │
│  │ • Apply Propiconazole    │   │
│  │   25% EC @ 500ml/hectare│   │
│  │ • Spray morning/evening  │   │
│  │                          │   │
│  │ 🌿 Organic Alternative:  │   │
│  │ • Neem oil spray         │   │
│  │   (5ml per liter)        │   │
│  │                          │   │
│  │ [Detailed Guide]         │   │
│  └─────────────────────────┘   │
│                                 │
│  Estimated Loss if Untreated:   │
│  15-20% yield reduction         │
│                                 │
│  [Save] [Share] [Get Help]      │
└─────────────────────────────────┘
```

**4. Market Prices Screen**
```
┌─────────────────────────────────┐
│ ← Market Prices                  │
├─────────────────────────────────┤
│  Search: [Wheat          ] 🔍   │
│  📍 Nagpur (50 km radius)        │
├─────────────────────────────────┤
│  Wheat - Today's Prices          │
│                                 │
│  ┌─────────────────────────┐   │
│  │ 🏪 Kalamna Mandi         │   │
│  │ ₹2,150/quintal ↗️ +₹50   │   │
│  │ 5.2 km away              │   │
│  │ Arrivals: 250 quintals   │   │
│  │ [View Details] [Navigate]│   │
│  └─────────────────────────┘   │
│                                 │
│  ┌─────────────────────────┐   │
│  │ 🏪 Kalmeshwar APMC       │   │
│  │ ₹2,100/quintal →         │   │
│  │ 32.8 km away             │   │
│  │ [View Details] [Navigate]│   │
│  └─────────────────────────┘   │
│                                 │
│  💡 AI Recommendation:           │
│  "Wait 3-5 days for better      │
│  prices. Trend shows upward     │
│  movement due to seasonal       │
│  demand increase."              │
│                                 │
│  [7-Day Price Forecast]         │
└─────────────────────────────────┘
```

### Design System

**Color Palette:**
- Primary: `#2C5F2D` (Forest Green) - Agriculture theme
- Secondary: `#97BC62` (Moss Green) - Supporting elements
- Accent: `#F96167` (Coral) - Alerts, CTAs
- Background: `#F5F5F5` (Light Gray)
- Text: `#333333` (Dark Gray)
- Success: `#4CAF50`
- Warning: `#FF9800`
- Error: `#F44336`

**Typography:**
- Headers: Roboto Bold, 24-32px
- Body: Roboto Regular, 16px
- Captions: Roboto Light, 14px
- Indian language support: Noto Sans Devanagari, Noto Sans Tamil, etc.

**Spacing:**
- Base unit: 8px
- Small: 8px, Medium: 16px, Large: 24px, XL: 32px

**Components:**
- Cards: Rounded corners (8px), shadow elevation 2
- Buttons: Primary (filled), Secondary (outlined), Ghost (text only)
- Icons: Material Icons, 24px standard size
- Input fields: Outlined style, 48px height for touch targets

---

## Security Architecture

### Authentication & Authorization

**JWT Token Structure:**
```json
{
  "userId": "uuid",
  "phone": "+919876543210",
  "role": "farmer",
  "language": "hi",
  "iat": 1676458800,
  "exp": 1676462400
}
```

**Security Measures:**
1. **Password/OTP:**
   - OTP-based authentication (no passwords for simplicity)
   - OTP expires in 5 minutes
   - Rate limiting: 3 OTP requests per hour per phone number

2. **Token Management:**
   - Access token: 1 hour expiry
   - Refresh token: 7 days expiry
   - Tokens stored in httpOnly cookies (web) or secure storage (mobile)

3. **API Security:**
   - Rate limiting: 100 requests per minute per user
   - API key validation for internal services
   - CORS policy: Whitelist only known domains
   - Input validation and sanitization (Joi schemas)

4. **Data Protection:**
   - Encryption at rest: AES-256
   - Encryption in transit: TLS 1.3
   - No storage of sensitive personal data
   - PII anonymization in logs

### Privacy Compliance

**Data Collection:**
- Minimal data collection (name, phone, location, crops only)
- Explicit consent during registration
- Opt-in for marketing communications

**User Rights:**
- Right to access personal data
- Right to delete account and all data
- Right to export data in JSON format
- Right to opt-out of analytics

**Data Retention:**
- User profiles: Retained until account deletion
- Query history: 90 days
- Uploaded images: 30 days, then auto-deleted
- Logs: 7 days for debugging, anonymized after

---

## Deployment Strategy

### Infrastructure (AWS)

```
Production Environment:
├── VPC (10.0.0.0/16)
│   ├── Public Subnet (10.0.1.0/24)
│   │   ├── ALB (Application Load Balancer)
│   │   └── NAT Gateway
│   ├── Private Subnet (10.0.2.0/24)
│   │   ├── ECS/EKS Cluster
│   │   │   ├── Web API Service (3 replicas)
│   │   │   ├── ML Service (2 replicas, GPU instances)
│   │   │   └── Bot Service (2 replicas)
│   │   └── ElastiCache (Redis Cluster)
│   └── Database Subnet (10.0.3.0/24)
│       └── RDS PostgreSQL (Multi-AZ)
├── S3 Buckets
│   ├── Static Assets (CloudFront CDN)
│   ├── User Uploads (temp storage)
│   └── ML Models
├── Lambda Functions
│   ├── Weather Alert Scheduler
│   ├── Price Data ETL
│   └── Image Cleanup (30-day)
└── SQS/SNS
    ├── SMS Queue
    ├── WhatsApp Queue
    └── Alert Topic
```

### Deployment Pipeline (CI/CD)

```
GitHub Repository
    ↓
GitHub Actions (CI)
    ├── Run tests (Jest, PyTest)
    ├── Lint code (ESLint, Pylint)
    ├── Security scan (Snyk)
    └── Build Docker images
    ↓
Amazon ECR (Container Registry)
    ↓
AWS CodeDeploy
    ↓
ECS/EKS Cluster
    ├── Blue-Green Deployment
    ├── Health Checks
    └── Auto-rollback on failure
```

### Monitoring & Observability

**Application Monitoring:**
- CloudWatch for logs aggregation
- Prometheus + Grafana for metrics
- AWS X-Ray for distributed tracing
- Sentry for error tracking

**Key Metrics:**
```
SLIs (Service Level Indicators):
- API response time (p50, p95, p99)
- Error rate (< 1%)
- Uptime (> 99.5%)
- ML model inference time (< 5s)

Business Metrics:
- Daily Active Users (DAU)
- Queries per user per day
- Disease detection accuracy
- Price prediction accuracy
- SMS/WhatsApp delivery success rate
```

**Alerting Rules:**
- API error rate > 5% for 5 minutes → PagerDuty alert
- Response time p95 > 5 seconds → Slack notification
- Database CPU > 80% → Auto-scaling trigger
- Disk usage > 85% → Email alert

---

## Testing Strategy

### Unit Testing

**Backend (Node.js/Python):**
```javascript
// Example: Jest test for price service
describe('PriceService', () => {
  describe('fetchMarketPrices', () => {
    it('should return prices for valid commodity and location', async () => {
      const prices = await priceService.fetchMarketPrices('wheat', 'nagpur');
      
      expect(prices).toBeDefined();
      expect(prices).toHaveLength(greaterThan(0));
      expect(prices[0]).toHaveProperty('market');
      expect(prices[0]).toHaveProperty('price');
      expect(prices[0].price).toBeGreaterThan(0);
    });

    it('should handle API errors gracefully', async () => {
      mockAgmarknetAPI.mockRejectedValue(new Error('API timeout'));
      
      await expect(
        priceService.fetchMarketPrices('wheat', 'invalid')
      ).rejects.toThrow('Unable to fetch prices');
    });
  });
});
```

**Frontend (React):**
```javascript
// Example: React Testing Library
describe('DiseaseDetectionScreen', () => {
  it('should upload image and show results', async () => {
    const { getByText, getByTestId } = render(<DiseaseDetectionScreen />);
    
    const file = new File(['dummy'], 'crop.jpg', { type: 'image/jpeg' });
    const input = getByTestId('image-upload');
    
    fireEvent.change(input, { target: { files: [file] } });
    fireEvent.click(getByText('Analyze Image'));
    
    await waitFor(() => {
      expect(getByText('Yellow Rust Detected')).toBeInTheDocument();
    });
  });
});
```

### Integration Testing

**API Integration:**
```javascript
describe('Advisory API Integration', () => {
  it('should process text query end-to-end', async () => {
    const response = await request(app)
      .post('/api/v1/advisory/query')
      .set('Authorization', `Bearer ${testToken}`)
      .send({
        query: 'How to control aphids on wheat?',
        language: 'en'
      });
    
    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty('response');
    expect(response.body.response).toContain('aphid');
  });
});
```

### Performance Testing

**Load Testing (k6):**
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'], // 95% of requests under 2s
    http_req_failed: ['rate<0.01'],    // Less than 1% errors
  },
};

export default function () {
  let response = http.post(
    'https://api.krishimitra.com/v1/advisory/query',
    JSON.stringify({
      query: 'How to increase wheat yield?',
      language: 'en'
    }),
    {
      headers: {
        'Authorization': `Bearer ${__ENV.TEST_TOKEN}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 2s': (r) => r.timings.duration < 2000,
  });
  
  sleep(1);
}
```

### End-to-End Testing (Playwright)

```javascript
test('farmer can detect disease and get treatment', async ({ page }) => {
  // Login
  await page.goto('https://app.krishimitra.com');
  await page.fill('#phone', '+919876543210');
  await page.click('button:has-text("Send OTP")');
  await page.fill('#otp', '123456');
  await page.click('button:has-text("Verify")');
  
  // Navigate to disease detection
  await page.click('text=Crop Check');
  
  // Upload image
  await page.setInputFiles('input[type="file"]', 'test/fixtures/wheat_rust.jpg');
  await page.click('button:has-text("Analyze Image")');
  
  // Verify results
  await expect(page.locator('text=Yellow Rust Detected')).toBeVisible();
  await expect(page.locator('text=Treatment')).toBeVisible();
  await expect(page.locator('text=Propiconazole')).toBeVisible();
});
```

---

## Scalability Considerations

### Horizontal Scaling

**Auto-Scaling Policies:**
```yaml
# ECS Service Auto-Scaling
WebAPIService:
  MinTasks: 2
  MaxTasks: 20
  TargetCPUUtilization: 70%
  TargetMemoryUtilization: 80%
  ScaleOutCooldown: 60s
  ScaleInCooldown: 300s

MLService:
  MinTasks: 1
  MaxTasks: 10
  TargetCPUUtilization: 80%
  ScaleOutCooldown: 120s  # Slower scale-out due to GPU warm-up
```

### Database Optimization

**PostgreSQL:**
- Read replicas for query-heavy operations
- Connection pooling (max 100 connections per instance)
- Partitioning for query_history table (by month)
- Indexes on frequently queried columns

**Redis:**
- Cluster mode for sharding
- Separate cache for different data types (prices, sessions, weather)
- LRU eviction policy
- Persistence: RDB snapshots every 5 minutes

### Caching Strategy

```
Browser Cache (1 day)
    ↓
CDN Cache (CloudFront, 1 hour)
    ↓
Redis Cache (1 hour)
    ↓
Database / External API
```

**Cache Invalidation:**
- Market prices: Time-based (1 hour TTL)
- Weather data: Time-based (15 minutes TTL)
- User profile: Event-based (on update)
- ML model predictions: Result-based (cache for 24 hours)

---

## Disaster Recovery

### Backup Strategy

**Database Backups:**
- Automated daily snapshots (RDS)
- Point-in-time recovery (up to 7 days)
- Cross-region replication (DR region: Mumbai → Singapore)

**Application Backups:**
- Docker images versioned in ECR
- Infrastructure as Code (Terraform state in S3)
- Configuration in AWS Systems Manager Parameter Store

**Recovery Time Objective (RTO):** 1 hour  
**Recovery Point Objective (RPO):** 1 hour

### Failover Procedures

1. **Database Failover:** Automatic (Multi-AZ RDS)
2. **Application Failover:** DNS-based routing to DR region
3. **Data Sync:** Continuous replication to DR region

---

## Development Workflow

### Local Development Setup

```bash
# Prerequisites
- Node.js 18+
- Python 3.10+
- Docker & Docker Compose
- AWS CLI (configured)

# Clone repository
git clone https://github.com/your-org/krishimitra.git
cd krishimitra

# Install dependencies
npm install
pip install -r requirements.txt

# Setup environment variables
cp .env.example .env
# Edit .env with your API keys

# Start local services (Docker Compose)
docker-compose up -d postgres redis mongodb

# Run database migrations
npm run migrate

# Start development servers
npm run dev:api      # Web API on port 3000
npm run dev:ml       # ML Service on port 8000
npm run dev:bot      # Bot Service on port 3001
npm run dev:frontend # React app on port 3002
```

### Code Organization

```
krishimitra/
├── services/
│   ├── web-api/
│   │   ├── src/
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── models/
│   │   │   ├── middleware/
│   │   │   └── utils/
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── ml-service/
│   │   ├── app/
│   │   │   ├── models/
│   │   │   ├── services/
│   │   │   └── utils/
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   └── bot-service/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/
│   │   └── utils/
│   └── public/
├── infrastructure/
│   ├── terraform/
│   ├── kubernetes/
│   └── docker-compose.yml
├── docs/
│   ├── api/
│   ├── architecture/
│   └── runbooks/
└── scripts/
```

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Status:** Draft - For AI4Bharat Hackathon Submission
