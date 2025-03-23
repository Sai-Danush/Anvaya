## Architecture

### Click to Chat Approach [Alpha]
User —> wa.me —> WhatsApp —> Node-RED —> Go Backend —> Supabase

Pros - Immediate Engagement, Familiar interface, no context switching
Cons -  Limited Rich UI elements, Harder to show available time slots 

### Web Form Approach [Beta]
User -> React Form -> Pre-filled WhatsApp message -> WhatsApp -> Node-RED -> Go Backend -> Supabase

Pros: Structured Input Collection, Better Visualization of Time Slots, Clear Progress Indicators
Cons: Additional Step, Platform Switch

Parameters/Metrics to keep in track:
1. Completion Rates: Users who are actually completing the bookings
2. Time to Complete Booking through Option Alpha/Beta

We could keep a date picker, so there is native date picker already present in whatsapp

### Process Flows

#### Option A[Click to Chat]
1. User clicks wa.me link
   → Opens WhatsApp with pre-configured message

2. Bot Welcome & Date Selection
   → Shows date picker
   → User selects date

3. Time Slot Selection
   → If slots available: Shows slot buttons
   → If no slots: Back to date picker
   → User selects time

4. Confirmation
   → Shows selected date/time
   → Asks for confirmation

5. User Details Collection
   → Name
   → Contact number (auto-captured)
   → Any specific concerns (optional)

6. Final Confirmation
   → Summary of appointment
   → Confirmation message
   → Calendar invite/reminder setup

#### Option B[Web Form]
1. User visits web form
   → Loads appointment booking interface

2. Form Steps:
   a. Date Selection
      → Calendar interface
      → Real-time slot availability display
   
   b. Time Slot Selection
      → Shows available slots for selected date
      → Visual time slot picker
   
   c. Personal Details
      → Name
      → WhatsApp number
      → Optional notes

3. Form Submission
   → Generates pre-filled WhatsApp message
   → Opens WhatsApp with appointment details

4. WhatsApp Confirmation
   → User receives confirmation
   → Gets calendar invite/reminder