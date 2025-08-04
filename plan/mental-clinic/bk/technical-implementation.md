# メンタルクリニック向けシステム - 技術実装計画

## 概要
予約管理システムの技術仕様、開発方針、実装詳細、およびスケーラビリティ対応戦略

---

## 1. システムアーキテクチャ

### 1.1 全体構成

#### Phase 1: モノリシック構成（MVP）
```
┌─────────────────────────────────────┐
│              Frontend              │
│     React.js + TypeScript          │
│        PWA対応                     │
└─────────────────────────────────────┘
                  │ HTTPS
┌─────────────────────────────────────┐
│            API Gateway             │
│         (AWS ALB + WAF)            │
└─────────────────────────────────────┘
                  │
┌─────────────────────────────────────┐
│            Backend API             │
│      Node.js + Express             │
│         (ECS Container)            │
└─────────────────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
┌───▼───┐   ┌────▼────┐   ┌────▼────┐
│  RDS  │   │  Redis  │   │   S3    │
│(PostgreSQL)│  (Cache)│   │(Storage)│
└───────┘   └─────────┘   └─────────┘
```

#### Phase 2: マイクロサービス移行準備
```
┌─────────────────────────────────────┐
│            API Gateway             │
│       (AWS API Gateway)            │
└─────────────────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
┌───▼─────┐   ┌──▼──────┐   ┌──▼────────┐
│Reservation│   │Patient  │   │Notification│
│  Service  │   │ Service │   │  Service   │
└───────────┘   └─────────┘   └────────────┘
```

### 1.2 技術スタック選定

#### フロントエンド
```
【コア技術】
- React.js 18+ : コンポーネントベース、豊富なエコシステム
- TypeScript : 型安全性、開発効率向上
- Vite : 高速ビルド、HMR対応
- React Router : SPA ルーティング

【UI/UX】
- Chakra UI : アクセシビリティ対応、カスタマイズ性
- React Hook Form : フォーム管理、バリデーション
- React Query : サーバー状態管理、キャッシュ
- Framer Motion : アニメーション、UX向上

【PWA対応】
- Workbox : サービスワーカー管理
- Web App Manifest : アプリライクな体験
- Offline対応 : 基本機能のオフライン動作
```

#### バックエンド
```
【コア技術】
- Node.js 18+ : JavaScript統一、豊富なライブラリ
- Express.js : 軽量、柔軟なWebフレームワーク
- TypeScript : フロント・バック統一言語
- Prisma : 型安全なORM、マイグレーション管理

【認証・セキュリティ】
- JWT : ステートレス認証
- bcrypt : パスワードハッシュ化
- helmet : セキュリティヘッダー
- rate-limiter : API レート制限

【バリデーション・テスト】
- Joi/Zod : スキーマバリデーション
- Jest : 単体テスト
- Supertest : API テスト
- Artillery : 負荷テスト
```

#### データベース
```
【メインDB】
- PostgreSQL 14+ : ACID準拠、JSON対応、拡張性
- 暗号化: AES-256 (rest), TLS 1.3 (transit)
- バックアップ: 日次フル、PITRリカバリ

【キャッシュ】
- Redis 7+ : セッション管理、クエリキャッシュ
- Redis Cluster : 高可用性、スケーラビリティ

【データ構造】
- JSONB活用 : 柔軟な患者データ格納
- パーティショニング : 日付ベースでのデータ分割
- インデックス最適化 : クエリパフォーマンス向上
```

#### インフラ・DevOps
```
【クラウド】
- AWS : 医療対応、豊富なサービス
- Multi-AZ : 高可用性構成
- VPC : ネットワーク分離、セキュリティ

【コンテナ】
- Docker : アプリケーションコンテナ化
- AWS ECS : マネージドコンテナオーケストレーション
- Fargate : サーバーレスコンテナ実行

【CI/CD】
- GitHub Actions : ソースコード連携、自動化
- Docker Registry : イメージ管理
- Blue-Green Deployment : ゼロダウンタイム配信
```

---

## 2. データベース設計

### 2.1 論理データモデル

#### エンティティ関係図（主要テーブル）
```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│   Clinics   │      │  Staff_Users │      │  App_Users  │
│             │─────▶│              │      │             │
│ - id        │ 1:N  │ - id         │      │ - id        │
│ - name      │      │ - clinic_id  │      │ - email     │
│ - address   │      │ - role       │      │ - password  │
│ - phone     │      │ - email      │      │ - role      │
└─────────────┘      └──────────────┘      └─────────────┘
       │                                           │
       │ 1:N                                   1:N │
       ▼                                           ▼
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│  Patients   │◀────▶│ Appointments │◀────▶│ TimeSlots   │
│             │ N:M  │              │ N:1  │             │
│ - id        │      │ - id         │      │ - id        │
│ - clinic_id │      │ - patient_id │      │ - clinic_id │
│ - name      │      │ - timeslot_id│      │ - date      │
│ - phone     │      │ - status     │      │ - start_time│
│ - symptoms  │      │ - type       │      │ - end_time  │
│ - history   │      │ - notes      │      │ - is_booked │
└─────────────┘      └──────────────┘      └─────────────┘
```

#### 詳細テーブル設計
```sql
-- クリニック情報
CREATE TABLE clinics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    address TEXT,
    phone VARCHAR(20),
    email VARCHAR(255),
    settings JSONB DEFAULT '{}',
    subscription_plan VARCHAR(50) DEFAULT 'basic',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 患者情報（プライバシー配慮）
CREATE TABLE patients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clinic_id UUID REFERENCES clinics(id),
    patient_code VARCHAR(20) UNIQUE, -- 匿名化ID
    encrypted_name TEXT, -- 暗号化された氏名
    encrypted_phone TEXT, -- 暗号化された電話番号
    encrypted_email TEXT, -- 暗号化されたメール
    date_of_birth DATE,
    gender VARCHAR(10),
    symptoms JSONB DEFAULT '[]',
    medical_history JSONB DEFAULT '{}',
    preferences JSONB DEFAULT '{}',
    last_visit TIMESTAMP WITH TIME ZONE,
    no_show_count INTEGER DEFAULT 0,
    no_show_risk_score DECIMAL(3,2) DEFAULT 0.5,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 予約情報
CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clinic_id UUID REFERENCES clinics(id),
    patient_id UUID REFERENCES patients(id),
    appointment_date DATE NOT NULL,
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    appointment_type VARCHAR(20) NOT NULL, -- 'initial', 'follow_up', 'emergency'
    status VARCHAR(20) DEFAULT 'scheduled', -- 'scheduled', 'confirmed', 'completed', 'cancelled', 'no_show'
    duration_minutes INTEGER DEFAULT 30,
    notes TEXT,
    reminder_sent JSONB DEFAULT '{}',
    created_by UUID REFERENCES app_users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- インデックス最適化
    INDEX idx_appointments_clinic_date (clinic_id, appointment_date),
    INDEX idx_appointments_patient (patient_id),
    INDEX idx_appointments_status (status)
);

-- 予約枠管理
CREATE TABLE time_slots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clinic_id UUID REFERENCES clinics(id),
    day_of_week INTEGER NOT NULL, -- 0=Sunday, 1=Monday, ...
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    slot_type VARCHAR(20) DEFAULT 'regular', -- 'regular', 'emergency', 'blocked'
    max_appointments INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 2.2 データ暗号化戦略

#### 暗号化レベル
```
【Level 1: 完全暗号化】
- 患者氏名、住所、電話番号
- メールアドレス
- 診療メモの詳細

【Level 2: ハッシュ化】
- パスワード (bcrypt)
- 検索用キー (HMAC)

【Level 3: マスキング】
- ログファイル出力時
- 開発環境でのテストデータ
- 監査ログでの個人情報部分
```

#### 暗号化実装
```javascript
// 暗号化ユーティリティ
class EncryptionService {
  private key: Buffer;
  
  constructor() {
    this.key = crypto.scryptSync(process.env.ENCRYPTION_KEY, 'salt', 32);
  }
  
  encrypt(text: string): string {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipher('aes-256-cbc', this.key);
    cipher.setAutoPadding(true);
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    return iv.toString('hex') + ':' + encrypted;
  }
  
  decrypt(encryptedText: string): string {
    const [ivHex, encrypted] = encryptedText.split(':');
    const iv = Buffer.from(ivHex, 'hex');
    const decipher = crypto.createDecipher('aes-256-cbc', this.key);
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}
```

---

## 3. API設計

### 3.1 RESTful API 設計原則

#### エンドポイント構造
```
【リソースベースURL】
GET    /api/v1/clinics/{clinicId}/appointments
POST   /api/v1/clinics/{clinicId}/appointments
PUT    /api/v1/clinics/{clinicId}/appointments/{appointmentId}
DELETE /api/v1/clinics/{clinicId}/appointments/{appointmentId}

【フィルタリング・ソート】
GET /api/v1/appointments?date=2025-08-01&status=scheduled&sort=start_time

【ページネーション】
GET /api/v1/appointments?page=1&limit=20&cursor=eyJpZCI6IjEyMyJ9
```

#### レスポンス形式統一
```json
{
  "success": true,
  "data": {
    "appointments": [...],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "hasNext": true,
      "cursor": "eyJpZCI6IjEyMyJ9"
    }
  },
  "meta": {
    "timestamp": "2025-08-02T10:30:00Z",
    "version": "1.0",
    "requestId": "req_123456789"
  }
}
```

### 3.2 主要API仕様

#### 予約管理API
```typescript
// 予約作成
POST /api/v1/appointments
interface CreateAppointmentRequest {
  patientId: string;
  appointmentDate: string; // ISO 8601
  startTime: string; // HH:mm
  appointmentType: 'initial' | 'follow_up' | 'emergency';
  notes?: string;
  reminderPreferences?: {
    email: boolean;
    sms: boolean;
    timings: number[]; // 通知タイミング（時間前）
  };
}

interface CreateAppointmentResponse {
  appointment: {
    id: string;
    patientId: string;
    appointmentDate: string;
    startTime: string;
    endTime: string;
    status: 'scheduled';
    confirmationCode: string;
  };
}

// 予約検索・フィルタリング
GET /api/v1/appointments
interface GetAppointmentsQuery {
  date?: string; // YYYY-MM-DD
  dateRange?: {
    start: string;
    end: string;
  };
  status?: 'scheduled' | 'confirmed' | 'completed' | 'cancelled' | 'no_show';
  patientId?: string;
  appointmentType?: string;
  sort?: 'date' | 'created_at' | 'patient_name';
  order?: 'asc' | 'desc';
}
```

#### 患者管理API
```typescript
// 患者登録（プライバシー配慮）
POST /api/v1/patients
interface CreatePatientRequest {
  name: string; // フロントエンドで暗号化
  phone?: string;
  email?: string;
  dateOfBirth?: string;
  gender?: 'male' | 'female' | 'other' | 'prefer_not_to_say';
  symptoms?: string[];
  preferences?: {
    preferredTimes: string[];
    reminderMethod: 'email' | 'sms' | 'both';
    anonymousDisplay: boolean;
  };
}

// 患者検索（匿名化対応）
GET /api/v1/patients/search
interface SearchPatientsQuery {
  query: string; // 患者コードまたは暗号化されたキーワード
  limit?: number;
  anonymousOnly?: boolean;
}
```

#### No-show予測API
```typescript
// No-show リスク予測
GET /api/v1/appointments/{appointmentId}/no-show-risk
interface NoShowRiskResponse {
  appointmentId: string;
  riskScore: number; // 0.0 - 1.0
  riskLevel: 'low' | 'medium' | 'high';
  factors: {
    historicalPattern: number;
    appointmentTiming: number;
    weatherFactor: number;
    patientCharacteristics: number;
  };
  recommendations: {
    reminderTiming: number[];
    alternativeSlots?: string[];
    preventiveActions: string[];
  };
}
```

### 3.3 リアルタイム通信

#### WebSocket接続
```typescript
// WebSocket イベント設計
interface WebSocketEvents {
  // 予約状況変更
  'appointment:created': AppointmentCreatedEvent;
  'appointment:updated': AppointmentUpdatedEvent;
  'appointment:cancelled': AppointmentCancelledEvent;
  
  // リアルタイム通知
  'notification:reminder': ReminderNotificationEvent;
  'notification:alert': AlertNotificationEvent;
  
  // システム状態
  'system:maintenance': MaintenanceNotificationEvent;
  'system:update': SystemUpdateEvent;
}

// 実装例
class AppointmentWebSocketHandler {
  constructor(private io: Server) {
    this.setupEventHandlers();
  }
  
  private setupEventHandlers() {
    this.io.on('connection', (socket) => {
      // クリニック別のルームに参加
      socket.on('join:clinic', (clinicId: string) => {
        socket.join(`clinic:${clinicId}`);
      });
      
      // 予約状況の購読
      socket.on('subscribe:appointments', (date: string) => {
        socket.join(`appointments:${date}`);
      });
    });
  }
  
  // 予約作成時の通知
  notifyAppointmentCreated(clinicId: string, appointment: Appointment) {
    this.io.to(`clinic:${clinicId}`).emit('appointment:created', {
      appointment,
      timestamp: new Date().toISOString()
    });
  }
}
```

---

## 4. セキュリティ実装

### 4.1 認証・認可システム

#### JWT実装
```typescript
interface JWTPayload {
  userId: string;
  clinicId: string;
  role: 'admin' | 'staff' | 'doctor';
  permissions: string[];
  exp: number;
  iat: number;
}

class AuthService {
  private readonly JWT_SECRET = process.env.JWT_SECRET!;
  private readonly JWT_EXPIRES = '24h';
  
  generateTokens(user: User): { accessToken: string; refreshToken: string } {
    const payload: JWTPayload = {
      userId: user.id,
      clinicId: user.clinicId,
      role: user.role,
      permissions: this.getPermissions(user.role),
      exp: Math.floor(Date.now() / 1000) + (24 * 60 * 60), // 24時間
      iat: Math.floor(Date.now() / 1000)
    };
    
    const accessToken = jwt.sign(payload, this.JWT_SECRET);
    const refreshToken = this.generateRefreshToken(user.id);
    
    return { accessToken, refreshToken };
  }
  
  private getPermissions(role: string): string[] {
    const permissions: Record<string, string[]> = {
      admin: ['read:all', 'write:all', 'delete:all', 'manage:users'],
      doctor: ['read:patients', 'write:appointments', 'read:reports'],
      staff: ['read:appointments', 'write:appointments', 'read:patients']
    };
    
    return permissions[role] || [];
  }
}
```

#### ロールベース認可
```typescript
// デコレータによる認可制御
function RequirePermission(permission: string) {
  return (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(req: AuthenticatedRequest, res: Response, next: NextFunction) {
      if (!req.user.permissions.includes(permission)) {
        return res.status(403).json({ error: 'Insufficient permissions' });
      }
      
      return originalMethod.call(this, req, res, next);
    };
  };
}

// 使用例
class AppointmentController {
  @RequirePermission('write:appointments')
  async createAppointment(req: AuthenticatedRequest, res: Response) {
    // 予約作成処理
  }
  
  @RequirePermission('read:patients')
  async getPatientAppointments(req: AuthenticatedRequest, res: Response) {
    // 患者予約取得処理
  }
}
```

### 4.2 入力値検証・サニタイゼーション

#### スキーマバリデーション
```typescript
import Joi from 'joi';

const appointmentCreateSchema = Joi.object({
  patientId: Joi.string().uuid().required(),
  appointmentDate: Joi.date().iso().min('now').required(),
  startTime: Joi.string().pattern(/^([01]?[0-9]|2[0-3]):[0-5][0-9]$/).required(),
  appointmentType: Joi.string().valid('initial', 'follow_up', 'emergency').required(),
  notes: Joi.string().max(1000).optional().allow(''),
  reminderPreferences: Joi.object({
    email: Joi.boolean().default(true),
    sms: Joi.boolean().default(false),
    timings: Joi.array().items(Joi.number().min(1).max(168)).default([24, 2])
  }).optional()
});

// ミドルウェア実装
function validateSchema(schema: Joi.ObjectSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    const { error, value } = schema.validate(req.body);
    
    if (error) {
      return res.status(400).json({
        success: false,
        error: 'Validation failed',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message
        }))
      });
    }
    
    req.body = value; // サニタイズされた値を設定
    next();
  };
}
```

### 4.3 API セキュリティ

#### レート制限
```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';

const rateLimitConfig = {
  // 一般API: 1分間に100リクエスト
  general: rateLimit({
    windowMs: 60 * 1000,
    max: 100,
    store: new RedisStore({
      client: redisClient,
      prefix: 'rl:general:'
    }),
    message: 'Too many requests, please try again later.'
  }),
  
  // 認証API: 1分間に5リクエスト
  auth: rateLimit({
    windowMs: 60 * 1000,
    max: 5,
    store: new RedisStore({
      client: redisClient,
      prefix: 'rl:auth:'
    }),
    message: 'Too many authentication attempts.'
  }),
  
  // 検索API: 1分間に50リクエスト
  search: rateLimit({
    windowMs: 60 * 1000,
    max: 50,
    store: new RedisStore({
      client: redisClient,
      prefix: 'rl:search:'
    })
  })
};
```

#### CORS・セキュリティヘッダー
```typescript
import cors from 'cors';
import helmet from 'helmet';

const securityMiddleware = [
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        imgSrc: ["'self'", "data:", "https:"],
        scriptSrc: ["'self'"],
        connectSrc: ["'self'", "https://api.clinic-system.com"]
      }
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true
    }
  }),
  
  cors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With']
  })
];
```

---

## 5. パフォーマンス最適化

### 5.1 データベース最適化

#### クエリ最適化
```sql
-- インデックス戦略
CREATE INDEX CONCURRENTLY idx_appointments_clinic_date_status 
ON appointments (clinic_id, appointment_date, status) 
WHERE status IN ('scheduled', 'confirmed');

-- パーティショニング（月別）
CREATE TABLE appointments_2025_08 PARTITION OF appointments
FOR VALUES FROM ('2025-08-01') TO ('2025-09-01');

-- 統計情報更新の自動化
CREATE OR REPLACE FUNCTION update_appointment_stats()
RETURNS void AS $$
BEGIN
  -- No-show率の更新
  UPDATE patients SET 
    no_show_risk_score = (
      SELECT COALESCE(
        COUNT(*) FILTER (WHERE status = 'no_show')::decimal / 
        NULLIF(COUNT(*), 0), 0.5
      )
      FROM appointments 
      WHERE patient_id = patients.id 
        AND appointment_date >= CURRENT_DATE - INTERVAL '6 months'
    );
END;
$$ LANGUAGE plpgsql;

-- 定期実行（毎日夜中）
SELECT cron.schedule('update-stats', '0 2 * * *', 'SELECT update_appointment_stats();');
```

#### 接続プール最適化
```typescript
import { Pool } from 'pg';

const dbPool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  
  // 接続プール設定
  min: 5,  // 最小接続数
  max: 20, // 最大接続数
  idleTimeoutMillis: 30000, // アイドルタイムアウト
  connectionTimeoutMillis: 2000, // 接続タイムアウト
  
  // SSL設定
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: false
  } : false
});

// ヘルスチェック
setInterval(async () => {
  try {
    await dbPool.query('SELECT NOW()');
    console.log('Database connection healthy');
  } catch (error) {
    console.error('Database connection failed:', error);
  }
}, 30000);
```

### 5.2 キャッシング戦略

#### Redis活用
```typescript
class CacheService {
  private redis: Redis;
  
  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT || '6379'),
      password: process.env.REDIS_PASSWORD,
      retryDelayOnFailover: 100,
      maxRetriesPerRequest: 3
    });
  }
  
  // 予約データのキャッシュ（5分間）
  async cacheAppointments(clinicId: string, date: string, appointments: any[]) {
    const key = `appointments:${clinicId}:${date}`;
    await this.redis.setex(key, 300, JSON.stringify(appointments));
  }
  
  // キャッシュ取得
  async getCachedAppointments(clinicId: string, date: string): Promise<any[] | null> {
    const key = `appointments:${clinicId}:${date}`;
    const cached = await this.redis.get(key);
    return cached ? JSON.parse(cached) : null;
  }
  
  // キャッシュ無効化
  async invalidateAppointmentCache(clinicId: string, date?: string) {
    if (date) {
      await this.redis.del(`appointments:${clinicId}:${date}`);
    } else {
      const keys = await this.redis.keys(`appointments:${clinicId}:*`);
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
    }
  }
}
```

#### アプリケーションレベルキャッシュ
```typescript
import NodeCache from 'node-cache';

class AppCache {
  private cache: NodeCache;
  
  constructor() {
    this.cache = new NodeCache({
      stdTTL: 600, // デフォルト10分
      checkperiod: 120, // 2分毎にチェック
      useClones: false // パフォーマンス向上
    });
  }
  
  // 診療スケジュール（変更頻度低）
  async getClinicSchedule(clinicId: string): Promise<any> {
    const key = `schedule:${clinicId}`;
    let schedule = this.cache.get(key);
    
    if (!schedule) {
      schedule = await this.fetchClinicScheduleFromDB(clinicId);
      this.cache.set(key, schedule, 3600); // 1時間キャッシュ
    }
    
    return schedule;
  }
  
  // 患者情報（セキュリティ重視、短時間）
  async getPatientInfo(patientId: string): Promise<any> {
    const key = `patient:${patientId}`;
    let patient = this.cache.get(key);
    
    if (!patient) {
      patient = await this.fetchPatientFromDB(patientId);
      this.cache.set(key, patient, 300); // 5分キャッシュ
    }
    
    return patient;
  }
}
```

---

## 6. 監視・ログ・エラーハンドリング

### 6.1 ログ戦略

#### 構造化ログ
```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json(),
    winston.format.printf(({ timestamp, level, message, ...meta }) => {
      return JSON.stringify({
        timestamp,
        level,
        message,
        service: 'clinic-reservation-api',
        environment: process.env.NODE_ENV,
        ...meta
      });
    })
  ),
  defaultMeta: { service: 'clinic-reservation-api' },
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    })
  ]
});

// セキュリティログ
export const securityLogger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ 
      filename: 'logs/security.log',
      maxsize: 10485760, // 10MB
      maxFiles: 5
    })
  ]
});
```

#### 監査ログ
```typescript
interface AuditLogEntry {
  userId: string;
  action: string;
  resource: string;
  resourceId?: string;
  changes?: any;
  ipAddress: string;
  userAgent: string;
  timestamp: Date;
  result: 'success' | 'failure';
  error?: string;
}

class AuditLogger {
  async logPatientAccess(userId: string, patientId: string, action: string, req: Request) {
    const entry: AuditLogEntry = {
      userId,
      action: `patient:${action}`,
      resource: 'patient',
      resourceId: patientId,
      ipAddress: req.ip,
      userAgent: req.get('User-Agent') || '',
      timestamp: new Date(),
      result: 'success'
    };
    
    securityLogger.info('Patient data access', entry);
  }
  
  async logAppointmentChange(userId: string, appointmentId: string, changes: any, req: Request) {
    const entry: AuditLogEntry = {
      userId,
      action: 'appointment:update',
      resource: 'appointment',
      resourceId: appointmentId,
      changes: this.sanitizeChanges(changes),
      ipAddress: req.ip,
      userAgent: req.get('User-Agent') || '',
      timestamp: new Date(),
      result: 'success'
    };
    
    logger.info('Appointment updated', entry);
  }
  
  private sanitizeChanges(changes: any): any {
    // 個人情報を除外してログに記録
    const sensitiveFields = ['name', 'phone', 'email', 'address'];
    const sanitized = { ...changes };
    
    sensitiveFields.forEach(field => {
      if (sanitized[field]) {
        sanitized[field] = '[REDACTED]';
      }
    });
    
    return sanitized;
  }
}
```

### 6.2 エラーハンドリング

#### 統一エラー処理
```typescript
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;
  public readonly errorCode: string;
  
  constructor(
    message: string,
    statusCode: number = 500,
    errorCode: string = 'INTERNAL_ERROR',
    isOperational: boolean = true
  ) {
    super(message);
    this.statusCode = statusCode;
    this.errorCode = errorCode;
    this.isOperational = isOperational;
    
    Error.captureStackTrace(this, this.constructor);
  }
}

// 特定エラークラス
export class ValidationError extends AppError {
  constructor(message: string, field?: string) {
    super(message, 400, 'VALIDATION_ERROR');
    this.name = 'ValidationError';
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    const message = id ? `${resource} with id ${id} not found` : `${resource} not found`;
    super(message, 404, 'NOT_FOUND');
    this.name = 'NotFoundError';
  }
}

// グローバルエラーハンドラー
export const errorHandler = (
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  let appError: AppError;
  
  if (error instanceof AppError) {
    appError = error;
  } else {
    // 予期しないエラー
    logger.error('Unexpected error:', {
      error: error.message,
      stack: error.stack,
      url: req.url,
      method: req.method,
      userId: req.user?.id
    });
    
    appError = new AppError(
      'Internal server error',
      500,
      'INTERNAL_ERROR',
      false
    );
  }
  
  res.status(appError.statusCode).json({
    success: false,
    error: {
      code: appError.errorCode,
      message: appError.message,
      ...(process.env.NODE_ENV === 'development' && { stack: appError.stack })
    },
    meta: {
      timestamp: new Date().toISOString(),
      requestId: req.headers['x-request-id']
    }
  });
};
```

### 6.3 モニタリング

#### ヘルスチェック
```typescript
class HealthService {
  async getHealthStatus(): Promise<any> {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkExternalAPIs()
    ]);
    
    const results = {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      version: process.env.APP_VERSION || '1.0.0',
      uptime: process.uptime(),
      checks: {
        database: this.getCheckResult(checks[0]),
        redis: this.getCheckResult(checks[1]),
        externalAPIs: this.getCheckResult(checks[2])
      }
    };
    
    // いずれかが失敗している場合
    const hasFailure = Object.values(results.checks).some(check => check.status === 'unhealthy');
    if (hasFailure) {
      results.status = 'unhealthy';
    }
    
    return results;
  }
  
  private async checkDatabase() {
    try {
      await dbPool.query('SELECT NOW()');
      return { status: 'healthy', responseTime: Date.now() };
    } catch (error) {
      return { status: 'unhealthy', error: error.message };
    }
  }
  
  private async checkRedis() {
    try {
      const start = Date.now();
      await redisClient.ping();
      return { status: 'healthy', responseTime: Date.now() - start };
    } catch (error) {
      return { status: 'unhealthy', error: error.message };
    }
  }
}
```

#### メトリクス収集
```typescript
import client from 'prom-client';

// カスタムメトリクス
const appointmentCounter = new client.Counter({
  name: 'appointments_total',
  help: 'Total number of appointments created',
  labelNames: ['clinic_id', 'appointment_type']
});

const noShowGauge = new client.Gauge({
  name: 'no_show_rate',
  help: 'Current no-show rate by clinic',
  labelNames: ['clinic_id']
});

const apiDuration = new client.Histogram({
  name: 'api_request_duration_seconds',
  help: 'Duration of API requests in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.1, 0.5, 1, 2, 5, 10]
});

// ミドルウェア
export const metricsMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    apiDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode.toString())
      .observe(duration);
  });
  
  next();
};
```

---

## 7. デプロイメント・DevOps

### 7.1 Docker構成

#### Dockerfile
```dockerfile
# Multi-stage build for production optimization
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Development dependencies for build
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production image
FROM node:18-alpine AS production

# セキュリティ: 非rootユーザー
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

WORKDIR /app

# 必要なファイルのみコピー
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nextjs:nodejs /app/package*.json ./

USER nextjs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

#### docker-compose.yml
```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    
  postgres:
    image: postgres:14-alpine
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped
    
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    restart: unless-stopped
    
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

### 7.2 CI/CD Pipeline

#### GitHub Actions
```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linting
      run: npm run lint
    
    - name: Run type checking
      run: npm run type-check
    
    - name: Run tests
      run: npm run test:coverage
      env:
        DATABASE_URL: postgresql://postgres:test@localhost:5432/test_db
        REDIS_URL: redis://localhost:6379
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      
  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run security audit
      run: npm audit --audit-level high
      
    - name: Run SAST scan
      uses: github/super-linter@v4
      env:
        DEFAULT_BRANCH: main
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        
  deploy:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-1
    
    - name: Build and push Docker image
      run: |
        docker build -t clinic-reservation:${{ github.sha }} .
        docker tag clinic-reservation:${{ github.sha }} ${{ secrets.ECR_REGISTRY }}/clinic-reservation:latest
        docker push ${{ secrets.ECR_REGISTRY }}/clinic-reservation:latest
    
    - name: Deploy to ECS
      run: |
        aws ecs update-service \
          --cluster clinic-reservation-cluster \
          --service clinic-reservation-service \
          --force-new-deployment
```

### 7.3 インフラ構成（Terraform）

#### main.tf
```hcl
# VPC設定
resource "aws_vpc" "clinic_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "clinic-reservation-vpc"
    Environment = var.environment
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "clinic_cluster" {
  name = "clinic-reservation-cluster"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
  
  tags = {
    Environment = var.environment
  }
}

# RDS Instance
resource "aws_db_instance" "clinic_db" {
  identifier             = "clinic-reservation-db"
  allocated_storage      = 20
  max_allocated_storage  = 100
  storage_type           = "gp2"
  engine                 = "postgres"
  engine_version         = "14.9"
  instance_class         = "db.t3.micro"
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  db_subnet_group_name   = aws_db_subnet_group.clinic_db_subnet_group.name
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = false
  final_snapshot_identifier = "clinic-db-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"
  
  encryption = true
  
  tags = {
    Name = "clinic-reservation-db"
    Environment = var.environment
  }
}

# ElastiCache Redis
resource "aws_elasticache_subnet_group" "clinic_redis_subnet_group" {
  name       = "clinic-redis-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}

resource "aws_elasticache_replication_group" "clinic_redis" {
  replication_group_id       = "clinic-redis"
  description                = "Redis cluster for clinic reservation system"
  
  node_type                  = "cache.t3.micro"
  port                       = 6379
  parameter_group_name       = "default.redis7"
  
  num_cache_clusters         = 2
  automatic_failover_enabled = true
  multi_az_enabled          = true
  
  subnet_group_name = aws_elasticache_subnet_group.clinic_redis_subnet_group.name
  security_group_ids = [aws_security_group.redis_sg.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token
  
  tags = {
    Name = "clinic-reservation-redis"
    Environment = var.environment
  }
}
```

---

**策定日**: 2025年8月2日
**技術レビュー**: 四半期毎
**責任者**: 技術責任者・システムアーキテクト