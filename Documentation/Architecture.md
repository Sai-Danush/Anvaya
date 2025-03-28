# Appointment Booking System Architecture

## 1. Architecture Options

### Option A: Click to Chat Approach [Alpha]
```
User → wa.me → WhatsApp → Node-RED → Go Backend → PostgreSQL
```

#### Advantages
- **Immediate Engagement**: Users start the process with minimal friction
- **Familiar Interface**: Leverages WhatsApp's familiar environment
- **No Context Switching**: Entire booking happens within WhatsApp

#### Limitations
- **Limited Rich UI Elements**: Constrained by WhatsApp's interface options
- **Slot Visualization**: Harder to display available time slots effectively

---

### Option B: Web Form Approach [Beta]
```
User → React Form → Pre-filled WhatsApp message → WhatsApp → Node-RED → Go Backend → PostgreSQL
```

#### Advantages
- **Structured Input Collection**: Form guides users through the booking process
- **Better Slot Visualization**: Calendar interface shows availability clearly
- **Clear Progress Indicators**: Users see where they are in the booking flow

#### Limitations
- **Additional Step**: Requires initial web form completion
- **Platform Switch**: Users must transition from web to WhatsApp

---

## 2. Performance Metrics

| Metric | Description | Importance |
|--------|-------------|------------|
| Completion Rates | Percentage of users successfully finalizing bookings | High |
| Time to Complete | Total duration from start to confirmation for both options | High |
| Dropout Points | Where users abandon the booking process | Medium |
| User Satisfaction | Post-booking feedback scores | Medium |

**Note**: WhatsApp already includes a native date picker that can be leveraged in both approaches.

---

## 3. Process Flows

### Option A: Click to Chat Flow
1. **Initial Contact**
   - User clicks wa.me link
   - WhatsApp opens with pre-configured message

2. **Date Selection**
   - Bot sends welcome message
   - Presents date picker interface
   - User selects preferred date

3. **Time Slot Selection**
   - System displays available slots for selected date
   - If no slots available, prompts to select another date
   - User selects preferred time slot

4. **Appointment Preview**
   - Shows selected date/time details
   - Requests confirmation to proceed

5. **User Information Collection**
   - Captures name
   - Automatically records WhatsApp number
   - Optional: Collects specific concerns/notes

6. **Confirmation & Reminders**
   - Presents appointment summary
   - Sends confirmation message
   - Sets up calendar invite and reminders

### Option B: Web Form Flow
1. **Form Access**
   - User visits appointment booking webpage
   - System loads interactive booking interface

2. **Multi-Step Form Process**
   - **Step 1**: Calendar-based date selection with real-time availability
   - **Step 2**: Visual time slot picker for selected date
   - **Step 3**: Personal details form (name, WhatsApp number, notes)

3. **WhatsApp Handoff**
   - Form generates pre-filled WhatsApp message with appointment details
   - System opens WhatsApp with the appointment information

4. **Confirmation Process**
   - User receives booking confirmation in WhatsApp
   - System sends calendar invite and sets reminders
   - Optional: Follow-up messages with preparation instructions

---

## 4. Implementation Considerations

### Technical Components
- **Frontend**: React with calendar/time selection components
- **Messaging**: WhatsApp Business API integration
- **Workflow**: Node-RED for conversation flow management
- **Backend**: Go services for business logic
- **Database**: Supabase for appointment and user data storage


