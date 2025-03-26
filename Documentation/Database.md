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
![Descriptive Text](Documentation/DB-ERD.png)

## Table Definitions

### users
Stores information about clients who use the booking system.
Phone number is UNIQUE as it's their primary identifier through WhatsApp

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    auth_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,  -- WhatsApp number
    email VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Set up RLS (Row Level Security) for users table
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only read/write their own data
CREATE POLICY "Users can see their own data" 
ON users FOR SELECT USING (auth.uid() = auth_id);

CREATE POLICY "Users can update their own data" 
ON users FOR UPDATE USING (auth.uid() = auth_id);

-- NEW POLICIES
CREATE POLICY "Users can insert their own data" 
ON users FOR INSERT WITH CHECK (auth.uid() = auth_id);

CREATE POLICY "Users cannot delete their accounts" 
ON users FOR DELETE USING (false);

-- Admin policy for user management (will need to be applied to all tables)
CREATE POLICY "Admins can manage all user data" 
ON users USING (auth.jwt() ->> 'role' = 'admin');
```

### psychologists
Contains data for the psychologist(s) offering appointments.

```sql
CREATE TABLE psychologists (
    id SERIAL PRIMARY KEY,
    auth_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    bio TEXT,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- RLS policies will limit write access to admins only
ALTER TABLE psychologists ENABLE ROW LEVEL SECURITY;

-- Policy: Everyone can see active psychologists
CREATE POLICY "Anyone can view active psychologists" 
ON psychologists FOR SELECT USING (is_active = TRUE);

-- NEW POLICIES
CREATE POLICY "Psychologists can update their own profile" 
ON psychologists FOR UPDATE 
USING (auth.uid() = auth_id);

CREATE POLICY "Only admins can create psychologist accounts" 
ON psychologists FOR INSERT
WITH CHECK (auth.jwt() ->> 'role' = 'admin');

CREATE POLICY "Only admins can delete psychologist accounts" 
ON psychologists FOR DELETE
USING (auth.jwt() ->> 'role' = 'admin');
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

-- RLS for availability
ALTER TABLE availability ENABLE ROW LEVEL SECURITY;

-- Policy: All users can read availability
CREATE POLICY "Anyone can see availability" 
ON availability FOR SELECT USING (TRUE);

-- NEW POLICIES
CREATE POLICY "Psychologists can manage their own availability" 
ON availability FOR INSERT
WITH CHECK (
    auth.uid() IN (
        SELECT auth_id FROM psychologists WHERE id = availability.psychologist_id
    )
);

CREATE POLICY "Psychologists can update their own availability" 
ON availability FOR UPDATE
USING (
    auth.uid() IN (
        SELECT auth_id FROM psychologists WHERE id = availability.psychologist_id
    )
);

CREATE POLICY "Psychologists can delete their own availability" 
ON availability FOR DELETE
USING (
    auth.uid() IN (
        SELECT auth_id FROM psychologists WHERE id = availability.psychologist_id
    )
);

CREATE POLICY "Admins can manage all availability" 
ON availability 
USING (auth.jwt() ->> 'role' = 'admin');
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

-- RLS for appointments
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own appointments
CREATE POLICY "Users can see their own appointments" 
ON appointments FOR SELECT 
USING (
    auth.uid() IN (
        SELECT auth_id FROM users WHERE id = appointments.user_id
    )
);

-- Policy: Psychologists can see appointments assigned to them
CREATE POLICY "Psychologists can see their appointments" 
ON appointments FOR SELECT 
USING (
    auth.uid() IN (
        SELECT auth_id FROM psychologists WHERE id = appointments.psychologist_id
    )
);

-- NEW POLICIES
CREATE POLICY "Users can create their own appointments" 
ON appointments FOR INSERT
WITH CHECK (
    auth.uid() IN (
        SELECT auth_id FROM users WHERE id = appointments.user_id
    )
);

CREATE POLICY "Users can update their own appointments" 
ON appointments FOR UPDATE
USING (
    auth.uid() IN (
        SELECT auth_id FROM users WHERE id = appointments.user_id
    )
);

CREATE POLICY "Psychologists can update their appointments" 
ON appointments FOR UPDATE
USING (
    auth.uid() IN (
        SELECT auth_id FROM psychologists WHERE id = appointments.psychologist_id
    )
);

CREATE POLICY "Users can cancel their own appointments" 
ON appointments FOR DELETE
USING (
    auth.uid() IN (
        SELECT auth_id FROM users WHERE id = appointments.user_id
    )
);

CREATE POLICY "Admins can manage all appointments" 
ON appointments 
USING (auth.jwt() ->> 'role' = 'admin');
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

-- NEW POLICIES
ALTER TABLE metrics ENABLE ROW LEVEL SECURITY;

CREATE POLICY "System services can insert metrics" 
ON metrics FOR INSERT
WITH CHECK (auth.jwt() ->> 'role' = 'service');

CREATE POLICY "Admins can view all metrics" 
ON metrics FOR SELECT
USING (auth.jwt() ->> 'role' = 'admin');

CREATE POLICY "No one can update metrics" 
ON metrics FOR UPDATE
USING (false);

CREATE POLICY "Only admins can delete metrics" 
ON metrics FOR DELETE
USING (auth.jwt() ->> 'role' = 'admin');
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

-- RLS for chat sessions
ALTER TABLE chat_sessions ENABLE ROW LEVEL SECURITY;

-- NEW POLICIES
CREATE POLICY "Users can see their own chat sessions" 
ON chat_sessions FOR SELECT
USING (
    auth.uid() IN (
        SELECT auth_id FROM users WHERE id = chat_sessions.user_id
    )
);

CREATE POLICY "Whatsapp service can manage chat sessions" 
ON chat_sessions
USING (auth.jwt() ->> 'role' = 'whatsapp_service');

CREATE POLICY "Admins can manage all chat sessions" 
ON chat_sessions
USING (auth.jwt() ->> 'role' = 'admin');
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

-- RLS for session states
ALTER TABLE session_states ENABLE ROW LEVEL SECURITY;

-- NEW POLICIES
CREATE POLICY "Whatsapp service can manage session states" 
ON session_states
USING (auth.jwt() ->> 'role' = 'whatsapp_service');

CREATE POLICY "Admins can view all session states" 
ON session_states FOR SELECT
USING (auth.jwt() ->> 'role' = 'admin');
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

#### Indexes to optimie queries

```sql
CREATE INDEX idx_appointments_date ON appointments(appointment_date);
CREATE INDEX idx_appointments_user ON appointments(user_id);
CREATE INDEX idx_appointments_status ON appointments(status);
CREATE INDEX idx_metrics_entry_method ON metrics(entry_method);
CREATE INDEX idx_metrics_event_type ON metrics(event_type);
CREATE INDEX idx_chat_sessions_whatsapp ON chat_sessions(whatsapp_id);
```
