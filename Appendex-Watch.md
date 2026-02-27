# **Appendix: Advanced Integration \- Real-Time Event Watching & The AppSheet API**

**Objective:** Explain the transition from "Polling" for events to "Pushing" events in real-time using Google Workspace Watches and direct AppSheet API injection.

**Context for Admins:** In Part 3 of the live session, we demonstrated how to pull ChromeOS login events by periodically checking the Admin SDK Reports API (a process known as *polling*). While effective, polling is rarely immediate. If a device is reported lost, an admin needs to know *instantly* when it comes online.

The provided `Watch.js` and `AppSheetApp.js` scripts take our toolbox to the next level. They shift the architecture from polling to a **Push Notification** model. Instead of asking Google, "Are there new logins?" every 15 minutes, we configure Google to actively shout, "A login just happened\!" the millisecond it occurs. We then use the **AppSheet API** to inject that data directly into the app, bypassing the typical Google Sheets sync delay.

## **1\. The Push Model (Google Workspace Watches)**

Google Workspace allows you to subscribe to specific events (like a device login) by establishing a "Watch" (Notification Channel). When an event occurs, Google sends an HTTP `POST` request to a Webhook URL that you designate.

### **How it works in `Watch.js`:**

1. **The Webhook Listener (`doPost`):** Because our Apps Script is deployed as a Web App (with access set to "Anyone"), it can act as the receiving Webhook. The `doPost(e)` function catches the incoming JSON payloads sent by the Admin Reports API.  
2. **Channel Creation (`createNewWatches_`):** This function tells Google *where* to send the data. It registers a Watch for `CHROME_OS_LOGIN_EVENT` and `CHROME_OS_LOGOUT_EVENT` and points the `address` to our Web App URL.  
3. **The Renewal Cycle (`executeRenewal_`):** Security is paramount. Watch channels for the Reports API expire automatically (maximum 6 hours). The script handles this by calculating the expiration timestamp and programmatically scheduling a Time-Driven Trigger (`scheduleNextRenewal_`) to spin up a new Watch before the old one dies.

**Documentation Links:**

* [Push Notifications for Admin SDK Reports API](https://developers.google.com/workspace/admin/reports/v1/guides/push)  
* [Apps Script Web Apps (`doPost`)](https://developers.google.com/apps-script/guides/web)

## **2\. Direct Injection via the AppSheet API**

By default, AppSheet relies on the underlying data source (like Google Sheets) to trigger a sync. If a webhook writes a new row to a Sheet, an AppSheet user might not see that row until their app runs its next background sync (which can take 30+ minutes).

To achieve true real-time visibility for IT staff or teachers, we must bypass the Sheet and talk directly to AppSheet's backend using the AppSheet API.

### **How it works in `AppSheetApp.js` & `Watch.js`:**

1. **The Library (`AppSheetApp.js`):** This script acts as a wrapper around Google's `UrlFetchApp`. It authenticates using your AppSheet App ID and Application Access Key, formatting your data into the exact JSON structure AppSheet requires.  
2. **The Execution (`doPost`):** When the `doPost` webhook catches a ChromeOS login, it doesn't open the Spreadsheet. Instead, it extracts the `deviceId`, `userEmail`, and `serialNumber`, and builds an `appSheetRows` object.  
3. **The API Call:**

```javascript
// Inside Watch.js doPost()
const app = new AppSheetApp(APPSHEET_APP_ID, APPSHEET_ACCESS_KEY);
const response = app.Add(APPSHEET_TABLE, appSheetRows);
```

4. This `Add` command pushes the data straight to the AppSheet servers. Any device currently running the app will receive the new data almost instantly via AppSheet's real-time broadcast capabilities.

**Documentation Links:**

* [Enable the AppSheet API](https://support.google.com/appsheet/answer/10105398)  
* [Add Records via AppSheet API](https://support.google.com/appsheet/answer/10104797)

## **3\. Initial Script Setup & Configuration**

To successfully deploy the provided scripts, several configuration steps are required within the Google Apps Script editor.

1. **Enable Advanced Google Services:** The script relies on native Workspace APIs. In the Apps Script editor, click the `+` next to **Services** and add the following:  
   * **Admin Directory API** (`admin` / `directory_v1`)  
   * **Admin Reports API** (`admin` / `reports_v1`)  
2. **Configure Script Properties:** To keep credentials secure, they are stored as environment variables. Navigate to **Project Settings** (the gear icon) \> **Script Properties** and add the following key-value pairs:  
   * `APPSHEET_APP_ID`: Your AppSheet application ID.  
   * `APPSHEET_ACCESS_KEY`: The application access key generated in AppSheet.  
   * `WEBHOOK_URL`: *(Added after Step 3\)* The URL of your deployed Web App.  
3. **Deploy as a Web App:**  
   * Click **Deploy** \> **New Deployment**.  
   * Select **Web App** as the type.  
   * *Crucial Settings:* Set "Execute as" to **Me (your admin email)** and "Who has access" to **Anyone**.  
   * Copy the resulting Web App URL and save it to your Script Properties as `WEBHOOK_URL`.  
4. **Initialize the Watch:** Once deployed and configured, manually run the `createNewWatches_` function from the Apps Script editor. This will register the initial webhook channel with Google and programmatically create the time-driven trigger necessary to keep the watch alive continuously.

## **4\. Security & Deployment Considerations**

When implementing this Push/API model, keep the following security practices in mind:

* **Execution Identity:** In the `appsscript.json` manifest, the Web App must be set to `executeAs: USER_DEPLOYING`. This ensures the script has your Admin privileges when receiving payloads or issuing subsequent commands.  
* **Webhook Access:** The access must be set to `ANYONE_ANONYMOUS` so Google's backend servers can successfully hit the webhook without a Google Workspace login prompt blocking the payload.  
* **Protecting Keys:** Never hardcode your `APPSHEET_ACCESS_KEY` in the script body. Notice in `Watch.js` that the key is pulled dynamically using `PropertiesService.getScriptProperties().getProperties()`. This keeps your API keys out of your source code and safely stored in the Apps Script project settings.

## **5\. Source Code**

### **Watch.gs**

```javascript
const PS = PropertiesService.getScriptProperties().getProperties();
const WEB_APP_URL = PS.WEB_APP_URL;

// AppSheet Configuration
const APPSHEET_APP_ID = "913baec7-60d8-4c96-b105-fd4acc6b385b";
const APPSHEET_ACCESS_KEY = PS.APPSHEET_ACCESS_KEY;
const APPSHEET_TABLE = "LoginHistory";

const doPost = (e) => {
  try {
    const json = JSON.parse(e.postData.contents);
    let activities = [];

    // 1. Normalize the data
    if (json.items) {
      activities = json.items; 
    } else if (json.kind === 'admin#reports#activity') {
      activities = [json]; 
    } else if (json.events) {
      activities = [json];
    }

    if (activities.length === 0) {
      return ContentService.createTextOutput('No activities found').setMimeType(ContentService.MimeType.TEXT);
    }

    const appSheetRows = [];

    // 2. Process the activities into AppSheet Objects
    activities.forEach(activity => {
      if (!activity.events || activity.events.length === 0) return;

      const event = activity.events[0];

      if (['CHROME_OS_LOGIN_EVENT', 'CHROME_OS_LOGOUT_EVENT'].includes(event.name)) {

        const getParam = (name) => {
          if (!event.parameters) return 'N/A';
          const param = event.parameters.find(p => p.name === name);
          return param ? param.value : 'N/A';
        };

        const serialNumber = getParam('DEVICE_NAME');
        const userEmail = getParam('DEVICE_USER');
        const reason = getParam('EVENT_REASON');
        const deviceId = getParam('DIRECTORY_DEVICE_ID');

        const finalUser = (userEmail !== 'N/A') ? userEmail : (activity.actor?.email || 'Unknown User');
        
        // Use Google's native unique identifier. This prevents duplicate records 
        // in AppSheet during the brief 1-second watch renewal overlap.
        const uniqueId = activity.id.uniqueQualifier || Utilities.getUuid();

        // Construct the object mapping to AppSheet columns
        appSheetRows.push({
          "ID": uniqueId, 
          "Event Date": new Date(activity.id.time).toISOString(), 
          "Event Type": event.name,
          "User Email": finalUser,
          "Serial Number": serialNumber,
          "Device ID": deviceId,
          "Session Reason": reason,
          "Device Status": "N/A"
        });
      }
    });

    // 3. Write to AppSheet
    if (appSheetRows.length > 0) {
      const app = new AppSheetApp(APPSHEET_APP_ID, APPSHEET_ACCESS_KEY);
      const response = app.Add(APPSHEET_TABLE, appSheetRows);
      console.log(`Success: Sent ${appSheetRows.length} events to AppSheet.`);
    }

    return ContentService.createTextOutput('OK').setMimeType(ContentService.MimeType.TEXT);

  } catch (err) {
    console.error(`Webhook Error: ${err.message}`);
    return ContentService.createTextOutput('Error Handled').setMimeType(ContentService.MimeType.TEXT);
  }
};

/**
 * ============================================================================
 * USER CONTROLS: Run these functions manually from the Apps Script Editor
 * ============================================================================
 */

/**
 * ▶️ Run this function ONCE to start the continuous monitoring cycle.
 */
function startWatchCycle() {
  console.log('🚀 Initializing the watch cycle for the first time...');
  executeRenewal_();
}

/**
 * 🛑 Run this function to completely stop monitoring and clean up all triggers.
 */
function stopWatchCycle() {
  console.log('Halting all operations...');
  const props = PropertiesService.getScriptProperties();
  const EVENTS_TO_WATCH = ['CHROME_OS_LOGIN_EVENT', 'CHROME_OS_LOGOUT_EVENT'];
  
  // Kill active watches
  EVENTS_TO_WATCH.forEach(eventName => {
    const channelId = props.getProperty(`WATCH_CHANNEL_${eventName}`);
    const resourceId = props.getProperty(`WATCH_RESOURCE_${eventName}`);
    
    if (channelId && resourceId) {
      try {
        AdminReports.Channels.stop({ id: channelId, resourceId: resourceId });
        console.log(`🛑 Stopped active watch for: ${eventName}`);
        props.deleteProperty(`WATCH_CHANNEL_${eventName}`);
        props.deleteProperty(`WATCH_RESOURCE_${eventName}`);
        props.deleteProperty(`WATCH_EXPIRATION_${eventName}`);
      } catch (e) {
        console.error(`Error stopping ${eventName}: ${e.message}`);
      }
    }
  });
  
  // Kill active triggers
  const triggers = ScriptApp.getProjectTriggers();
  triggers.forEach(trigger => {
    if (trigger.getHandlerFunction() === 'executeRenewal_') {
      ScriptApp.deleteTrigger(trigger);
    }
  });
  
  console.log('🛑 All watches stopped and renewal triggers deleted.');
}


/**
 * ============================================================================
 * INTERNAL SYSTEM FUNCTIONS: Do not run these manually
 * ============================================================================
 */

/**
 * Creates new watches FIRST, then stops the old ones to prevent data gaps.
 * This is called by the automated trigger.
 */
function executeRenewal_() {
  console.log('🔄 Executing seamless watch renewal...');
  const props = PropertiesService.getScriptProperties();
  const EVENTS_TO_WATCH = ['CHROME_OS_LOGIN_EVENT', 'CHROME_OS_LOGOUT_EVENT'];
  
  // 1. Read the OLD channel data into memory FIRST
  const oldChannels = [];
  EVENTS_TO_WATCH.forEach(eventName => {
    const channelId = props.getProperty(`WATCH_CHANNEL_${eventName}`);
    const resourceId = props.getProperty(`WATCH_RESOURCE_${eventName}`);
    
    if (channelId && resourceId) {
      oldChannels.push({ eventName, channelId, resourceId });
    }
  });

  // 2. Start fresh watches (overwrites the Script Properties with NEW IDs)
  const expirationTimestamp = createNewWatches_();

  // 3. Stop the OLD watches using our saved in-memory data
  oldChannels.forEach(old => {
    try {
      AdminReports.Channels.stop({ id: old.channelId, resourceId: old.resourceId });
      console.log(`🗑️ Cleaned up old watch for: ${old.eventName}`);
    } catch (e) {
      console.error(`Error stopping old watch ${old.eventName}: ${e.message}`);
    }
  });

  // 4. Schedule the next automated renewal
  if (expirationTimestamp) {
    scheduleNextRenewal_(expirationTimestamp);
  } else {
    console.error('❌ Failed to get an expiration timestamp. The cycle has broken.');
  }
}

/**
 * Talks to Google Admin API to setup watches for ChromeOS Login/Logout events.
 * Returns the earliest expiration timestamp (in milliseconds).
 */
function createNewWatches_() {
  const EVENTS_TO_WATCH = ['CHROME_OS_LOGIN_EVENT', 'CHROME_OS_LOGOUT_EVENT'];
  const props = PropertiesService.getScriptProperties();
  
  let earliestExpiration = null;

  EVENTS_TO_WATCH.forEach(eventName => {
    const channelId = Utilities.getUuid();
    const resource = {
      id: channelId,
      type: "web_hook",
      address: WEB_APP_URL,
      params: { ttl: "21600" } // 6 hours
    };

    try {
      const response = AdminReports.Activities.watch(resource, 'all', 'chrome', { eventName: eventName });
      console.log(`✅ Successfully watching: ${eventName} (Channel: ${response.id})`);
      
      props.setProperty(`WATCH_CHANNEL_${eventName}`, response.id);
      props.setProperty(`WATCH_RESOURCE_${eventName}`, response.resourceId);
      props.setProperty(`WATCH_EXPIRATION_${eventName}`, String(response.expiration));

      const expTimestamp = parseInt(response.expiration, 10);
      
      if (!earliestExpiration || expTimestamp < earliestExpiration) {
        earliestExpiration = expTimestamp;
      }

    } catch (e) {
      console.error(`❌ Error watching ${eventName}: ${e.message}`);
    }
  });

  return earliestExpiration;
}

/**
 * Schedules the executeRenewal_ function to run dynamically before the watches expire.
 * @param {number} expirationTimestamp - The epoch time in ms when the watch expires.
 */
function scheduleNextRenewal_(expirationTimestamp) {
  const FUNCTION_NAME = 'executeRenewal_';

  // 1. Clean up any existing triggers for this specific function
  const triggers = ScriptApp.getProjectTriggers();
  triggers.forEach(trigger => {
    if (trigger.getHandlerFunction() === FUNCTION_NAME) {
      ScriptApp.deleteTrigger(trigger);
    }
  });

  // 2. Calculate the trigger time (Expiration minus a 15-minute safety buffer)
  const BUFFER_MS = 15 * 60 * 1000; 
  const nextRunTime = new Date(expirationTimestamp - BUFFER_MS);

  // 3. Create the one-time trigger
  ScriptApp.newTrigger(FUNCTION_NAME)
    .timeBased()
    .at(nextRunTime)
    .create();

  console.log(`🕒 New watches expire at: ${new Date(expirationTimestamp).toISOString()}`);
  console.log(`⏰ Next renewal trigger scheduled for: ${nextRunTime.toISOString()}`);
}
```

### **AppSheetApp.gs**

Copy the code from [https://github.com/mhawksey/AppSheetApp/](https://github.com/mhawksey/AppSheetApp/) 
