# **Session Guide: The Automated IT Toolbox**

**Speaker:** Martin Hawksey  
**Event:** Google for Education IT Admin Summit 2026  
**Slides:** [IT Admin Summit 2026 - Delegated Power: Building an Admin Toolbox](https://docs.google.com/presentation/d/12kUVf7Qo1KEvVxNhgea-36ryz5U3KP3A8Kuu4XTEAYA/edit?slide=id.g2b4e78bce1c_0_1501#slide=id.g2b4e78bce1c_0_1501)  
**Duration:** 50 Minutes

# **Introduction: The "Admin Debt" Problem**

**Objective:** Define the session's scope and the "Admin Toolbox" concept.

**Dialogue:** "Welcome everyone. We have 50 minutes to turn the Google Admin Console from a static interface into a dynamic, automated engine. We are building an 'Admin Toolbox'. We will start with a basic directory sync, move to hierarchical security filters, and finish by using the AppSheet API to keep our data fresh without lifting a finger."

* **Context:** Explain that the Admin Console is powerful, yet it often acts as a bottleneck for delegated tasks. We either grant too much access or we do everything ourselves: a situation that creates "Admin Debt".

# **Part 1 Building a Directory App**

**Dialogue:** "We begin with the foundation. Every admin needs a map of their users: but a static list buried inside the Admin Console is difficult to use for delegated apps. We need a live connection. We are building a data bridge that turns raw directory records into a functional database that understands who people are and, more importantly, who they report to." [Directory API Overview | Admin console | Google for Developers](https://developers.google.com/workspace/admin/directory/v1/guides) 

## **Step 1: The Initial Sync**

**Objective:** Move data from the Admin SDK to a Google Sheet.

**Dialogue:** "I have a blank Google Sheet. We need our user data here. Instead of a manual export, we use Apps Script to talk to the Admin SDK. I am going to ask Gemini to help me write the boilerplate for a basic directory fetch."

* **Action:** Open [gemini.google.com](http://gemini.google.com) and use the following prompt.

> **Gemini Prompt 1:** \> "Write a modern Google Apps Script function called syncBasicDirectory that uses the Admin SDK Directory API to list all users in my domain. It should clear the sheet named 'Directory' and write the 'Name', 'Email', 'Department', 'Title', 'Org Unit', 'Manager Email' of each user"

### **Code snippet**

```javascript
/**
 * Fetches all users in the Google Workspace domain and writes their basic
 * directory information to a sheet named 'Directory'.
 */
function syncBasicDirectory() {
  const sheetName = 'Directory';
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(sheetName);
  
  // Create the sheet if it doesn't exist, otherwise clear existing data
  if (!sheet) {
    sheet = ss.insertSheet(sheetName);
  } else {
    sheet.clear();
  }
  
  const headers = ['Name', 'Email', 'Department', 'Title', 'Org Unit', 'Manager Email'];
  const rows = [];
  
  let pageToken;
  
  // Loop through all pages of users in the domain
  do {
    const response = AdminDirectory.Users.list({
      customer: 'my_customer',
      maxResults: 500,
      projection: 'full', // 'full' is required to get organizations and relations data
      pageToken: pageToken
    });
    
    const users = response.users || [];
    
    users.forEach(user => {
      const name = user.name?.fullName || '';
      const email = user.primaryEmail || '';
      const orgUnit = user.orgUnitPath || '/';
      
      // Extract Organization Info (Department, Title)
      // Defaults to the primary organization if multiple exist
      const primaryOrg = user.organizations?.find(org => org.primary) || user.organizations?.[0];
      const department = primaryOrg?.department || '';
      const title = primaryOrg?.title || '';
      
      // Extract Manager Email from relations
      const managerRelation = user.relations?.find(rel => rel.type === 'manager');
      const managerEmail = managerRelation?.value || '';
      
      rows.push([name, email, department, title, orgUnit, managerEmail]);
    });
    
    pageToken = response.nextPageToken;
  } while (pageToken);
  
  // Write everything to the sheet
  // 1. Write headers
  sheet.appendRow(headers);
  sheet.getRange(1, 1, 1, headers.length).setFontWeight('bold'); // Format headers
  
  // 2. Batch write all user data for optimal performance
  if (rows.length > 0) {
    sheet.getRange(2, 1, rows.length, headers.length).setValues(rows);
  }
}
```

## **Step 2: Build the App**

**Objective:** Generate the basic AppSheet interface.

**Dialogue:** "**Important:** Delete Sheet1 (AppSheet doesn’t like a blank first tab) \- Now we have data. We hit 'Extensions' \> 'AppSheet' \> 'Create an App'. In sixty seconds, we have a responsive directory app. But here is the problem: we need a hierarchy to securely show related contacts."

### **Key Concept \- AppSheet UI**

The AppSheet editor has three primary sections:

1. The **quick nav menu**, which takes you to the various areas of the editor.  
2. The main **editor body** \- this is where you can modify your app’s definition.   
3. The **emulator** \- which shows you a live interactive preview of your application as you build

## **Step 3: The Reporting Path**

**Objective:** Flatten the organisational structure for mobile performance.

**Dialogue:** "Our directory has a 'Managed By' field. We want to turn that into a full reporting chain like CEO \> Manager \> Staff. This allows us to apply a security filter so managers only see their team. Let's iterate on our script."

> **Gemini Prompt 2:** \> "Update the previous script to fetch the 'manager' relation for each user. Create an additional column called 'Reporting Path'. For each user, recursively trace their managers up to the top and join them with '-\>' to create a string like '[boss@school.com](mailto:boss@school.com)\-\>manager@school.com-\>user@school.com'."

### **Code Snippet**

```javascript
/**
 * Fetches all users, maps their reporting structures, and writes 
 * the data and full reporting paths to the 'Directory' sheet.
 */
function syncBasicDirectory() {
  const sheetName = 'Directory';
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(sheetName);
  
  if (!sheet) {
    sheet = ss.insertSheet(sheetName);
  } else {
    sheet.clear();
  }
  
  const headers = ['Name', 'Email', 'Department', 'Title', 'Org Unit', 'Manager Email', 'Reporting Path'];
  
  let pageToken;
  const userDataList = [];
  const managerMap = new Map(); // Stores the email -> managerEmail relationship
  
  // 1. Fetch all users and map their direct managers
  do {
    const response = AdminDirectory.Users.list({
      customer: 'my_customer',
      maxResults: 500,
      projection: 'full',
      pageToken: pageToken
    });
    
    const users = response.users || [];
    
    users.forEach(user => {
      const name = user.name?.fullName || '';
      const email = user.primaryEmail || '';
      const orgUnit = user.orgUnitPath || '/';
      
      const primaryOrg = user.organizations?.find(org => org.primary) || user.organizations?.[0];
      const department = primaryOrg?.department || '';
      const title = primaryOrg?.title || '';
      
      const managerRelation = user.relations?.find(rel => rel.type === 'manager');
      const managerEmail = managerRelation?.value || '';
      
      // Save data for later processing
      userDataList.push({ name, email, department, title, orgUnit, managerEmail });
      
      // Add to our lookup map
      managerMap.set(email, managerEmail);
    });
    
    pageToken = response.nextPageToken;
  } while (pageToken);
  
  // 2. Helper function to trace the reporting path upwards
  function getReportingPath(userEmail) {
    const path = [userEmail];
    let currentManager = managerMap.get(userEmail);
    const visited = new Set([userEmail]); // Used to detect infinite loops/circular references
    
    while (currentManager && currentManager !== '') {
      // Prevent infinite loops if users are accidentally set as each other's managers
      if (visited.has(currentManager)) {
        path.unshift(`[Cycle Detected: ${currentManager}]`);
        break;
      }
      
      path.unshift(currentManager);
      visited.add(currentManager);
      
      // Move up one level in the hierarchy
      currentManager = managerMap.get(currentManager);
    }
    
    return path.join('->');
  }
  
  // 3. Build the final rows array
  const rows = userDataList.map(user => {
    const reportingPath = getReportingPath(user.email);
    return [
      user.name, 
      user.email, 
      user.department, 
      user.title, 
      user.orgUnit, 
      user.managerEmail, 
      reportingPath
    ];
  });
  
  // 4. Write data to the sheet
  sheet.appendRow(headers);
  sheet.getRange(1, 1, 1, headers.length).setFontWeight('bold');
  
  if (rows.length > 0) {
    sheet.getRange(2, 1, rows.length, headers.length).setValues(rows);
  }
}

```

## **Step 4: Security Filters**

**Context:** If you are following along you might need to skip this step if your Directory data doesn’t have a suitable attribute populated. 

**Objective:** Apply the granular access control.

**Dialogue:** "Back in AppSheet, we regenerate the columns. We now have a 'Reporting Path'. In the Table Setting we can make our data Read-Only and then go to Security \> Security Filters and add a simple expression: `CONTAINS([Reporting Path], USEREMAIL())`. Now the app is personalised and secure."

### **Key Concept \- AppSheet Security Filters**

AppSheet Security Filters execute server side removing the risk of inappropriate data being sent to your app and also can be used to improve your app performance. [Security filters: The Essentials \- AppSheet Help](https://support.google.com/appsheet/answer/10104488) 

# **Part 2: Share Drive approval and creation**

**Dialogue:** "Data is just the beginning. Now we put that directory to work. One of the most common tickets for any IT team is a request for a new Shared Drive: usually involving manual naming, manual member assignment, and manual oversight. We can change that. We are going to build a system where the app does the heavy lifting: ensuring naming standards and permissions are correct every single time."

## **Step 1: Creating dummy data**

**Objective:** Use Gemini to create a realistic test environment with complex permissions.

**Dialogue:** "Before we automate, we need to test. I want to generate dummy data that includes lists of people with different access levels. I'll ask Gemini to build a test sheet that includes columns for Content Managers and Contributors."

> **Gemini Prompt 3:** \> "Can you write a modern Google Apps Script function called generateDummyRequests to create dummy data for an AppSheet app. It should populate a sheet called 'DriveRequests' with 10 rows. Headers should be ‘RequestID’, 'Shared Drive Name', 'Requester Email', ‘Manager Email’ 'Content Managers', 'Contributors', 'Status', and 'Drive ID'. Use realistic school project names. 'Content Managers' and 'Contributors' should contain comma-separated lists of dummy emails. Randomly set the status to 'Approved', 'Pending' or 'Declined'. I've provided my existing Google Sheet for context."

### **Code Snippet**

```javascript
/**
 * Generates 10 rows of dummy data for an AppSheet app
 * and writes it to a sheet named 'DriveRequests'.
 */
function generateDummyRequests() {
  const sheetName = 'DriveRequests';
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(sheetName);
  
  // Create or clear the sheet
  if (!sheet) {
    sheet = ss.insertSheet(sheetName);
  } else {
    sheet.clear();
  }
  
  const headers = [
    'RequestID', 'Shared Drive Name', 'Requester Email', 
    'Manager Email', 'Content Managers', 'Contributors', 
    'Status', 'Drive ID'
  ];
  
  // Dummy data pools
  const projects = [
    '2026 Yearbook Committee', 
    'Spring Musical Production', 
    'Varsity Debate Team', 
    'Robotics Club - FIRST 2026', 
    'Science Fair Planning', 
    'Student Council Archives', 
    'Math Department Faculty', 
    'PTA Spring Fundraiser', 
    'Senior Prom Committee', 
    'Athletics Media Resources'
  ];
  
  const statuses = ['Approved', 'Pending', 'Declined'];
  const names = ['alice', 'bob', 'charlie', 'diana', 'evan', 'fiona', 'george', 'hannah', 'ian', 'julia'];
  
  // Helper functions for data generation
  const getRandomEmail = () => `${names[Math.floor(Math.random() * names.length)]}@school.edu`;
  
  const getEmailList = (count) => {
    return Array.from({ length: count }, getRandomEmail).join(', ');
  };

  const getFakeDriveId = () => {
     const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_';
     const randomString = Array.from({ length: 17 }, () => chars[Math.floor(Math.random() * chars.length)]).join('');
     return '0A' + randomString; // Google Drive Shared Drive IDs often start with 0A
  };
  
  const rows = [];
  
  // Generate exactly 10 rows
  for (let i = 0; i < 10; i++) {
    const requestId = Utilities.getUuid(); // Perfect for AppSheet key columns
    const driveName = projects[i]; 
    const requesterEmail = getRandomEmail();
    const managerEmail = getRandomEmail();
    const contentManagers = getEmailList(2); // Generates 2 comma-separated emails
    const contributors = getEmailList(3);    // Generates 3 comma-separated emails
    const status = statuses[Math.floor(Math.random() * statuses.length)];
    
    // Only generate a Drive ID if the request is 'Approved'
    const driveId = status === 'Approved' ? getFakeDriveId() : '';
    
    rows.push([
      requestId, 
      driveName, 
      requesterEmail, 
      managerEmail, 
      contentManagers, 
      contributors, 
      status, 
      driveId
    ]);
  }
  
  // Write data to the sheet
  sheet.appendRow(headers);
  sheet.getRange(1, 1, 1, headers.length).setFontWeight('bold');
  sheet.getRange(2, 1, rows.length, headers.length).setValues(rows);
  
  // Auto-resize columns to make the dummy data easily readable
  sheet.autoResizeColumns(1, headers.length);
}
```

## **Step 2: Integrating the Request System**

**Objective:** Connect the automation sheet to AppSheet with relational logic and security.

**Dialogue:** "Our spreadsheet is ready, but we need to wire it into the AppSheet interface. We add the DriveRequests table as a new data source. We change the Requester Email to a 'Ref' type. Think of a Reference as a digital bridge between tables: it allows AppSheet to 'look up' information in the Directory based on the email address. We then set the initial value of the Manager Email column to pull directly from the Directory using a lookup expression. This means when a user starts a request, the app instantly identifies their manager without any manual input. We also apply a security filter so users only see their own requests or those they are responsible for approving. To make the process even smoother for our managers: we will add a reference form for approvals and set up quick action buttons. This allows them to approve or deny a request with a single tap. Finally, we create a 'Shared Drives' view to give our users a dedicated place to manage these interactions."

* **Action:**   
  * Update some of the Manager Email values to your own email address  
  * Add `DriveRequests` as a data source in the AppSheet editor.  
  * In Data \> Columns, update `Requester Email` type to `Ref` (pointing to the `Directory` table) and Initial Value `USEREMAIL()`.  
  * Set the 'Initial Value' of `Manager Email` to `[Requester Email].[Manager Email]`.  
  * In the Table Setting remove Deletes and go to Security \> Security Filters and add: `OR([Requester Email] = USEREMAIL(), [Manager Email] = USEREMAIL())`.  
  * Go to Views and create a new 'Shared Drives' view.  
  * Add a reference view for 'Shared Drive Approvals' as a Detail view.  
  * Add Status action buttons to set the Status to 'Approved' or 'Denied' on Drive Requests data.

### **Key Concept \- AppSheet Table Relationships**

Now we will explore the concept of Ref (reference) column types. This allows us to establish relationships between tables.   

A reference is a link between a column of one table and the key of another table \- indicating that the two connected pieces of data are related to one another. Another way to look at it is the REF column type will provide context for a particular value that matches the key value in the source table. 

In the example below if we look at our “Requests” and “Employees” table \- we notice that the information contained in the User column of the Requests table maps to the User column found in the Users table.

We can use References to relate these two tables and draw line of sight between related data points across tables. 

## **Step 3: The Approval Workflow**

**Objective:** Enable managers to approve or decline requests directly within their Gmail inbox using AppSheet's Dynamic Email (AMP) capability.

**Dialogue:** "We want to make approval as frictionless as possible. If a manager has to leave their inbox, log into an app, and find a record, that's where the process stalls. By using AppSheet's Dynamic Email, we send the app *to* them. They see the request details and the 'Approve' or 'Decline' buttons right inside the message. One tap in Gmail, and the data updates in our Sheet instantly."

* **Action:**  
  * **Create Bot:** Go to **Automation** \> **Bots** \> **New Bot**. Name it "Send Approval Email".  
  * **Event:** Data Change \> Adds Only \> Table `DriveRequests`.  
  * **Step 1 (Send Email):**  
    * **Settings:** "Send an email".  
    * **To:** `[Manager Email]`  
    * **Use Dynamic Email:** Toggle this **ON**.  
    * **Display Table:** Select `DriveRequests`.  
    * **App View:** Choose the detail view for the request `Shared Drive Approvals`. This ensures the "Approve" and "Decline" actions are visible in the email.  
    * **Subject:** `Shared Drive Request - <<[Shared Drive Name]>>`

## **Step 4: Automated Provisioning**

**Objective:** Replace manual drive creation with an automated script triggered by the AppSheet approval.

**Dialogue:** "The manager has tapped 'Approve'. In a traditional workflow, that triggers an email to IT. In our Toolbox, it triggers a script. We’re going to use the Google Drive API to create the drive, name it according to our project standards, and automatically add the right people with the right permissions. No more copy-pasting email addresses or forgetting to set someone as a 'Contributor' instead of a 'Manager'."

> **Gemini Prompt 4:** \> "Write a modern Google Apps Script function called `createSharedDriveFromAppSheet` that accepts strings containing `driveName`, `requesterEmail`, `contentManagers`, and `contributors`. It should create a Shared Drive, add the requester as an 'organizer', and add the members from the comma-separated strings as 'fileOrganizer' and 'writer' using the most efficient method. Return the Drive ID."

* **Action:**  
  * To use this function in an AppSheet automation we need to create a standalone script (a quick way to do this is [script.new](https://script.new))   
  * Before running or calling this script from AppSheet, you must enable the **Drive API** Advanced Service:  
* In the Apps Script editor, click on **Services** (the `+` icon) on the left sidebar.  
* Scroll down and select **Drive API**.   
  * Note: The service will default to `v3` but Gemini still usually thinks Apps Script uses the Drive API `v2`  
* Click **Add**.

### **Code Snippet**

```javascript
/**
 * Creates a Shared Drive and assigns roles to the requester and other users,
 * with retry logic to wait for organizational policies to apply.
 * * @param {string} driveName - The name of the Shared Drive to create.
 * @param {string} requesterEmail - Email of the user requesting the drive.
 * @param {string} contentManagers - Comma-separated list of emails.
 * @param {string} contributors - Comma-separated list of emails.
 * @returns {string} The ID of the newly created Shared Drive.
 */
function createSharedDriveFromAppSheet(driveName, requesterEmail, contentManagers, contributors) {
  // 1. Create the Shared Drive
  const requestId = Utilities.getUuid();
  const drive = Drive.Drives.insert({ name: driveName }, requestId);
  const driveId = drive.id;
  
  // Optional: Add a small initial pause to give Google a head start
  Utilities.sleep(2000); 
  
  // 2. Helper function to parse comma-separated emails
  const parseEmails = (emailString) => {
    if (!emailString) return [];
    return emailString.split(',')
      .map(email => email.trim())
      .filter(email => email.length > 0);
  };
  
  // 3. Helper function with Retry Logic (Exponential Backoff)
  const addPermissionWithRetry = (email, role, maxRetries = 4) => {
    let attempt = 0;
    
    while (attempt < maxRetries) {
      try {
        Drive.Permissions.insert({
          role: role,
          type: 'user',
          value: email 
        }, driveId, {
          supportsAllDrives: true,
          sendNotificationEmails: false
        });
        
        // If successful, exit the retry loop
        return; 
        
      } catch (error) {
        attempt++;
        const errorMessage = error.message || '';
        
        // Check if it's the specific policy/timing error
        if (errorMessage.includes('policy has been applied') || errorMessage.includes('Forbidden')) {
          if (attempt >= maxRetries) {
            console.error(`Failed to add ${email} as ${role} after ${maxRetries} attempts:`, errorMessage);
            break; // Give up after max retries
          }
          // Wait 2 seconds, then 4 seconds, then 6 seconds...
          Utilities.sleep(2000 * attempt); 
        } else {
          // If it's a different error (like an invalid email address), log it and don't retry
          console.error(`Error adding ${email} as ${role}:`, errorMessage);
          break;
        }
      }
    }
  };

  // 4. Consolidate all users and roles into a single processing list
  const permissionsToApply = [];
  
  if (requesterEmail) {
    permissionsToApply.push({ email: requesterEmail.trim(), role: 'organizer' });
  }
  
  parseEmails(contentManagers).forEach(email => {
    permissionsToApply.push({ email, role: 'fileOrganizer' });
  });
  
  parseEmails(contributors).forEach(email => {
    permissionsToApply.push({ email, role: 'writer' });
  });

  // 5. Apply all permissions using the new retry function
  permissionsToApply.forEach(user => addPermissionWithRetry(user.email, user.role));
  
  return driveId;
}
```

### **Key Concept \- Exponential Backoff & Propagation Delays**

When you create a resource (like a Shared Drive) and immediately try to assign permissions to it, Google's backend sometimes needs a few seconds to sync. If you just let the script fail, the user doesn't get access. We solve this using "Exponential Backoff" in our `addPermissionWithRetry` function. If the API throws a timing error, the script catches it, waits 2 seconds, tries again, then waits 4 seconds, etc. This is a vital technique for building resilient IT automations that don't randomly fail due to API rate limits or sync delays.

## **Step 5: Connecting the "Bot"**

**Objective:** Finalise the AppSheet Automation loop.

**Dialogue:** "Now we connect the dots. In AppSheet, we go to **Automation**. We create a new Bot triggered when a `DriveRequests` record is updated and the `Status` equals 'Approved'.

We add a step called 'Call Provisioning Script'. We point it to our Apps Script file and select the `createSharedDriveFromAppSheet` function. We map our columns—Shared Drive Name, Requester Email, etc.—to the script parameters.

Finally, we add one more step: 'Update Drive ID'. We take the return value from the script and write it back to our spreadsheet. The loop is closed. The requester gets their drive, and we have a perfect audit trail. To only trigger this when a Status is Approved we will ask Gemini for help"

> **Gemini Prompt 5:** \> “In my AppSheet app I have a table called DriveRequests with a column called Status. I would like to trigger this function in an automation only when the Status is approved and there is no Drive ID. How can I do this?” 

**Action Steps in AppSheet:**

* **Create Bot:** New Bot \> "When a DriveRequest is Approved".  
* **Event:** Data Change \> Updates Only \> Table `DriveRequests`. Condition: `AND([Status] = "Approved", ISBLANK([Drive ID]))`.  
* **Call Script:**   
  * Name `Create Shared Drive`  
  * Settings: "Call a script".  
  * Function: `createSharedDriveFromAppSheet`.  
  * Inputs:  
    * `driveName`: `[Shared Drive Name]`  
    * `requesterEmail`: `[Requester Email]`  
    * `contentManagers`: `[Content Managers]`  
    * `contributors`: `[Contributors]`  
* **Step 2 (Update Record):**  
  * Settings: "Run a data action" \> "Set row values".  
  * Column: `Drive ID`.  
  * Value: `[Create Shared Drive].[Output]`

### **Key Concept \- AppSheet Automations with Apps Script**

You'll need to authorise the first time you access the project and anytime authentication scopes are added. The script will always run as the app owner regardless of the account used to authorise the project. [Call Apps Script from an automation \- AppSheet Help](https://support.google.com/appsheet/answer/11997142) 

## **Step 6: Testing our Shared Drive Approval Flow**

**Objective:** Demonstrate how to safely test applications and role-based security using the AppSheet emulator.

**Dialogue:** "We've built a complex workflow complete with security filters and automated scripts. Before we deploy this to our end-users, we absolutely must test it. A golden rule of AppSheet development is: *never test in production with your primary admin account*. As the app creator, your admin account often bypasses standard security filters, giving you a false sense of what the end-user will actually see.

Thankfully, AppSheet has a brilliant feature for this. Down here in the emulator, we can change the email address the app simulates. I'm going to switch from my admin account to a test user: `martin.test@appsdemo.se`."

*(Instructor changes the emulator email and hits Apply)*

"Notice how the view instantly changes? Our security filters just kicked in. Now, acting exactly as Martin would, I'll navigate to our newly created 'Shared Drives' view. I'll tap the plus button to request a new drive. Because we set up those initial values and references earlier, notice that the 'Requester Email' is automatically filled with Martin's address, and the app instantly knows who Martin's manager is. Let's submit this test request."

**Action Steps in AppSheet:**

* **Change Emulator User:** In the AppSheet editor, look at the mobile emulator pane on the right side.  
* **Preview As:** Click into the "Preview app as" text field (usually displaying your admin email) and change it to `martin.test@appsdemo.se`.  
* **Apply:** Hit the "Apply" button or press Enter to refresh the emulator.  
* **Navigate:** In the emulated app, click on the **Shared Drives** view you created in Step 2\.  
* **Create Request:** Click the **Add (+)** floating action button to create a new Drive Request.  
* **Verify Automation:** Show the audience that `Requester Email` is automatically set to `martin.test@appsdemo.se` and that `Manager Email` has correctly populated via the Directory table reference.  
* **Editing:** Quickly show how we can navigate to the current view and remove unnecessary fields  
* Demonstrate: Make sure `martin.hawksey@appsdemo.se` is included in the form  
* **Submit:** Fill in dummy details for the Drive Name and click **Save**.  
* **AMP Email:** Show email with the approval button (click)

### **Key Concept \- Safe App Testing**

Testing your app using the "Preview app as" feature is the only way to accurately verify your `USEREMAIL()` expressions, Security Filters, and Initial Values. It guarantees that data visibility and automations function exactly as intended for different tiers of users (staff vs. managers) without needing to log in and out of different test accounts.

# **Part 3: Chromebook Management (Listing and Locking Devices)**

**Dialogue:** Hardware is our next stop. Managing a fleet of thousands of Chromebooks requires more than just a passive dashboard; it requires the ability to take immediate action. Now, for many of you, Google already provides powerful built-in tools like 'Class Tools' which allow teachers to lock or unlock devices during a lesson. If you have those licenses, that is absolutely your best path.

But today, we are looking under the hood. We're exploring the ChromeOS APIs to show you how to build custom security triggers directly into your own Toolbox. Whether it's a specialized 'Lost Device' protocol or giving delegated control to a specific lab tech, we’re highlighting the opportunity to extend the Admin Console's reach into the hands of those who need it most, when they need it.

## **Step 1: The Login Audit**

**Objective:** Create a real-time activity feed of ChromeOS login/logout events within the AppSheet app.

**Dialogue:** "A list of serial numbers is helpful, but in a crisis, like a missing device, context is king. Who was the last person to sign in? When did they sign out? We're going to use the Reports API to pull a live audit log of login events directly into our app. Instead of digging through Admin Console logs, we’ll have a chronological feed that tells us exactly who was using what, and when."

> **Gemini Prompt 6:** \> "Can you write a modern Google Apps Script that uses the AdminReports advanced service to fetch CHROME\_OS\_LOGIN\_EVENT and CHROME\_OS\_LOGOUT\_EVENT. It should check the date of the last entry in a sheet called 'LoginHistory' and insert only newer events at the top of the sheet (Row 2). The data I would like returned is a unique Event ID (set as a text), Event Timestamp, Event Type, User Email (of who logged in), Serial Number and Device ID"

Because the Reports API is a distinct endpoint from the Directory API, you need to add it to your Services list (Gemini should hopefully tell you this\!).

1. In the Apps Script editor, click on **Services** (the `+` icon).  
2. Select **Admin SDK API**.  
3. **Crucial Step:** In the "Identifier" box at the bottom of the popup, change it from `AdminDirectory` to **`AdminReports`**. (If you still need the Directory API for your previous script, simply add the Admin SDK API a second time so you have both identifiers active in your project).

### **Code Snippet**

```javascript
/**
 * Fetches newer Chrome OS login and logout events and inserts 
 * them at the top of the 'LoginHistory' sheet.
 */
function syncChromeOsLogins() {
  const sheetName = 'LoginHistory';
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(sheetName);
  
  // Headers matching your final requirements
  const headers = ['Unique Event ID', 'Event Timestamp', 'Event Type', 'User Email', 'Serial Number', 'Device ID'];
  
  // 1. Create the sheet and format headers if it doesn't exist
  if (!sheet) {
    sheet = ss.insertSheet(sheetName);
    sheet.appendRow(headers);
    sheet.getRange(1, 1, 1, headers.length).setFontWeight('bold');
    sheet.getRange(1, 1, 1, headers.length).createFilter(); 
  }
  
  // 2. Determine the time window to query
  let startTimeIso = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString(); 
  
  // Check Row 2, Column 2 (B2) since the Timestamp is the second column
  const lastEntryDate = sheet.getRange(2, 2).getValue(); 
  
  // If there is a valid timestamp, grab newer events only (+1 millisecond to prevent duplicates)
  if (lastEntryDate && lastEntryDate instanceof Date && !isNaN(lastEntryDate.getTime())) {
    startTimeIso = new Date(lastEntryDate.getTime() + 1).toISOString();
  }
  
  const allEvents = [];
  const eventNamesToFetch = ['CHROME_OS_LOGIN_EVENT', 'CHROME_OS_LOGOUT_EVENT'];
  
  // 3. Fetch data from the AdminReports API
  eventNamesToFetch.forEach(eventName => {
    let pageToken;
    do {
      try {
        const response = AdminReports.Activities.list('all', 'chrome', {
          eventName: eventName,
          startTime: startTimeIso,
          maxResults: 1000,
          pageToken: pageToken
        });
        
        if (response.items && response.items.length > 0) {
          allEvents.push(...response.items);
        }
        
        pageToken = response.nextPageToken;
      } catch (error) {
        console.error(`Error fetching ${eventName}:`, error.message);
        break; 
      }
    } while (pageToken);
  });
  
  // 4. Exit early if there's nothing new to add
  if (allEvents.length === 0) {
    console.log('No new Chrome OS events found.');
    return;
  }
  
  // 5. Sort everything chronologically (newest first)
  allEvents.sort((a, b) => new Date(b.id.time) - new Date(a.id.time));
  
  // 6. Map the JSON into clean sheet rows
  const newRows = allEvents.map(activity => {
    // We keep this as the raw string from the API
    const eventId = activity.id.uniqueQualifier || 'Unknown';
    const timestamp = new Date(activity.id.time);
    
    const event = activity.events?.[0];
    const eventType = event?.name || 'Unknown';
    const params = event?.parameters || [];
    
    // Helper to safely extract nested Chrome device parameters
    const getParam = (name) => {
      const p = params.find(p => p.name === name);
      return p ? (p.value || p.intValue || p.boolValue || '') : '';
    };
    
    // Extract data using the keys from your Workspace payload
    const userEmail = getParam('DEVICE_USER') || 'Unknown';
    const serialNumber = getParam('DEVICE_NAME') || 'Unknown';
    const deviceId = getParam('DIRECTORY_DEVICE_ID') || 'Unknown';
    
    return [eventId, timestamp, eventType, userEmail, serialNumber, deviceId];
  });
  
  // 7. Insert new rows safely at the top
  sheet.insertRowsAfter(1, newRows.length);
  
  // 8. IMPORTANT: Format Column A as Plain Text BEFORE pasting the data
  // This stops Sheets from converting long IDs into scientific notation or mangling the numbers
  sheet.getRange(2, 1, newRows.length, 1).setNumberFormat('@');
  
  // 9. Write the values to the sheet
  sheet.getRange(2, 1, newRows.length, headers.length).setValues(newRows);
  
  console.log(`Success! Prepended ${newRows.length} new events to the sheet.`);
}
```

## **Step 2: Integrating the Audit Data & Slices**

**Objective:** Connect the login logs to the AppSheet app and use a Slice to manage security and visibility.

**Dialogue:** "Now we bring this data into our app. We’ll add the `LoginHistory` table and set it to 'Updates Only'. To handle our security, we’re going to use a Virtual Column to pull in the full 'Reporting Path' from our directory. Then, we create a **Slice** called 'Reporting Logins'. This slice acts as a filter, ensuring a teacher or manager only sees device history if their email exists anywhere in the student's reporting chain. This is far more flexible than a simple 1-to-1 manager lookup."

**Action Steps in AppSheet:**

1. **Add Table:** Add `LoginHistory` and set updates to **Updates Only**.  
2. **Update:** User Email to a `REF` our Directory table 

### **Advanced Concept \- Watching Report API events** 

This has demonstrated how to pull event data periodically checking the Admin SDK Reports API (a process known as polling). Instead of asking we can configure the Reports API to actively push data from Google Workspace. An example of this is included in [Appendix \- Watch](?tab=t.ro5rgv183dfl)

## **Step 3: The "Remote Lock" Command**

**Objective:** Build the function that AppSheet will trigger to issue the `disable` command.

**Dialogue:** "Now for the 'Action' part of the Toolbox. We are creating a function that takes a `deviceId` and issues a remote command. Because we’ve synced the `deviceId` in our login logs, we can trigger this action directly from the history view. In the Admin Console, this is a multi-step process. In our AppSheet app, it will be a single button labeled 'Lock Device' giving the teacher or tech immediate, delegated control the moment they spot a suspicious login or a lost device report."

**Action:** Add the command function to your Apps Script file.

> **Gemini Prompt 7:** \> "Can you write a modern Google Apps Script function called issueDeviceCommand that accepts two string parameters: `deviceId` and `action`. It should use the AdminDirectory advanced service to perform the action to lock or unlock a ChromeOS device. Return a success or error message"

**Action Steps in Apps Script:**

1. Open your standalone Apps Script file (created earlier).  
2. Copy the `issueDeviceCommand` code snippet below into the editor.  
3. Before this script can run, you must enable the **Admin SDK API** Advanced Service:  
   * Click on **Services** (the `+` icon) on the left sidebar of the Apps Script editor.  
   * Scroll down and select **Admin SDK API**.  
   * Click **Add**.

### **Code Snippet**

```javascript
/**
 * Issues a command to lock (disable) or unlock (reenable) a ChromeOS device.
 * * @param {string} deviceId - The unique Google Workspace ID of the ChromeOS device.
 * @param {string} command - The command to issue ('lock' or 'unlock').
 * @returns {string} Success or error message.
 */
function issueDeviceCommand(deviceId, command) {
  if (!deviceId || !command) {
    return 'Error: Both deviceId and command parameters are required.';
  }

  const customerId = 'my_customer'; // 'my_customer' is a safe alias for your own Workspace domain
  const normalizedCommand = command.trim().toLowerCase();
  
  let actionStr;
  
  // Map friendly terms to exact Google API action terms
  if (normalizedCommand === 'lock' || normalizedCommand === 'disable') {
    actionStr = 'disable';
  } else if (normalizedCommand === 'unlock' || normalizedCommand === 'reenable') {
    actionStr = 'reenable';
  } else {
    return `Error: Invalid command "${command}". Please use "lock" or "unlock".`;
  }

  try {
    // Call the Admin SDK Directory API
    AdminDirectory.Chromeosdevices.action(
      { action: actionStr }, 
      customerId, 
      deviceId
    );
    
    // Create a user-friendly confirmation message
    const friendlyStatus = actionStr === 'disable' ? 'locked (disabled)' : 'unlocked (re-enabled)';
    return `Success: Device ${deviceId} has been successfully ${friendlyStatus}.`;
    
  } catch (error) {
    // Catch and return any API errors (e.g., invalid device ID, insufficient permissions)
    return `Error: Failed to ${normalizedCommand} device ${deviceId}. API Response: ${error.message}`;
  }
}
```

### **Key Concept \- Execution Context and Least Privilege**

When AppSheet calls an Apps Script, that script runs under the authority of the Google account that *authored and authorized* the script, regardless of who tapped the button in AppSheet. This is powerful because it allows a teacher to lock a device without needing Admin Console access. However, it requires care: you should ideally use a dedicated service account or custom admin role to authorize these scripts, adhering to the principle of "Least Privilege" rather than using your Super Admin account for everything.

## **Step 4: Adding the Action Buttons to the App**

**Objective:** Create conditional buttons that trigger the hardware states.

**Dialogue:** "Because we want a great user experience, we're going to create two distinct actions: one to 'Lock' and one to 'Unlock'. AppSheet will automatically hide the Lock button once a device is secured and show the Unlock button instead. When either is tapped, it sets our `Device Status` to a specific keyword—'lock' or 'unlock'—which our automation bot passes directly to the Google API. It's clean, intuitive, and extremely fast to build."

**Action Steps in AppSheet:**

1. **Add Data Column:** In the `LoginHistory` sheet, add a column `Device Status`. Regenerate the table in AppSheet. Update the column to a `Enum` and add the values `lock` and `unlock`, check ‘Allow other values  
2. **Create Action:** Next to LoginHistory click the `+`, AppSheet should hopefully suggest Add individual buttons to set state of…   
3. **Create Bot:**  
   * **Event:** Data Change (Updates only) on `LoginHistory` where `OR([Device Status] = "lock", [Device Status] = "unlock")`.  
   * **Step 1 (Call Script):** \* **Name:** "Device Command Script".  
     * **Function:** `issueDeviceCommand`.  
     * **Inputs:** `deviceId` \= `[Device ID]`, `action` \= `[Device Status]`.  
   * **Step 2 (Set Row Values):**  
     * **Set Column:** `Device Status` \= `[Device Command Script].[Output]`.

# **Part 4: Chromebook Management  (Gemini API Damage Reports)** 

**Dialogue:** "In Part 3, we successfully pulled live ChromeOS device logs into our Admin Toolbox app. But what happens when one of those devices is physically damaged? Typically, that means a vague helpdesk ticket reading 'my screen is broken'. Let's extend our current app to handle this automatically. We are going to add a feature where an IT tech can select a device directly from the ChromeOS logs, snap a photo of the damage, and have the Gemini API instantly analyse and triage the report.

## **Step 1: Generating the Data Structure with Gemini** 

**Objective:** Use the Gemini web app to generate the underlying Google Sheet structure and populate it with dummy data. 

**Dialogue:** "Before we write any code or build our AppSheet interface, we need a place to store our damage reports. Instead of manually typing out columns and making up dummy serial numbers to test our app, let's ask Gemini to do the heavy lifting. We can use a simple prompt to generate a table and instantly export it." 

> **Gemini Prompt 8: \>** "Can you create a table for an IT hardware 'Damage Reports' tracker. Include the following columns: Report ID (unique 6-digit alphanumeric), Device ID (standard Chromebook serial format), Image, Response Text,  Short Name, User Email and Report Timestamp. Please populate it with 5 rows of dummy data." 

**Action:**

1. Open the Gemini web app ([gemini.google.com](http://gemini.google.com)).  
2. Paste the prompt above.  
3. Once the table is generated, click the "copy" icon at the bottom right of the response.  
4. In your AppSheet data source create a new tab called ‘DamageReports’ and paste the data.

Alternatively you can create the table using the Gemini Sidepanel in Google Sheets if it is available to you.

## **Step 2: Connecting the Frontend (AppSheet Interface)**

**Objective:** Add the new table to the AppSheet app and create an inline action to trigger the damage report form from the ChromeOS logs. 

**Dialogue:** "Now that we have our data structure, let's bring it into our Admin Toolbox. We'll add the new table and create an inline action on our ChromeOS logs. This will allow our techs to tap a button, pull up a form, and snap a picture right from their phones." 

**Action:**

1. In the AppSheet editor, navigate to **Data \> Tables** and add the new 'Damage Reports' tab you just added to your Google Sheet.  
2. Ensure the `Image` column type is set to 'Image', `Report Timestamp` is set to 'DateTime' and has an initial value of `Now()` , and `User Email` is set as a '`Ref`' to the 'Directory' table (set the initial value to `USEREMAIL()` so it auto-selects the current user).  
3. Navigate to **Actions**. Create a new action on your existing `LoginHistory` table called 'Report Damage'.  
4. Set 'Do this' to 'App: go to another view within this app'.  
5. Use the expression `LINKTOFORM("DamageReports_Form", "Device ID", [Device ID], "User Email", [User Email])`. This will open the form and auto-fill the device serial number.

## **Step 3: The AI Backend (Apps Script)**

**Objective:** Set up the Google Apps Script environment and the Gemini API bridge. 

**Dialogue:** "Our app can take photos, but now we need it to think. We are going to write a Google Apps Script using the '[GeminiApp](https://github.com/mhawksey/GeminiApp)' library to handle the API calls. We'll set up a script that takes the image saved to Google Drive, pairs it with a prompt, and asks the Gemini 2.5 Flash model to act as an expert IT technician to analyze the damage, for example, spotting a broken hinge and knowing it's a high severity risk because it might pinch the display cable." 

**Action:**

1. Navigate to your existing AppSheet automation script or use script.new to create a new Apps Script project .  
2. Add the `GeminiApp.js` library to your project.  
3. Add the following image processing code to your main `Code.gs` file.  
4. Add a Script Property with the name `'GEMINI_API_KEY'` with a Gemini API key from AI Studio ([ai.dev](https://ai.dev)) 

This example is using an AI Studio key. For other authentication methods see [Appendix \- Gemini API Authentication](?tab=t.szxh3lyuh0xi)

### **Code Snippet (Code.gs):**

```javascript
// See https://github.com/mhawksey/geminiapp for config options
const genAI = new GeminiApp(PropertiesService.getScriptProperties().getProperty('GEMINI_API_KEY'));

function runTextOnly() {

  // For text-only input, use the gemini-pro model
  const model = genAI.getGenerativeModel({ model: "gemini-flash-latest" });

  const prompt = "Write a story about a magic backpack."

  const result = model.generateContent(prompt);
  const response = result.response;
  const text = response.text();
  console.log(text);
}


/**
 * Processes an image with a given prompt using the Gemini API.
 *
 * @param {string} [prompt=""] - The prompt to use for image processing.
 * @param {string} imagePath - The image filename, of the image on Google Drive.
 * @param {number} temp - The temperature parameter for the Gemini model.
 * @returns {object} - A JSON object containing the response text and a short image description.
 */
function processImage(prompt = "", imagePath = "DamageReports_Images/ba2c35fe.Image.200133.jpg", temp = 1) {
  try {
    // Construct the detailed prompt for the AI model
    prompt = `${prompt}
              You are an expert IT hardware technician. Analyze this image of a damaged Chromebook or laptop.`;

    // Define the JSON schema to strictly control the AI's output format
    const schema = {
      "type": "object",
      "properties": {
        "damageSummary": {
          "type": "string",
          "description": "A short, 5-10 word title describing the damage (e.g., Detached Left Hinge)"
        },
        "primaryPart": {
          "type": "string",
          "description": "The main component damaged (e.g., Hinge, Screen, Keyboard, Chassis)"
        },
        "severity": {
          "type": "string",
          "description": "Low, Medium, High based on usability and risk of further damage"
        },
        "detailedAnalysis": {
          "type": "string",
          "description": "A detailed explanation of the visible damage, potential unseen secondary damage (like cable pinch from a broken hinge), and recommended next steps."
        }
      },
      "required": ["damageSummary", "primaryPart", "severity", "detailedAnalysis"]
    };

    // Configure the AI model's behavior, enforcing the JSON schema
    const generationConfig = {
      'maxOutputTokens': 2048,     // Maximum length of the AI's response
      'temperature': temp,         // Controls the creativity of the response (higher = more creative)
      'responseMimeType': 'application/json', // Expect JSON output
      'responseSchema': schema     // Apply the defined structure schema
    }

    // Get the specific generative model you want to use
    const model = genAI.getGenerativeModel({ model: "gemini-2.5-flash", generationConfig });

    // Get the ID of the image file you want to process
    const fileId = getFirstFileByName_(imagePath);

    // Prepare the image file for the AI model
    const fileParts = [
      fileToGenerativePart_(fileId),
    ];

    // Send the prompt and the image to the AI model and get the result
    const result = model.generateContent([prompt, ...fileParts]);
    const responseText = result.response.text();

    console.log(responseText);

    // Parse the AI's JSON response
    return JSON.parse(responseText);

  } catch (e) {
    // Handle any errors that occur during the process
    return { responseText: e.message, shortName: 'Oops something went wrong' }
  }
}


/**
 * Gets the ID of the first file with a matching name in Google Drive. 
 * Note: This function is likely intended for internal use due to the leading underscore.
 *
 * @param {string} filename - The name of the file to search for.
 * @returns {string|null} The ID of the first matching file, or null if no file is found.
 */
function getFirstFileByName_(filePath) {
  const parts = filePath.split('/');
  const filename = parts.pop();
  const pages = Drive.Files.list({
    q: `name contains '${filename}'`,
    corpora: "allDrives",
    includeItemsFromAllDrives: true,
    supportsAllDrives: true,
    fields: 'files(id)',
    pageSize: 1
  })
  if (pages.files) {
    return pages.files[0].id; // Return the first matching file 
  } else {
    return null; // No file found
  }
}

/**
 * Converts a Drive file into a format suitable for use with a generative AI model.
 * 
 * @param {string} id - The ID of the file in Google Drive.
 * @returns {Object} An object containing the file's data encoded in Base64 and its MIME type, formatted for use with a generative AI model.
 */
function fileToGenerativePart_(id) {
  const file = DriveApp.getFileById(id);
  const imageBlob = file.getBlob();
  const base64EncodedImage = Utilities.base64Encode(imageBlob.getBytes())
  return {
    inlineData: {
      data: base64EncodedImage,
      mimeType: file.getMimeType()
    },
  };
}
```

## **Step 4: Automating the Hand-off (AppSheet Bots)**

**Objective:** Trigger the AI script automatically from AppSheet when a new report is saved. 

**Dialogue:** "The final piece is the hand-off. We need AppSheet to tell our script to run the moment a tech saves a damage report form, automatically passing the photo over for analysis." 

**Action:**

1. Navigate to **Automation \> Bots** in the AppSheet editor. Create a new Bot that triggers when a new record is added ('Adds\_Only') to the 'Damage Reports' table.  
2. Add a Process step and use the name ‘processImage’  
3. Under the Task Settings for this Bot, select 'Run a task' \> 'Call a script'   
4. Click the 'Choose file' button, select the Apps Script project you just created, and click Authorize.  
5. From the 'Choose a function' dropdown, select `processImage`.  
6. Pass the `[Image]` column as the `imagePath` argument, and click Save.  
7. Add a second step to the Bot called `Update Report`.  
8. Set the task to 'Run a data action' \> 'Set row values'.  
9. Map the columns to the script output we defined in our JSON structure:  
   1. `Primary Part` \= `[processImage].[primaryPart]`  
   2. `Severity` \= `[processImage].[severity]`  
   3. `Damage Summary` \= `[processImage].[damageSummary]`  
   4. `Detailed Analysis` \= `[processImage].[detailedAnalysis]`  
10. Click Save.

### **Key Concept \- Structured Output with Gemini (JSON)**

The secret to making this production-ready isn't just the AI; it's how we enforce the data structure. Look closely at our script's `generationConfig`. Instead of hoping the AI follows text instructions to format its output, we use **JSON Controlled Generation**. By providing a strict `responseSchema` alongside the `responseMimeType: 'application/json'` setting, this ensures predictable results and simplifies extracting structured data from unstructured text. [Structured output | Generative AI on Vertex AI | Google Cloud Documentation](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/multimodal/control-generated-output) 
