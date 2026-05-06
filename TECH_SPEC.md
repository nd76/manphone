# ManPhone Technical Specification

## Database Schema (Supabase/PostgreSQL)

### users
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone TEXT UNIQUE NOT NULL,  -- E.164 format
  display_name TEXT NOT NULL,
  avatar_url TEXT,
  snooze_used BOOLEAN DEFAULT false,  -- reset nightly
  push_token TEXT,  -- Expo push token
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

### vip_contacts
```sql
CREATE TABLE vip_contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  contact_phone TEXT NOT NULL,  -- E.164
  nickname TEXT NOT NULL,
  alert_level TEXT NOT NULL CHECK (alert_level IN ('red', 'urgent', 'standard')),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(owner_id, contact_phone)
);
```

### quick_replies
```sql
CREATE TABLE quick_replies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  text TEXT NOT NULL,  -- max 120 chars
  sort_order INTEGER DEFAULT 0,
  is_default BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Seed 6 defaults on user creation
CREATE OR REPLACE FUNCTION seed_quick_replies()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO quick_replies (owner_id, text, is_default, sort_order) VALUES
  (NEW.id, 'On my way! 🏃', true, 0),
  (NEW.id, 'Give me 10 minutes', true, 1),
  (NEW.id, 'Yes, I''ll be there ✅', true, 2),
  (NEW.id, 'Can''t make it, sorry', true, 3),
  (NEW.id, 'Call you in 5 min 📞', true, 4),
  (NEW.id, 'Just saw this — on it!', true, 5);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER seed_replies_on_user_create
AFTER INSERT ON users
FOR EACH ROW
EXECUTE FUNCTION seed_quick_replies();
```

### alert_events
```sql
CREATE TABLE alert_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  vip_contact_id UUID NOT NULL REFERENCES vip_contacts(id) ON DELETE CASCADE,
  trigger_type TEXT NOT NULL CHECK (trigger_type IN ('call', 'sms', 'app')),
  status TEXT NOT NULL CHECK (status IN ('pending', 'snoozed', 'replied', 'called', 'dismissed')),
  attempt_count INTEGER DEFAULT 1,
  message TEXT,
  snoozed_until TIMESTAMPTZ,
  resolved_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

### reply_log
```sql
CREATE TABLE reply_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  alert_event_id UUID NOT NULL REFERENCES alert_events(id) ON DELETE CASCADE,
  owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  reply_text TEXT NOT NULL,
  delivery_method TEXT NOT NULL CHECK (delivery_method IN ('sms', 'push', 'app')),
  delivered_at TIMESTAMPTZ DEFAULT now()
);

-- Auto-delete replies after 90 days (cron job on backend)
```

### RLS Policies (all tables)
```sql
-- users table
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users_select_own" ON users FOR SELECT
  USING (auth.uid() = id);
CREATE POLICY "users_update_own" ON users FOR UPDATE
  USING (auth.uid() = id);

-- vip_contacts table
ALTER TABLE vip_contacts ENABLE ROW LEVEL SECURITY;
CREATE POLICY "vip_select_own" ON vip_contacts FOR SELECT
  USING (auth.uid() = owner_id);
CREATE POLICY "vip_insert_own" ON vip_contacts FOR INSERT
  WITH CHECK (auth.uid() = owner_id);
CREATE POLICY "vip_update_own" ON vip_contacts FOR UPDATE
  USING (auth.uid() = owner_id);
CREATE POLICY "vip_delete_own" ON vip_contacts FOR DELETE
  USING (auth.uid() = owner_id);

-- similar for quick_replies, alert_events, reply_log
```

---

## API Endpoints (Node.js on Railway)

Base: `/v1`  
Auth: `Authorization: Bearer <supabase_jwt>`

### Auth

| Method | Route | Body | Returns |
|---|---|---|---|
| POST | /auth/send-otp | `{phone}` | `{success}` |
| POST | /auth/verify-otp | `{phone, code}` | `{session, user}` |
| POST | /auth/refresh | — | `{session}` |
| DELETE | /auth/session | — | `{success}` |

### VIP Contacts

| Method | Route | Body | Returns |
|---|---|---|---|
| GET | /vips | — | `VIP[]` |
| POST | /vips | `{contact_phone, nickname, alert_level}` | `VIP` |
| PATCH | /vips/:id | `{nickname?, alert_level?}` | `VIP` |
| DELETE | /vips/:id | — | `{success}` |

**Rules:**
- Max 5 VIPs per user (enforced in app)
- Free: max 3 VIPs (gated by RevenueCat)
- Red alert level: Plus/Gifted only

### Quick Replies

| Method | Route | Body | Returns |
|---|---|---|---|
| GET | /replies | — | `QuickReply[]` |
| PATCH | /replies/:id | `{text}` | `QuickReply` |
| POST | /replies/:id/send | `{alert_event_id}` | `{delivered}` |

### Alerts (Server → App)

| Method | Route | Body | Returns |
|---|---|---|---|
| GET | /alerts | — | `AlertEvent[]` |
| POST | /alerts/inbound | Twilio webhook | `{processed}` |
| PATCH | /alerts/:id/snooze | — | `AlertEvent` |
| PATCH | /alerts/:id/dismiss | — | `AlertEvent` |

### User

| Method | Route | Body | Returns |
|---|---|---|---|
| GET | /user | — | `User` |
| PATCH | /user | `{display_name?, avatar_url?}` | `User` |
| POST | /user/push-token | `{token}` | `{success}` |
| DELETE | /user | — | `{success}` |

---

## Alert Escalation Engine

**Trigger:** Inbound SMS/call from VIP phone number

**Flow:**
1. Twilio webhook fires POST /alerts/inbound
2. Server looks up sender phone → finds vip_contact
3. If no VIP match → ignore
4. If VIP found → create alert_event (status=pending, attempt_count=1)
5. Send push notification (Expo API)
6. Start escalation timer (60s check interval in production, 5s in dev)
7. If attempt_count=1 after 5 min → increment attempt_count=2, escalate push
8. Red alert: push cannot be swiped (unkillable)
9. Urgent/standard: elevated push only
10. User can snooze once (10 min)
11. After snooze expires → resend escalated push
12. Alert expires after 2 hours with no action → status=dismissed

**Alert Levels:**
- 🔴 Red: full-screen takeover, cannot dismiss without action
- 🟠 Urgent: elevated push + vibration
- 🟡 Standard: regular push, escalates to elevated on attempt 2

---

## Push Notification Spec

| Type | Title | Body | iOS Priority | Android Channel |
|---|---|---|---|---|
| Standard | "{Name} is trying to reach you" | "Tap to reply" | normal | manphone_standard |
| Urgent | "⚠ {Name} needs you" | "They tried twice" | high | manphone_urgent |
| Red Alert | "🔴 {Name} — RED ALERT" | "Open now" | critical | manphone_critical |
| Reply Sent | "Reply delivered ✓" | "Sent to {Name}" | low | manphone_info |

---

## Environment Variables

### App (.env)
```
EXPO_PUBLIC_SUPABASE_URL=
EXPO_PUBLIC_SUPABASE_ANON_KEY=
EXPO_PUBLIC_API_BASE_URL=
```

### Backend (Railway)
```
SUPABASE_URL=
SUPABASE_SERVICE_KEY=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=
TWILIO_VERIFY_SERVICE_ID=
SENTRY_DSN=
```

---

## Monetisation

| Plan | Price | VIPs | Red Alert | Snooze |
|---|---|---|---|---|
| Free | $0 | 3 | No | 1x/alert |
| Plus | $3.99/mo | 5 | Yes | 2x/alert |
| Gifted | $4.99 one-time | 5 | Yes | 2x/alert |

**Gifted:** Partner buys as gift code. Gift code redeems Plus for 1 year.

---

## Build Timeline (12 weeks)

- **Weeks 1-2:** Setup + Session 1 (auth)
- **Weeks 3-4:** Session 2 (VIP CRUD)
- **Weeks 5-6:** Session 3 (push + alerts)
- **Weeks 7-8:** Session 4 (quick replies)
- **Weeks 9-10:** Session 5 (RevenueCat)
- **Weeks 11-12:** Session 6 (polish + submission)

---

## Critical Notes

- **iOS builds:** EAS Build (cloud) only. No local Xcode required on Windows.
- **Android:** Can test locally via Expo Go or Android emulator.
- **Push notifications:** iOS critical alerts need Apple entitlement (apply in session 6).
- **Snooze reset:** Daily cron job at midnight (local time) resets snooze_used→false.
- **Alert expiry:** 2-hour cron job purges dismissed alerts older than 30 days.
- **Reply log retention:** 90-day cron job deletes old reply_log entries.
