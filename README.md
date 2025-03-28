# Anvaya Psychologist Booking System - Implementation Plan

## Project Overview
Anvaya is a scheduling/booking system with dual interfaces:
1. **Patient Interface**: WhatsApp-based booking system using click-to-chat
2. **Psychologist Interface**: Web dashboard for appointment management and billing

## System Architecture
```
Patient Flow:
User → wa.me link → WhatsApp → Node-RED → Go Backend → Supabase

Psychologist Flow:
Psychologist → React Dashboard → Go Backend → Supabase
```

## Implementation Phases

### Phase 1: Database Setup
- **Objective**: Implement database schema in PostgreSQL
- **Tasks**:
  - Set up PostgreSQL server instance
  - Create database and user roles with appropriate permissions
  - Create tables following defined schema (users, psychologists, availability, appointments, etc.)
  - Implement custom authentication system for both patient and psychologist roles
  - Create database-level security policies (or application-level alternatives)
  - Create necessary indexes for query optimization
  - Test database relationships with sample data

### Phase 2: Go Backend Development
- **Objective**: Create robust API services for both interfaces
- **Tasks**:
  - Set up Go project structure
  - Implement user authentication and management
  - Create appointment scheduling services
  - Build availability management systems
  - Develop conflict resolution for scheduling
  - Create business logic for scheduling constraints
  - Implement notification services

### Phase 3: WhatsApp Integration with Node-RED
- **Objective**: Create conversational booking flow for patients
- **Tasks**:
  - Set up WhatsApp Business API connection
  - Configure Node-RED for conversation management
  - Design conversation flow for appointment booking
  - Implement date selection interface
  - Create time slot selection process
  - Handle appointment confirmation
  - Set up appointment reminders
- **Key Considerations**:
  - User experience within messaging constraints
  - Error handling and fallback mechanisms
  - State management for interrupted conversations

### Phase 4: Psychologist Dashboard Development
- **Objective**: Create web interface for psychologists to manage appointments
- **Tasks**:
  - Set up React project with authentication
  - Create appointment calendar views (daily/weekly/monthly)
  - Implement availability management interface
  - Build patient information viewing system
  - Develop appointment details and notes functionality
  - Create notification system for new bookings
  - Integrate with Google Calendar API for schedule synchronization
- **Key Considerations**:
  - Responsive design for multiple devices
  - Intuitive UX for calendar management
  - Real-time updates for new appointments
  - Handling calendar permissions and authentication

### Phase 5: Google Calendar Integration
- **Objective**: Synchronize appointments with Google Calendar
- **Tasks**:
  - Set up Google Calendar API credentials
  - Implement OAuth 2.0 authentication flow
  - Create calendar event creation functionality
  - Build two-way synchronization between system and Google Calendar
  - Implement automatic updates when appointments change
  - Add calendar invitation sending to patients
- **Key Considerations**:
  - Handling OAuth token refresh and storage
  - Managing calendar permissions securely
  - Preventing synchronization conflicts
  - Respecting user privacy and data protection

### Phase 6: Billing System Implementation
- **Objective**: Add financial tracking capabilities
- **Tasks**:
  - Create invoice generation functionality
  - Implement payment status tracking
  - Build payment reminder system
  - Develop financial reporting views
  - Create receipt generation
  - Implement outstanding payment tracking
- **Key Considerations**:
  - Secure handling of financial data
  - Clear visualization of payment status
  - Automated reminder systems

## Technical Components

### Frontend (React Dashboard)
- Calendar visualization component
- Patient management interface
- Appointment scheduling tools
- Billing and invoice management
- Data visualization for reports
- Google Calendar integration component

### Backend (Go)
- REST API endpoints for all services
- Authentication middleware
- Business logic for scheduling
- Integration with Supabase
- Notification service

### Messaging (WhatsApp via Node-RED)
- Conversational flow management
- Date and time selection handlers
- Confirmation processes
- Reminder system

### Database (Supabase)
- User authentication
- Data storage for all entities
- Row-level security
- Real-time subscription capabilities

## Future Enhancements (Post-MVP)
- Analytics dashboard for psychologists
- Patient portal web interface
- Online payment integration
- Advanced reporting capabilities
- Multi-psychologist practice management