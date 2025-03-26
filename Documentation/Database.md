# Psychologist Booking System Database Tables

## Database Tables Overview

| Table Name | Description |
|------------|-------------|
| `users` | Stores client information who book appointments |
| `psychologists` | Stores information about psychologists providing services |
| `availability` | Defines recurring time slots when psychologists are available |
| `appointments` | Records all booked appointments and their status |
| `metrics` | Tracks user interaction data for A/B testing |
| `chat_sessions` | Monitors WhatsApp conversation lifecycles |
| `session_states` | Manages conversation flow state for the WhatsApp bot |

This is the Entity Relationship Diagram for the Database\
![Entity Relationship Diagram](/Documentation/DB-ERD.png)

## Table Definitions

### users
Stores information about clients who use the booking system.
Phone number is UNIQUE as it's their primary identifier through WhatsApp

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,  -- WhatsApp number
    email VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);
```

### psychologists
Contains data for the psychologist(s) offering appointments.

```sql
CREATE TABLE psychologists (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    bio TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);
```

### availability
Defines when psychologists are available for appointments.
```sql
CREATE TABLE availability (
    id SERIAL PRIMARY KEY,
    psychologist_id INTEGER NOT NULL REFERENCES psychologists(id) ON DELETE CASCADE,
    day_of_week INTEGER NOT NULL, -- 0-6 (Sunday-Saturday)
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    CONSTRAINT valid_time_range CHECK (start_time < end_time)
);
```

### appointments
Core table storing all booked appointments.
```sql
CREATE TABLE appointments (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    psychologist_id INTEGER NOT NULL REFERENCES psychologists(id) ON DELETE CASCADE,
    appointment_date DATE NOT NULL,
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled', -- scheduled, confirmed, cancelled, completed
    notes TEXT,
    entry_method VARCHAR(20) NOT NULL, -- 'click_to_chat' or 'web_form'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    CONSTRAINT valid_appointment_time CHECK (start_time < end_time)
);
```

### metrics
Captures data for A/B testing of different booking approaches.
```sql
CREATE TABLE metrics (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE SET NULL,
    entry_method VARCHAR(20) NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
    appointment_id INTEGER REFERENCES appointments(id) ON DELETE SET NULL,
    session_id VARCHAR(100) NOT NULL,
    time_taken INTEGER,
    additional_data JSONB
);
```

### chat_sessions
Tracks WhatsApp conversations with users.
```sql
CREATE TABLE chat_sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE SET NULL,
    whatsapp_id VARCHAR(100) NOT NULL, -- WhatsApp chat identifier
    session_start TIMESTAMP WITH TIME ZONE DEFAULT now(),
    session_end TIMESTAMP WITH TIME ZONE,
    status VARCHAR(20) NOT NULL DEFAULT 'active', -- active, completed, abandoned
    entry_method VARCHAR(20) NOT NULL, -- 'click_to_chat' or 'web_form'
    appointment_id INTEGER REFERENCES appointments(id) ON DELETE SET NULL
);
```

### session_states
Manages the state of bot conversations in WhatsApp.
```sql
CREATE TABLE session_states (
    id SERIAL PRIMARY KEY,
    chat_session_id INTEGER NOT NULL REFERENCES chat_sessions(id) ON DELETE CASCADE,
    current_step VARCHAR(50) NOT NULL, -- 'welcome', 'date_selection', 'time_selection', etc.
    context JSONB NOT NULL DEFAULT '{}', -- Stores conversation context
    last_updated TIMESTAMP WITH TIME ZONE DEFAULT now()
);
```

#### Functions for Automatic Timestamp Updates

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

#### Triggers for automatic timestamp updates

```sql
CREATE TRIGGER update_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_psychologists_updated_at
BEFORE UPDATE ON psychologists
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_availability_updated_at
BEFORE UPDATE ON availability
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_appointments_updated_at
BEFORE UPDATE ON appointments
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

#### Indexes to optimize queries

```sql
CREATE INDEX idx_appointments_date ON appointments(appointment_date);
CREATE INDEX idx_appointments_user ON appointments(user_id);
CREATE INDEX idx_appointments_status ON appointments(status);
CREATE INDEX idx_metrics_entry_method ON metrics(entry_method);
CREATE INDEX idx_metrics_event_type ON metrics(event_type);
CREATE INDEX idx_chat_sessions_whatsapp ON chat_sessions(whatsapp_id);
```

## Row Level Security (RLS) Implementation

Note: These RLS policies should be implemented after the core functionality is working. During development, you can work without these restrictions to simplify testing and debugging.

### Service Role Implementation Note

This system uses a WhatsApp-based architecture where users primarily interact through messaging rather than direct web authentication. The backend services (Node-RED for WhatsApp integration and Go backend for business logic) will access Supabase using a service role.

To implement this:
1. Create a service role API key in Supabase dashboard (Settings > API)
2. Use this key in your Node-RED flows and Go backend services
3. Store this key securely as an environment variable, never in code
4. The service role bypasses RLS, so your backend can perform all necessary operations

### RLS Policies

Apply these policies after your core implementation is complete and working:

#### Users Table RLS

```sql
-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Service role policy (for WhatsApp integration via Node-RED and Go backend)
CREATE POLICY "Service role can manage all user data" 
ON users 
USING (auth.role() = 'service_role');

-- Admin policy (for web dashboard access)
CREATE POLICY "Admins can manage all user data" 
ON users 
USING (auth.jwt() ->> 'role' = 'admin');
```

#### Psychologists Table RLS

```sql
-- Enable RLS
ALTER TABLE psychologists ENABLE ROW LEVEL SECURITY;

-- Service role policy
CREATE POLICY "Service role can manage psychologist data" 
ON psychologists 
USING (auth.role() = 'service_role');

-- Admin policy
CREATE POLICY "Admins can manage all psychologist data" 
ON psychologists 
USING (auth.jwt() ->> 'role' = 'admin');

-- Policy for psychologists to view their own data (for web dashboard)
CREATE POLICY "Psychologists can view their own data" 
ON psychologists FOR SELECT 
USING (auth.jwt() ->> 'psychologist_id'::text = id::text);

-- Policy for psychologists to update their own data (for web dashboard)
CREATE POLICY "Psychologists can update their own data" 
ON psychologists FOR UPDATE 
USING (auth.jwt() ->> 'psychologist_id'::text = id::text);
```

#### Availability Table RLS

```sql
-- Enable RLS
ALTER TABLE availability ENABLE ROW LEVEL SECURITY;

-- Service role policy
CREATE POLICY "Service role can manage availability" 
ON availability 
USING (auth.role() = 'service_role');

-- Admin policy
CREATE POLICY "Admins can manage all availability" 
ON availability 
USING (auth.jwt() ->> 'role' = 'admin');

-- Policy for psychologists to manage their own availability (for web dashboard)
CREATE POLICY "Psychologists can manage their own availability" 
ON availability
USING (
  auth.jwt() ->> 'psychologist_id'::text = psychologist_id::text
);

-- Public read policy (allow anyone to view active availability)
CREATE POLICY "Anyone can view active availability" 
ON availability FOR SELECT 
USING (is_active = TRUE);
```

#### Appointments Table RLS

```sql
-- Enable RLS
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;

-- Service role policy
CREATE POLICY "Service role can manage appointments" 
ON appointments 
USING (auth.role() = 'service_role');

-- Admin policy
CREATE POLICY "Admins can manage all appointments" 
ON appointments 
USING (auth.jwt() ->> 'role' = 'admin');

-- Policy for psychologists to view their own appointments (for web dashboard)
CREATE POLICY "Psychologists can view their appointments" 
ON appointments FOR SELECT 
USING (auth.jwt() ->> 'psychologist_id'::text = psychologist_id::text);

-- Policy for psychologists to update their appointments (for web dashboard)
CREATE POLICY "Psychologists can update their appointments" 
ON appointments FOR UPDATE
USING (auth.jwt() ->> 'psychologist_id'::text = psychologist_id::text);
```

#### Metrics Table RLS

```sql
-- Enable RLS
ALTER TABLE metrics ENABLE ROW LEVEL SECURITY;

-- Service role policy
CREATE POLICY "Service role can manage metrics" 
ON metrics 
USING (auth.role() = 'service_role');

-- Admin policy
CREATE POLICY "Admins can view all metrics" 
ON metrics FOR SELECT
USING (auth.jwt() ->> 'role' = 'admin');
```

#### Chat Sessions Table RLS

```sql
-- Enable RLS
ALTER TABLE chat_sessions ENABLE ROW LEVEL SECURITY;

-- Service role policy
CREATE POLICY "Service role can manage chat sessions" 
ON chat_sessions 
USING (auth.role() = 'service_role');

-- Admin policy
CREATE POLICY "Admins can view all chat sessions" 
ON chat_sessions FOR SELECT
USING (auth.jwt() ->> 'role' = 'admin');
```

#### Session States Table RLS

```sql
-- Enable RLS
ALTER TABLE session_states ENABLE ROW LEVEL SECURITY;

-- Service role policy
CREATE POLICY "Service role can manage session states" 
ON session_states 
USING (auth.role() = 'service_role');

-- Admin policy
CREATE POLICY "Admins can view all session states" 
ON session_states FOR SELECT
USING (auth.jwt() ->> 'role' = 'admin');
```
