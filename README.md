# Google Calendar to Office 365 One-Way Sync

An n8n workflow that provides one-way synchronization from your Google Calendar to your Office 365 calendar.

## Features

- **Real-time sync** using Google Calendar triggers
- **Handles all event operations:**
  - ✅ New events created in Google Calendar
  - ✅ Updated events in Google Calendar
  - ✅ Cancelled/deleted events from Google Calendar
- **Preserves all event details:**
  - Event title/subject
  - Description/body
  - Start and end times (including all-day events)
  - Location
  - Time zones
  - Privacy settings
- **Google Meet/Hangouts integration:**
  - Video meeting links
  - Phone dial-in numbers with PINs
  - SIP addresses
  - Meeting codes
- **Automatic categorization:** All synced events are tagged with the "Strike Team" category
- **Safe operation:** Only manages events it creates - never touches your existing Office 365 events

## Prerequisites

1. An n8n instance (self-hosted or n8n Cloud) - tested with versions 1.21.3 and 2.2.6
2. Google Calendar OAuth2 credentials configured in n8n
3. Microsoft Outlook/Office 365 OAuth2 credentials configured in n8n
4. A category named "Strike Team" created in your Office 365 calendar

## Initial Sync of Existing Events

This workflow only syncs **new events going forward**. To import existing Google Calendar events to Office 365, use Google Calendar's ICS export feature:

1. Go to [Google Calendar Settings](https://calendar.google.com/calendar/r/settings)
2. Click on your calendar → **Export calendar**
3. In Outlook, go to **File** → **Open & Export** → **Import/Export**
4. Select **Import an iCalendar (.ics) file**
5. Choose the exported file and import

## Installation

### Step 1: Import the Workflow

1. Open your n8n instance
2. Go to **Workflows** → **Add Workflow** → **Import from File**
3. Select `google-calendar-to-office365-sync.json`
4. Click **Import**

### Step 2: Configure Credentials

After importing, you'll need to connect your credentials:

#### Google Calendar Credentials
1. Click on each **Google Calendar Trigger** node and select your Google Calendar OAuth2 credentials
2. Click on each **Get Full Event Details** HTTP Request node and select your Google Calendar OAuth2 credentials

#### Microsoft Outlook Credentials
1. Click on each **HTTP Request** node that calls Microsoft Graph API
2. Select your Microsoft Outlook OAuth2 credentials

### Step 3: Configure Your Calendars

The workflow includes **⚙️ CONFIGURATION** nodes where you can easily set:

| Setting | Description | Default |
|---------|-------------|---------|
| `googleCalendarId` | The Google Calendar to sync FROM. Use `primary` for your main calendar, or the calendar ID from Google Calendar settings. | `primary` |
| `office365CalendarName` | The name of the Office 365 calendar to sync TO. | `Calendar` |
| `categoryName` | The category to apply to synced events. Must exist in Office 365. | `Strike Team` |

**To find your Google Calendar ID:**
1. Go to Google Calendar settings
2. Click on your calendar
3. Scroll to "Integrate calendar"
4. Copy the Calendar ID

**Note:** You'll see 3 CONFIG nodes (one for each trigger). Keep all three in sync with the same values.

### Step 4: Activate the Workflow

1. Click the **Inactive** toggle in the top-right corner to activate the workflow
2. The workflow will now listen for events from your Google Calendar

## How It Works

### Event Creation Flow
```
Google Calendar Trigger (eventCreated)
    ↓
Get Full Event Details (HTTP Request)
    - Fetches complete event including Google Meet data
    ↓
Transform for Create (JavaScript)
    - Extracts all event details
    - Formats Google Meet information as HTML
    - Adds tracking ID: [GCAL_SYNC_ID:xxx]
    ↓
Create Event in Office 365 (HTTP Request)
    - Creates event with "Strike Team" category
```

### Event Update Flow
```
Google Calendar Trigger (eventUpdated)
    ↓
Get Full Event Details (HTTP Request)
    ↓
Transform for Update (JavaScript)
    ↓
Get Synced Events (HTTP Request)
    - Fetches ONLY events with "Strike Team" category
    ↓
Find Matching Event (JavaScript)
    - Searches for tracking ID in event body
    ↓
[If Found] → Update Event in Office 365
[If Not Found] → Create Event in Office 365
```

### Event Cancellation/Deletion Flow
```
Google Calendar Trigger (eventCancelled)
    ↓
Extract Delete ID (JavaScript)
    ↓
Get Synced Events (HTTP Request)
    - Fetches ONLY events with "Strike Team" category
    ↓
Find Event to Delete (JavaScript)
    - Searches for tracking ID in event body
    ↓
[If Found] → Delete Event from Office 365
[If Not Found] → Skip (nothing to delete)
```

## Event Tracking

The workflow embeds a tracking ID in the Office 365 event body. This allows the workflow to:
- Find and update the correct Office 365 event when a Google event is modified
- Find and delete the correct Office 365 event when a Google event is cancelled

The tracking ID appears at the bottom of the event body as:
```
[GCAL_SYNC_ID:google_event_id]
```

## Safety Features

- **Category-based filtering**: Update and delete operations ONLY search events with the "Strike Team" category
- **Tracking ID matching**: Events are matched by their embedded tracking ID, not by title or time
- **Your existing events are safe**: Events without the "Strike Team" category are never touched

## Google Meet Information

When a Google Calendar event includes a Google Meet video conference, the following information is added to the Office 365 event body:

- **🎥 Google Meet Link**: Direct video meeting URL
- **Conference Details**:
  - 📹 Video link with meeting code
  - 📞 Phone dial-in numbers with PINs
  - 🖥️ SIP addresses for room systems

## Troubleshooting

### Events not syncing
1. Check that the workflow is **Active**
2. Verify credentials are properly connected and not expired
3. Check the **Executions** tab for any error messages

### Update/Delete not working
1. Verify the "Strike Team" category exists in Office 365
2. Check that synced events have the `[GCAL_SYNC_ID:xxx]` tracking marker in their body
3. Look at the execution details to see debug output (number of Strike Team events found, tracking ID being searched)

### Wrong calendar being synced
1. Verify the `calendarId` in the trigger nodes matches your Google Calendar name or ID
2. The Google Calendar API uses "primary" for your main calendar

### Duplicate events
If you're seeing duplicate events:
1. The update flow couldn't find the matching event (tracking ID mismatch)
2. Delete duplicates manually from Office 365
3. Ensure the tracking ID in event bodies isn't modified

### Credential errors
1. Re-authenticate your Google and Microsoft credentials in n8n
2. Ensure the OAuth scopes include calendar read/write permissions

## Limitations

- **One-way sync only**: Changes in Office 365 are not synced back to Google Calendar
- **Recurrence**: Recurring events are synced as individual instances
- **Attachments**: File attachments are not synced
- **Attendees**: Attendee information is not synced to avoid sending duplicate invites

## License

This workflow is provided as-is for personal and organizational use.
