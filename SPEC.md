# Muhabbet — Aile İletişim Uygulaması Teknik Spesifikasyonu

**Versiyon:** 1.0 · **Tarih:** 2026-09-29 · **Durum:** Taslak

## 1. Genel Bakış

Muhabbet, aile üyeleri arasında güvenli iletişim sağlayan bir mobil uygulamadır.
Tüm mesajlaşma uçtan uca şifrelidir (E2E); sunucu mesaj içeriğini hiçbir zaman
düz metin olarak göremez.

### Özellikler
- **Sohbet:** 1-1 ve grup metin mesajlaşması, okundu bilgisi, yazıyor göstergesi
- **Video konferans:** 1-1 ve grup görüşmesi (8 kişiye kadar), WebRTC tabanlı
- **Medya paylaşımı:** Dosya, fotoğraf, video (şifreli olarak S3'te saklanır)
- **Bildirimler:** Push notification (FCM/APNs), uygulama içi bildirim merkezi

### Teknik Gereklilikler
| Gereklilik | Çözüm |
|---|---|
| iOS + Android | React Native (tek kod tabanı, native modül desteği) |
| E2E şifreleme | Signal Protocol (libsignal) — X3DH anahtar anlaşması + Double Ratchet |
| Offline mesajlaşma | Cihazda SQLite kuyruk + sunucuda mesaj saklama, bağlantı gelince senkronizasyon |
| Push notification | FCM (Android) + APNs (iOS), içeriksiz "uyandırma" push'u (E2E korunur) |

### Mimari Tercihler
- **Backend:** Node.js (NestJS) — WebSocket ve REST aynı süreçte, TypeScript ile tip güvenliği
- **Veritabanı:** PostgreSQL (kalıcı veri) + Redis (presence, oturum, pub/sub)
- **Gerçek zamanlı:** Socket.IO (mesajlaşma sinyalleşmesi), LiveKit (WebRTC SFU — video konferans)
- **Entegrasyonlar:** AWS S3 (şifreli medya depolama, presigned URL), Firebase Cloud Messaging (push), Sentry (hata izleme), PostHog (self-hosted analytics — mesaj içeriği asla loglanmaz)

---

## 2. Teknik Mimari Diagram (ASCII)

```
┌─────────────────────────┐        ┌─────────────────────────┐
│      iOS Uygulaması     │        │    Android Uygulaması   │
│  (React Native + Swift  │        │ (React Native + Kotlin  │
│   native modüller)      │        │   native modüller)      │
│  ┌───────────────────┐  │        │  ┌───────────────────┐  │
│  │ libsignal (E2E)   │  │        │  │ libsignal (E2E)   │  │
│  │ SQLite (offline)  │  │        │  │ SQLite (offline)  │  │
│  └───────────────────┘  │        │  └───────────────────┘  │
└────────┬───────▲────────┘        └────────┬───────▲────────┘
         │       │ APNs                     │       │ FCM
         │       │                          │       │
         │  ┌────┴──────────────────────────┼───────┴─────┐
         │  │        Push Servisi (FCM/APNs köprüsü)      │
         │  └────▲────────────────────────────────────────┘
         │       │ içeriksiz uyandırma push'u
─────────▼───────┴──────────────────────────▼───────────────────
                    HTTPS (REST) + WSS (Socket.IO)
                              │
                    ┌─────────▼──────────┐
                    │   API Gateway /    │
                    │   Load Balancer    │
                    │   (nginx + TLS)    │
                    └─────────┬──────────┘
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
     ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐
     │  NestJS API    │ │ NestJS WS    │ │  LiveKit SFU     │
     │  (REST: auth,  │ │ (Socket.IO:  │ │  (WebRTC video   │
     │  kullanıcı,    │ │ mesaj teslim,│ │  konferans,      │
     │  medya, anahtar│ │ presence,    │ │  TURN/STUN dahil)│
     │  dağıtımı)     │ │ typing)      │ │                  │
     └───────┬────────┘ └──────┬───────┘ └────────┬─────────┘
             │                 │                  │
      ┌──────┴─────────┬───────┴──────┐           │ oda token'ı
      ▼                ▼              ▼           │ REST API'den
┌───────────┐   ┌────────────┐  ┌──────────┐      │ alınır
│PostgreSQL │   │   Redis    │  │  AWS S3  │◄─────┘
│(kullanıcı,│   │ (presence, │  │ (şifreli │
│ sohbet    │   │  pub/sub,  │  │  medya,  │
│ metadata, │   │  oturum,   │  │ presigned│
│ şifreli   │   │  rate      │  │  URL)    │
│ mesaj     │   │  limit)    │  └──────────┘
│ kuyruğu)  │   └────────────┘
└───────────┘
```

**Veri akışı (mesaj gönderimi):**
1. İstemci mesajı alıcının her cihazı için ayrı ayrı şifreler (Double Ratchet).
2. Mesaj WSS üzerinden gönderilir; istemci offline ise SQLite kuyruğuna yazılır, bağlantı gelince gönderilir.
3. Sunucu şifreli payload'ı PostgreSQL'e yazar (alıcı teslim alana kadar saklanır), online alıcılara Socket.IO ile iletir.
4. Offline alıcılara içeriksiz push gönderilir ("Yeni mesajınız var"); uygulama açılınca mesajlar çekilip cihazda çözülür.
5. Tüm alıcı cihazlar teslim onayı verince sunucudaki şifreli kopya silinir (TTL: maks. 30 gün).

---

## 3. Database Schema

### PostgreSQL

```sql
-- Kullanıcılar (telefon numarası ile kayıt)
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number    VARCHAR(20) UNIQUE NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    avatar_url      TEXT,                          -- S3 key, şifreli değil (profil herkese açık)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ
);

-- Cihazlar (E2E: her cihazın kendi anahtar seti var)
CREATE TABLE devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    platform        VARCHAR(10) NOT NULL CHECK (platform IN ('ios', 'android')),
    push_token      TEXT,                          -- FCM/APNs token
    identity_key    BYTEA NOT NULL,                -- Signal identity public key
    signed_prekey   BYTEA NOT NULL,
    signed_prekey_sig BYTEA NOT NULL,
    registered_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, identity_key)
);

-- Tek kullanımlık ön-anahtarlar (X3DH için, istemci yükler, sunucu dağıtır)
CREATE TABLE one_time_prekeys (
    id              BIGSERIAL PRIMARY KEY,
    device_id       UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    key_id          INT NOT NULL,
    public_key      BYTEA NOT NULL,
    used            BOOLEAN NOT NULL DEFAULT false,
    UNIQUE (device_id, key_id)
);

-- Sohbetler (1-1 ve grup)
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type            VARCHAR(10) NOT NULL CHECK (type IN ('direct', 'group')),
    name            VARCHAR(100),                  -- sadece grup için
    avatar_url      TEXT,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conversation_members (
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(10) NOT NULL DEFAULT 'member' CHECK (role IN ('admin', 'member')),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    muted_until     TIMESTAMPTZ,                   -- bildirim susturma
    PRIMARY KEY (conversation_id, user_id)
);

-- Mesaj kuyruğu: sunucu SADECE şifreli blob saklar, teslimden sonra silinir
CREATE TABLE message_queue (
    id              BIGSERIAL PRIMARY KEY,
    message_id      UUID NOT NULL,                 -- istemci üretir (idempotency)
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    sender_device_id UUID NOT NULL REFERENCES devices(id),
    recipient_device_id UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    ciphertext      BYTEA NOT NULL,                -- Double Ratchet çıktısı
    message_type    VARCHAR(20) NOT NULL DEFAULT 'text'
                    CHECK (message_type IN ('text', 'media', 'receipt', 'key_update')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '30 days',
    UNIQUE (message_id, recipient_device_id)
);
CREATE INDEX idx_queue_recipient ON message_queue (recipient_device_id, created_at);

-- Medya metadata (içerik S3'te, şifreleme anahtarı mesajın içinde E2E taşınır)
CREATE TABLE media_objects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    s3_key          TEXT NOT NULL UNIQUE,
    uploader_id     UUID NOT NULL REFERENCES users(id),
    size_bytes      BIGINT NOT NULL,
    content_type    VARCHAR(100) NOT NULL,         -- application/octet-stream (şifreli)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '90 days'
);

-- Görüşme kayıtları (sadece metadata — kim, ne zaman, süre)
CREATE TABLE call_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    initiated_by    UUID NOT NULL REFERENCES users(id),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ,
    type            VARCHAR(10) NOT NULL CHECK (type IN ('audio', 'video'))
);
```

### Redis (geçici veri)
| Key | Tip | Amaç |
|---|---|---|
| `presence:{userId}` | STRING + TTL 60s | Online durumu (heartbeat ile yenilenir) |
| `typing:{convId}` | SET + TTL 5s | Yazıyor göstergesi |
| `socket:{deviceId}` | STRING | Hangi WS sunucusuna bağlı (yatay ölçekleme) |
| `otp:{phone}` | STRING + TTL 5dk | SMS doğrulama kodu |
| `ratelimit:{ip}` | Sliding window | Rate limiting |

### İstemci SQLite (offline)
```sql
-- Çözülmüş mesajlar SADECE cihazda saklanır
CREATE TABLE local_messages (
    message_id      TEXT PRIMARY KEY,
    conversation_id TEXT NOT NULL,
    sender_id       TEXT NOT NULL,
    body            TEXT,              -- düz metin (cihaz şifrelemesi: SQLCipher)
    media_local_path TEXT,
    status          TEXT NOT NULL,     -- pending | sent | delivered | read
    created_at      INTEGER NOT NULL
);
CREATE TABLE outbox (                  -- offline gönderim kuyruğu
    message_id      TEXT PRIMARY KEY,
    payload         BLOB NOT NULL,     -- şifrelenmiş, gönderime hazır
    retry_count     INTEGER DEFAULT 0,
    created_at      INTEGER NOT NULL
);
CREATE TABLE signal_sessions (         -- Double Ratchet oturum durumu
    address         TEXT PRIMARY KEY,  -- userId.deviceId
    session_state   BLOB NOT NULL
);
```

---

## 4. API Endpoints

### REST (HTTPS, `Authorization: Bearer <JWT>`)

```
# Kimlik doğrulama
POST   /v1/auth/otp/request          { phone }                 → 204 (SMS gönderildi)
POST   /v1/auth/otp/verify           { phone, code }           → { accessToken, refreshToken, userId }
POST   /v1/auth/refresh              { refreshToken }          → { accessToken }

# Cihaz & E2E anahtar yönetimi
POST   /v1/devices                   { platform, pushToken, identityKey,
                                       signedPrekey, signedPrekeySig,
                                       oneTimePrekeys[] }      → { deviceId }
PUT    /v1/devices/:id/push-token    { pushToken }             → 204
POST   /v1/devices/:id/prekeys       { oneTimePrekeys[] }      → 204   # stok azalınca yenile
GET    /v1/users/:userId/key-bundles                           → [{ deviceId, identityKey,
                                                                    signedPrekey, oneTimePrekey }]

# Kullanıcı & sohbet
GET    /v1/users/me                                            → profil
PATCH  /v1/users/me                  { displayName, avatarUrl }→ profil
POST   /v1/contacts/sync             { phoneHashes[] }         → eşleşen kullanıcılar
POST   /v1/conversations             { type, memberIds[], name }→ { conversationId }
GET    /v1/conversations                                       → sohbet listesi (metadata)
PATCH  /v1/conversations/:id         { name, avatarUrl }       → 204
POST   /v1/conversations/:id/members { userId }                → 204
DELETE /v1/conversations/:id/members/:userId                   → 204
PUT    /v1/conversations/:id/mute    { until }                 → 204

# Offline senkronizasyon
GET    /v1/messages/pending?deviceId=...                       → [{ messageId, convId,
                                                                    senderDeviceId, ciphertext }]
POST   /v1/messages/ack              { messageIds[] }          → 204   # sunucudan silinir

# Medya (E2E: istemci dosyayı AES-256-GCM ile şifreler, anahtar mesajla gider)
POST   /v1/media/upload-url          { sizeBytes, contentType }→ { mediaId, presignedPutUrl }
GET    /v1/media/:id/download-url                              → { presignedGetUrl }

# Video konferans
POST   /v1/calls                     { conversationId, type }  → { callId, livekitToken, livekitUrl }
POST   /v1/calls/:id/join                                      → { livekitToken }
POST   /v1/calls/:id/end                                       → 204
```

### WebSocket (Socket.IO, `wss://`, JWT ile el sıkışma)

```
# İstemci → Sunucu
message:send      { messageId, conversationId,
                    payloads: [{ recipientDeviceId, ciphertext }] }   # cihaz başına şifreli
message:ack       { messageIds[] }                                    # teslim onayı
receipt:read      { conversationId, messageId }                       # okundu (E2E payload olarak)
typing:start      { conversationId }
typing:stop       { conversationId }
presence:ping     {}                                                  # 30 sn'de bir heartbeat

# Sunucu → İstemci
message:new       { messageId, conversationId, senderDeviceId, ciphertext, createdAt }
message:delivered { messageId, recipientUserId }
typing:update     { conversationId, userIds[] }
presence:update   { userId, status, lastSeenAt }
call:incoming     { callId, conversationId, callerId, type }
conversation:updated { conversationId }                               # üye/isim değişikliği
```

---

## 5. Platform-Spesifik Başlangıç Kodu

Proje yapısı (React Native, TypeScript):
```
muhabbet/
├── apps/mobile/          # React Native uygulaması
│   ├── ios/              # Xcode projesi + Podfile
│   ├── android/          # Gradle projesi
│   └── src/
├── apps/server/          # NestJS backend
└── packages/shared/      # ortak tipler (API kontratları)
```

### Ortak: WebSocket + Offline kuyruk (`apps/mobile/src/services/MessageService.ts`)

```typescript
import { io, Socket } from 'socket.io-client';
import NetInfo from '@react-native-community/netinfo';
import { db } from './LocalDb';            // SQLCipher ile açılan SQLite
import { encryptForRecipients } from './SignalService';

export class MessageService {
  private socket: Socket;

  constructor(private apiUrl: string, private getToken: () => string) {
    this.socket = io(apiUrl, {
      transports: ['websocket'],
      auth: (cb) => cb({ token: this.getToken() }),
    });
    this.socket.on('connect', () => this.flushOutbox());
    this.socket.on('message:new', (msg) => this.onIncoming(msg));
    NetInfo.addEventListener((state) => {
      if (state.isConnected && this.socket.disconnected) this.socket.connect();
    });
  }

  async send(conversationId: string, plaintext: string) {
    const messageId = crypto.randomUUID();
    const payloads = await encryptForRecipients(conversationId, plaintext);
    // Önce yerel DB'ye yaz — UI anında güncellenir, offline'da kaybolmaz
    await db.insertLocalMessage({ messageId, conversationId, body: plaintext, status: 'pending' });
    await db.insertOutbox({ messageId, payload: JSON.stringify({ conversationId, payloads }) });
    this.flushOutbox();
  }

  private async flushOutbox() {
    if (!this.socket.connected) return;
    for (const item of await db.getOutbox()) {
      this.socket.emit('message:send', { messageId: item.messageId, ...JSON.parse(item.payload) },
        async (ack: { ok: boolean }) => {
          if (ack.ok) {
            await db.deleteOutbox(item.messageId);
            await db.updateStatus(item.messageId, 'sent');
          }
        });
    }
  }

  private async onIncoming(msg: { messageId: string; senderDeviceId: string; ciphertext: string }) {
    // SignalService cihazdaki oturumla çözer, local_messages'a yazar
  }
}
```

### iOS — İçeriksiz push ile uyandırma (`ios/Muhabbet/NotificationService.swift`)

APNs push'u içerik taşımaz; Notification Service Extension mesajı çekip cihazda çözer:

```swift
import UserNotifications

class NotificationService: UNNotificationServiceExtension {
  var contentHandler: ((UNNotificationContent) -> Void)?
  var bestAttempt: UNMutableNotificationContent?

  override func didReceive(_ request: UNNotificationRequest,
                           withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void) {
    self.contentHandler = contentHandler
    bestAttempt = (request.content.mutableCopy() as? UNMutableNotificationContent)
    guard let bestAttempt = bestAttempt else { return }

    // Push payload'ında sadece messageId var; içerik sunucudan çekilip cihazda çözülür
    MessageFetcher.shared.fetchAndDecrypt { result in
      switch result {
      case .success(let message):
        bestAttempt.title = message.senderName
        bestAttempt.body = message.preview      // çözülmüş metin — sadece cihazda
      case .failure:
        bestAttempt.title = "Muhabbet"
        bestAttempt.body = "Yeni mesajınız var"
      }
      contentHandler(bestAttempt)
    }
  }

  override func serviceExtensionTimeWillExpire() {
    if let handler = contentHandler, let content = bestAttempt { handler(content) }
  }
}
```

`Info.plist` izinleri: `NSCameraUsageDescription`, `NSMicrophoneUsageDescription`,
`NSPhotoLibraryUsageDescription`. Capabilities: Push Notifications, Background Modes
(remote-notification, voip), App Groups (extension ↔ app SQLite paylaşımı), Keychain Sharing.

### Android — FCM data mesajı (`android/app/src/main/java/com/muhabbet/MessagingService.kt`)

```kotlin
package com.muhabbet

import com.google.firebase.messaging.FirebaseMessagingService
import com.google.firebase.messaging.RemoteMessage

class MessagingService : FirebaseMessagingService() {

    // Data-only mesaj: bildirim içeriği sunucudan gelmez, cihazda üretilir
    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        val messageId = remoteMessage.data["messageId"] ?: return
        // WorkManager ile arka planda: bekleyen mesajları çek → libsignal ile çöz →
        // local DB'ye yaz → NotificationCompat ile çözülmüş önizlemeyi göster
        MessageSyncWorker.enqueue(applicationContext, messageId)
    }

    override fun onNewToken(token: String) {
        // Token'ı API'ye kaydet: PUT /v1/devices/:id/push-token
        PushTokenRepository.uploadToken(applicationContext, token)
    }
}
```

`AndroidManifest.xml`: `INTERNET`, `CAMERA`, `RECORD_AUDIO`, `POST_NOTIFICATIONS`,
`FOREGROUND_SERVICE` (devam eden görüşme için), service tanımı `MessagingService`
(`com.google.firebase.MESSAGING_EVENT` intent filter).

---

## 6. Gerekli Paketler

### npm — Mobil (`apps/mobile/package.json`)
| Paket | Amaç |
|---|---|
| `react-native` (0.7x) | Çatı |
| `@signalapp/libsignal-client` | E2E (Signal Protocol) |
| `socket.io-client` | Gerçek zamanlı mesajlaşma |
| `@livekit/react-native` + `livekit-client` | Video konferans (WebRTC) |
| `react-native-quick-sqlite` + SQLCipher | Şifreli yerel DB / offline kuyruk |
| `@react-native-community/netinfo` | Bağlantı durumu izleme |
| `@react-native-firebase/app`, `@react-native-firebase/messaging` | FCM push |
| `@notifee/react-native` | Yerel bildirim gösterimi |
| `react-native-keychain` | Anahtar saklama (Keychain/Keystore) |
| `react-native-image-picker`, `react-native-document-picker` | Medya seçimi |
| `react-native-compressor` | Görüntü/video sıkıştırma (yükleme öncesi) |
| `react-native-blob-util` | S3 presigned URL'e şifreli upload/download |
| `@reduxjs/toolkit` veya `zustand` | State yönetimi |
| `react-navigation` (native-stack) | Navigasyon |

### npm — Backend (`apps/server/package.json`)
| Paket | Amaç |
|---|---|
| `@nestjs/core`, `@nestjs/platform-socket.io`, `@nestjs/websockets` | API + WS |
| `prisma`, `@prisma/client` | PostgreSQL ORM + migration |
| `ioredis` | Redis istemcisi |
| `@socket.io/redis-adapter` | WS yatay ölçekleme |
| `livekit-server-sdk` | Görüşme odası token üretimi |
| `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner` | Medya depolama |
| `firebase-admin` | FCM/APNs push gönderimi |
| `@nestjs/jwt`, `argon2` | Kimlik doğrulama |
| `@nestjs/throttler` | Rate limiting |
| `zod` | Girdi doğrulama |

### CocoaPods (`apps/mobile/ios/Podfile` — autolinking dışı ekstralar)
```ruby
pod 'Firebase/Messaging'
pod 'SQLCipher', '~> 4.5'
# LiveKit, libsignal ve RN modülleri autolinking ile gelir
# Notification Service Extension target'ına da SQLCipher + libsignal eklenmeli
```

---

## 7. Deployment Plan

### Ortamlar
| Ortam | Amaç | Altyapı |
|---|---|---|
| `dev` | Geliştirme | Docker Compose (lokal: Postgres, Redis, LiveKit, MinIO) |
| `staging` | Test + TestFlight/Internal Track | Tek node, küçük instance'lar |
| `prod` | Canlı | Aşağıdaki topoloji |

### Prod Altyapısı (AWS örneği)
- **API + WS:** ECS Fargate (min 2 task, ALB arkasında; WS için sticky session + Redis adapter)
- **LiveKit SFU:** EC2 (UDP portları gerekli — Fargate uygun değil), ko-lokasyonda TURN sunucusu
- **PostgreSQL:** RDS (Multi-AZ, otomatik yedekleme, 7 gün PITR)
- **Redis:** ElastiCache (replikalı)
- **S3:** Medya bucket'ı — SSE-S3 (içerik zaten istemcide şifreli), 90 gün lifecycle silme kuralı
- **TLS:** ACM sertifikası, ALB'de sonlandırma; HSTS zorunlu

### CI/CD (GitHub Actions)
1. **PR:** lint + tip kontrolü + birim testler (backend: Jest, mobil: Jest + RN Testing Library)
2. **`main` merge:** backend Docker imajı → ECR → staging'e otomatik deploy; Prisma migration `migrate deploy` ile
3. **Mobil release branch:** Fastlane ile
   - iOS: build + TestFlight'a yükleme (`fastlane ios beta`)
   - Android: AAB build + Play Internal Track (`fastlane android beta`)
4. **Prod deploy:** staging smoke testleri sonrası manuel onayla; mağaza yayınları manuel promote

### Yayın Sırası (ilk sürüm)
1. Backend + DB + LiveKit staging'de ayağa kalkar; OTP için SMS sağlayıcısı (ör. Twilio/Netgsm) bağlanır
2. Mobil beta (TestFlight + Play Internal) — aile içi kapalı test, 2 hafta
3. Anahtar doğrulama UX'i (güvenlik numarası karşılaştırma) ve cihaz kaybı senaryoları test edilir
4. Mağaza incelemeleri: App Store şifreleme beyanı (`ITSAppUsesNonExemptEncryption` + yıllık self-classification raporu), Play Data Safety formu
5. Kademeli yayın: Play'de %10 → %50 → %100; iOS phased release

### İzleme & Operasyon
- Sentry (crash + hata), CloudWatch (altyapı metrikleri), PostHog (ürün metrikleri — yalnızca olay adları, asla içerik)
- Uyarılar: WS bağlantı hata oranı, mesaj kuyruğu gecikmesi (p95 < 500ms hedefi), push teslim oranı
- Veri saklama: teslim edilen mesajlar anında, teslim edilmeyenler 30 günde silinir; medya 90 gün

### Güvenlik Kontrol Listesi
- [ ] E2E anahtarları yalnızca Keychain/Keystore'da; yedeklemeye dahil edilmez
- [ ] Sunucu loglarında telefon numarası maskeleme, içerik asla loglanmaz
- [ ] Sertifika pinning (mobil istemcide)
- [ ] OTP rate limiting (numara başına 3/saat) + cihaz kayıt bildirimi (yeni cihaz girişinde diğer cihazlara uyarı)
- [ ] Bağımsız güvenlik denetimi (yayın öncesi pentest)
