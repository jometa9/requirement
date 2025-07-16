# COPIER API - Expert Advisor Integration Guide

> **Complete documentation for MetaTrader Expert Advisors**
> Version: 1.0.0
> Date: 2024

## 📋 Table of Contents

1. [Initial Setup](#initial-setup)
2. [Authentication](#authentication)
3. [Main Endpoints](#main-endpoints)
4. [Workflow](#workflow)
5. [Code Examples](#code-examples)
6. [Error Codes](#error-codes)
7. [Response Messages](#response-messages)

---

## 🔧 Initial Setup

### Base URL
```
http://localhost:80/api
```

### Required Headers
```http
Content-Type: application/json
x-account-id: {accountId}
x-api-key: COPIER_APIKEY
```

> **🔑 Fixed API Key**: All MT4/MT5 accounts must use the same fixed API key `COPIER_APIKEY` to authenticate their requests. This key validates that orders come from authorized sources, while `x-account-id` identifies the specific account.

---

## 🔐 Authentication

### 1. Check Account Type
**Endpoint:** `GET /orders/account-type`
**Description:** First endpoint that EA must call to identify if it's master, slave, or pending.

```http
GET /api/orders/account-type
x-account-id: {accountId}
x-api-key: COPIER_APIKEY
```

**Responses:**

#### Pending Account (New):
```json
{
  "accountId": "123456",
  "type": "pending"
}
```

#### Configured Account (Master/Slave):
```json
{
  "accountId": "123456",
  "type": "master" // or "slave"
}
```

---

## 📡 Main Endpoints

### For MASTER Accounts

#### 1. Send New Order
**Endpoint:** `POST /orders/neworder`
**Description:** Sends a new trading order that will be copied to connected slave accounts.

```http
POST /api/orders/neworder
x-account-id: {masterAccountId}
x-api-key: COPIER_APIKEY
Content-Type: application/x-www-form-urlencoded

counter=1&id0=12345&sym0=EURUSD&typ0=buy&lot0=0.1&price0=1.12345&sl0=1.12000&tp0=1.13000&account0={masterAccountId}
```

**Response:**
```
OK
```
*Note: Response is plain text "OK" when order is processed successfully.*

#### 2. Get Trading Configuration
**Endpoint:** `GET /trading-config/{masterAccountId}`

```http
GET /api/trading-config/master_123
x-account-id: master_123
x-api-key: COPIER_APIKEY
```

### For SLAVE Accounts

#### 1. Receive Pending Orders
**Endpoint:** `GET /orders/neworder`
**Description:** Gets orders that should be copied from its master account.

```http
GET /api/orders/neworder
x-account-id: {slaveAccountId}
x-api-key: COPIER_APIKEY
```

**Response (when master is online and has orders):**
```
[1]
[12345,EURUSD,buy,0.1,1.12345,1.12000,1.13000,1704110400,master_123]
```
*Note: Response is CSV format where each line represents an order. First line is counter, following lines are: [orderId,symbol,type,lot,price,sl,tp,timestamp,account]*

**Response (when master is offline or no orders):**

**Master Offline/Not Registered:**
```
0
```

**Master Online but No Orders:**
```
[3]
```
*Note: Returns "0" (special response) when:*
- *Master account is offline*
- *Slave account is not registered*
- *Master account is not registered*
- *Copier is disabled for master*

*Note: Returns "[counter]" (only counter) when:*
- *Master is online but has no pending orders*
- *All orders are filtered out by slave configuration*

**Important:** 
- The "0" response is **NOT** the counter from the CSV format. It's a special server response indicating "no orders available".
- The "[counter]" response means "master is online but no new orders available".
- The EA should **NOT** close existing positions when receiving either response - it only means no new orders are available.

#### 2. Process Received Orders
**Description:** No need to confirm to server - slave simply executes received orders in MT4/MT5 platform.

### For All Accounts

#### 1. Ping/Keep Alive
**Endpoint:** `POST /accounts/ping`
**Description:** Keeps connection active and reports EA status.

```http
POST /api/accounts/ping
x-account-id: {accountId}
x-api-key: COPIER_APIKEY
Content-Type: application/json

{
  "status": "online",
  "lastActivity": "2024-01-01T12:00:00Z"
}
```

**Response:**
```json
{
  "message": "Ping successful",
  "accountId": "123456",
  "accountType": "master",
  "timestamp": "2024-01-01T12:00:00Z",
  "status": "active"
}
```

#### 2. Check Server Status
**Endpoint:** `GET /status`

```http
GET /api/status
```

---

## 🔄 Workflow

### For MASTER EA:
1. **Initialization:**
   - Call `GET /orders/account-type` to verify type
   - If "pending", wait for admin configuration
   - If "master", continue with workflow

2. **Active Trading:**
   - When opening position → `POST /orders/neworder`
   - Every 30 seconds → `POST /accounts/ping`

### For SLAVE EA:
1. **Initialization:**
   - Call `GET /orders/account-type` to verify type
   - If "pending", wait for admin configuration
   - If "slave", continue with workflow

2. **Copy Trades:**
   - Every 5 seconds → `GET /orders/neworder`
   - For each received order → Execute in MT4/MT5
   - Every 30 seconds → `POST /accounts/ping`

---

## 💻 Code Examples (MQL5)

### Function to Check Account Type
```mql5
bool CheckAccountType()
{
   string url = API_BASE_URL + "/orders/account-type";
   string headers = "x-account-id: " + AccountInfoInteger(ACCOUNT_LOGIN) + "\r\n";
   headers += "x-api-key: " + API_KEY + "\r\n";

   char post[], result[];
   string result_string;

   int res = WebRequest("GET", url, headers, 5000, post, result, result_string);

   if(res == 200)
   {
      // Parse JSON response
      // Determine if master, slave or pending
      return true;
   }

   return false;
}
```

### Function to Send Order (Master)
```mql5
bool SendOrderToAPI(string symbol, double volume, int type, double price, double sl, double tp, long orderId)
{
   string url = API_BASE_URL + "/orders/neworder";
   string headers = "x-account-id: " + AccountInfoInteger(ACCOUNT_LOGIN) + "\r\n";
   headers += "x-api-key: " + API_KEY + "\r\n";
   headers += "Content-Type: application/x-www-form-urlencoded\r\n";

   string postData = StringFormat(
      "counter=1&id0=%d&sym0=%s&typ0=%s&lot0=%.2f&price0=%.5f&sl0=%.5f&tp0=%.5f&account0=%d",
      orderId, symbol, (type == ORDER_TYPE_BUY) ? "buy" : "sell", volume, price, sl, tp, AccountInfoInteger(ACCOUNT_LOGIN)
   );

   char post[], result[];
   string result_string;

   StringToCharArray(postData, post, 0, StringLen(postData));

   int res = WebRequest("POST", url, headers, 5000, post, result, result_string);

   return (res == 200 && result_string == "OK");
}
```

### Function to Receive Orders (Slave)
```mql5
bool GetPendingOrders()
{
   string url = API_BASE_URL + "/orders/neworder";
   string headers = "x-account-id: " + AccountInfoInteger(ACCOUNT_LOGIN) + "\r\n";
   headers += "x-api-key: " + API_KEY + "\r\n";

   char post[], result[];
   string result_string;

   int res = WebRequest("GET", url, headers, 5000, post, result, result_string);

   if(res == 200)
   {
      // Check for empty response (master offline or not registered)
      if(result_string == "0")
      {
         UpdateChartMessage("⏸️ COPIER STATUS: NO ORDERS\\nMaster offline or not registered\\nExisting positions maintained");
         return true; // Success but no orders - DO NOT close existing positions
      }
      
      // Parse CSV format
      // First line is counter [1]
      // Following lines are orders [id,symbol,type,lot,price,sl,tp,timestamp,account]
      string lines[];
      int lineCount = StringSplit(result_string, '\n', lines);
      
      if(lineCount > 1) // Has orders (more than just counter)
      {
         UpdateChartMessage("📈 COPIER STATUS: ORDERS RECEIVED\\nProcessing new trades from master");
         
         for(int i = 1; i < lineCount; i++) // Skip counter line
         {
            if(StringLen(lines[i]) > 0 && StringFind(lines[i], "[") >= 0)
            {
               string cleanLine = StringSubstr(lines[i], 1, StringLen(lines[i]) - 2); // Remove []
               string fields[];
               int fieldCount = StringSplit(cleanLine, ',', fields);
               
               if(fieldCount >= 7)
               {
                  // Execute order: fields[0]=id, fields[1]=symbol, fields[2]=type, etc.
                  ExecuteSlaveOrder(fields);
               }
            }
         }
      }
      else if(lineCount == 1 && StringFind(lines[0], "[") >= 0)
      {
         // Only counter received (master online but no orders)
         UpdateChartMessage("⏸️ COPIER STATUS: NO ORDERS\\nMaster online but no pending orders\\nExisting positions maintained");
      }
      else
      {
         UpdateChartMessage("⏸️ COPIER STATUS: NO ORDERS\\nNo pending orders from master\\nExisting positions maintained");
      }
      return true;
   }

   return false;
}

// Function to execute slave order (example)
void ExecuteSlaveOrder(string &fields[])
{
   // fields[0] = orderId
   // fields[1] = symbol
   // fields[2] = type (buy/sell)
   // fields[3] = lot
   // fields[4] = price
   // fields[5] = stopLoss
   // fields[6] = takeProfit
   // fields[7] = timestamp
   // fields[8] = masterAccount
   
   // Execute the order in MT4/MT5
   // This is just an example - implement according to your EA logic
   Print("Executing order: ", fields[1], " ", fields[2], " ", fields[3], " lots");
}
```

### Function to Ping Server
```mql5
bool PingServer()
{
   string url = API_BASE_URL + "/accounts/ping";
   string headers = "x-account-id: " + AccountInfoInteger(ACCOUNT_LOGIN) + "\r\n";
   headers += "x-api-key: " + API_KEY + "\r\n";
   headers += "Content-Type: application/json\r\n";

   string postData = StringFormat(
      "{\"status\":\"online\",\"lastActivity\":\"%s\"}",
      TimeToString(TimeCurrent(), TIME_DATE|TIME_SECONDS)
   );

   char post[], result[];
   string result_string;

   StringToCharArray(postData, post, 0, StringLen(postData));

   int res = WebRequest("POST", url, headers, 5000, post, result, result_string);

   return (res == 200);
}
```

---

## ⚠️ Error Codes

| Code | Description | Specific Error Messages | Action |
|------|-------------|------------------------|--------|
| 200 | OK | - | Continue |
| 400 | Bad Request | - `"masterAccountId is required"`<br>- `"slaveAccountId is required"`<br>- `"Both slaveAccountId and masterAccountId are required"`<br>- `"Account ID is required"`<br>- `"No data received"`<br>- `"Platform {platform} is not supported"` | Check request parameters |
| 401 | Unauthorized | - `"Account ID is required"`<br>- `"Invalid or missing API key"`<br>- `"API Key required - use requireValidSubscription middleware"`<br>- `"Authentication required"` | Check account ID and API key |
| 403 | Forbidden | - `"Account pending configuration"`<br>- `"Access denied - This endpoint is only available for master accounts"`<br>- `"Access denied - This endpoint is only available for slave accounts"`<br>- `"POST requests (sending trades) are only allowed for master accounts"`<br>- `"GET requests (receiving trades) are only allowed for slave accounts"`<br>- `"Account mismatch"`<br>- `"Only master accounts can create orders"`<br>- `"Only slave accounts can retrieve orders"`<br>- `"Master account {id} is not registered"`<br>- `"Slave account {id} is not registered"` | Check account permissions and type |
| 404 | Not Found | - `"Master account {id} not found"`<br>- `"Slave account {id} not found"`<br>- `"Pending account {id} not found"` | Check account registration |
| 409 | Conflict | - `"Master account {id} is already registered"`<br>- `"Slave account {id} is already registered"`<br>- `"Account {id} already exists as master or slave"` | Account already exists |
| 500 | Server Error | - `"Failed to register master account"`<br>- `"Failed to register slave account"`<br>- `"Failed to connect accounts"`<br>- `"Failed to save account configuration"`<br>- `"Error writing CSV for account {id}"`<br>- `"Error reading CSV for master account {id}"`<br>- `"Internal server error"` | Retry after 30s |

---

## 📝 Response Messages

### Success Messages

#### Account Type Check (Pending):
```json
{
  "accountId": "123456",
  "type": "pending"
}
```

#### Account Type Check (Master):
```json
{
  "accountId": "123456",
  "type": "master"
}
```

#### Account Type Check (Slave):
```json
{
  "accountId": "123456",
  "type": "slave"
}
```

#### Ping Response:
```json
{
  "message": "Ping successful",
  "accountId": "123456",
  "accountType": "master",
  "timestamp": "2024-01-01T12:00:00Z",
  "status": "active"
}
```

### Error Messages

#### 401 - Unauthorized:
```json
{
  "error": "Account ID is required",
  "message": "Please provide accountId in headers (x-account-id), query params, or request body"
}
```

```json
{
  "error": "Invalid or missing API key",
  "message": "Please provide the correct API key in x-api-key header"
}
```

#### 403 - Forbidden (Pending Account):
```json
{
  "error": "Account pending configuration",
  "message": "This account is pending configuration. Please contact administrator to set up as master or slave.",
  "accountType": "pending",
  "status": "awaiting_configuration",
  "nextSteps": [
    "Contact administrator",
    "Account will be configured as master or slave",
    "Then EA can begin trading operations"
  ]
}
```

#### 403 - Forbidden (Wrong Account Type):
```json
{
  "error": "Access denied",
  "message": "POST requests (sending trades) are only allowed for master accounts",
  "accountType": "slave",
  "allowedMethods": ["GET"]
}
```

```json
{
  "error": "Access denied",
  "message": "GET requests (receiving trades) are only allowed for slave accounts",
  "accountType": "master",
  "allowedMethods": ["POST"]
}
```

#### 403 - Forbidden (Account Mismatch):
```json
{
  "error": "Account mismatch. Authenticated as 123456 but trying to create order for 789012"
}
```

#### 403 - Forbidden (Not Registered):
```json
{
  "error": "Master account 123456 is not registered. Please register the account first."
}
```

```json
{
  "error": "Slave account 123456 is not registered"
}
```

#### 400 - Bad Request:
```json
{
  "error": "masterAccountId is required"
}
```

```json
{
  "error": "slaveAccountId is required"
}
```

```json
{
  "error": "No data received"
}
```

#### 409 - Conflict:
```json
{
  "error": "Master account 123456 is already registered"
}
```

```json
{
  "error": "Slave account 123456 is already registered"
}
```

#### 500 - Server Error:
```json
{
  "error": "Failed to register master account"
}
```

```json
{
  "error": "Error writing CSV for account 123456"
}
```

```json
{
  "error": "Error reading CSV for master account 123456"
}
```

---

## 🔧 Advanced Configuration

### Fixed Variables (Defined by System)

These variables are **FIXED** and should **NOT** be changed by users:

```mql5
// FIXED VARIABLES - DO NOT MODIFY
#define API_BASE_URL "http://localhost:80/api"
#define API_KEY "COPIER_APIKEY"
#define PING_INTERVAL 50        // seconds - fixed by system
#define POLL_INTERVAL 5         // seconds - fixed by system (slaves only)
#define MAX_RETRIES 3           // fixed by system
#define STATUS_CHECK_INTERVAL 60 // seconds - fixed by system (pending accounts)
#define CONNECTION_TIMEOUT 5000  // milliseconds - fixed by system
#define RETRY_DELAY 30          // seconds - fixed by system
```

### System-Defined Intervals

The following intervals are **automatically managed** by the system:

| Interval | Value | Purpose | Account Type |
|----------|-------|---------|--------------|
| **PING_INTERVAL** | 30 seconds | Keep-alive ping | All accounts |
| **POLL_INTERVAL** | 5 seconds | Check for new orders | Slave accounts only |
| **STATUS_CHECK_INTERVAL** | 60 seconds | Check account status | Pending accounts only |
| **CONNECTION_TIMEOUT** | 5 seconds | HTTP request timeout | All accounts |
| **RETRY_DELAY** | 30 seconds | Wait before retry | All accounts (on error) |
| **MAX_RETRIES** | 3 attempts | Maximum retry attempts | All accounts |

### User-Configurable Variables (Optional)

These variables can be modified by users if needed:

```mql5
// USER-CONFIGURABLE VARIABLES (Optional)
input bool ENABLE_DEBUG_MODE = false;     // Enable detailed logging
input bool SHOW_CHART_MESSAGES = true;    // Show status on chart
input string CUSTOM_SERVER_URL = "";      // Override server URL (advanced users only)
```

### Implementation Example

```mql5
// Global variables (system-managed)
string g_lastStatus = "";
string g_lastError = "";
datetime g_lastUpdate = 0;
int g_retryCount = 0;
bool g_isConnected = false;

// Timer variables (system-managed)
datetime g_lastPing = 0;
datetime g_lastPoll = 0;
datetime g_lastStatusCheck = 0;

// Function to check if it's time to ping
bool ShouldPing()
{
   return (TimeCurrent() - g_lastPing) >= PING_INTERVAL;
}

// Function to check if it's time to poll (slaves only)
bool ShouldPoll()
{
   return (TimeCurrent() - g_lastPoll) >= POLL_INTERVAL;
}

// Function to check if it's time to check status (pending only)
bool ShouldCheckStatus()
{
   return (TimeCurrent() - g_lastStatusCheck) >= STATUS_CHECK_INTERVAL;
}

// Function to handle retry logic
void HandleRetry(string operation)
{
   if(g_retryCount < MAX_RETRIES)
   {
      g_retryCount++;
      if(ENABLE_DEBUG_MODE)
         Print("Retrying ", operation, " (attempt ", g_retryCount, "/", MAX_RETRIES, ")");
      
      // Wait before retry
      Sleep(RETRY_DELAY * 1000);
   }
   else
   {
      g_retryCount = 0;
      UpdateChartMessage("", "MAX RETRIES REACHED\nCheck connection and settings");
   }
}
```

### Automatic Interval Management

The EA automatically manages these intervals based on account type:

#### For Master Accounts:
- **Ping every 30 seconds** → Keep connection alive
- **Send orders immediately** → When trades are opened
- **Status check every 60 seconds** → Verify account status

#### For Slave Accounts:
- **Ping every 30 seconds** → Keep connection alive
- **Poll every 5 seconds** → Check for new orders from master
- **Status check every 60 seconds** → Verify account status

#### For Pending Accounts:
- **Ping every 30 seconds** → Keep connection alive
- **Status check every 60 seconds** → Check if admin has configured account
- **No polling** → Cannot receive orders until configured

### Error Handling with Fixed Intervals

```mql5
// Function to handle connection errors with fixed retry logic
void HandleConnectionError(string error)
{
   g_isConnected = false;
   g_retryCount++;
   
   if(g_retryCount <= MAX_RETRIES)
   {
      UpdateChartMessage("", "CONNECTION ERROR\nRetrying in " + IntegerToString(RETRY_DELAY) + " seconds");
      Sleep(RETRY_DELAY * 1000);
   }
   else
   {
      UpdateChartMessage("", "CONNECTION FAILED\nMax retries reached\nCheck settings");
      g_retryCount = 0;
   }
}
```

### Connectivity Test:
```mql5
bool TestAPIConnection()
{
   string url = API_BASE_URL + "/status";
   char post[], result[];
   string result_string;

   int res = WebRequest("GET", url, "", 5000, post, result, result_string);
   return (res == 200);
}
```

---

## 📝 Important Notes

1. **Authentication:** Use MT4/MT5 account number as `x-account-id` header and fixed API key `COPIER_APIKEY`.
2. **Fixed API Key:** All accounts use the same API key `COPIER_APIKEY` to authenticate requests.
3. **Data Format:** Master sends data in form-urlencoded format, slave receives CSV.
4. **Multiple Orders:** A master can send multiple orders using indices (id0, id1, etc.).
5. **Polling:** Slave accounts should poll every 5 seconds maximum.
6. **Keep Alive:** All accounts should ping every 30 seconds.
7. **Error Handling:** Implement automatic retries for 500 errors.
8. **Logs:** Log all API calls for debugging.

---

## 📊 Chart Messages & User Feedback

### Display Messages on Chart

The EA should display status messages on the chart to keep users informed. Use `Comment()` function to show real-time status.

### Status Messages by Account Type

#### For Pending Accounts:
```
🔄 COPIER STATUS: PENDING CONFIGURATION
Account: 123456
Status: Awaiting administrator setup
Action: Contact administrator to configure as master/slave
Last Check: 2024-01-01 12:00:00
```

#### For Master Accounts:
```
✅ COPIER STATUS: MASTER ACTIVE
Account: 123456
Status: Sending trades to slaves
Connected Slaves: 3
Last Order: 2024-01-01 12:00:00
Last Ping: 2024-01-01 12:00:00
```

#### For Slave Accounts:
```
📥 COPIER STATUS: SLAVE ACTIVE
Account: 123456
Status: Receiving trades from master
Master: MASTER001
Last Order: 2024-01-01 12:00:00
Last Ping: 2024-01-01 12:00:00
```

### Error Messages on Chart

#### Connection Errors:
```
❌ COPIER ERROR: CONNECTION FAILED
Account: 123456
Error: Cannot connect to server
Action: Check internet connection
Retry: 30 seconds
```

#### Authentication Errors:
```
❌ COPIER ERROR: AUTHENTICATION FAILED
Account: 123456
Error: Invalid API key or account ID
Action: Check EA settings
Status: Disabled
```

#### Permission Errors:
```
❌ COPIER ERROR: ACCESS DENIED
Account: 123456
Error: Account not configured for this operation
Action: Contact administrator
Status: Disabled
```

#### Server Errors:
```
⚠️ COPIER WARNING: SERVER ERROR
Account: 123456
Error: Internal server error
Action: Retrying in 30 seconds
Status: Retry mode
```

### Implementation Example

```mql5
// Global variables for status tracking
string g_lastStatus = "";
string g_lastError = "";
datetime g_lastUpdate = 0;
int g_retryCount = 0;

// Function to update chart message
void UpdateChartMessage(string status, string error = "")
{
   string message = "";
   
   if(error != "")
   {
      message = "❌ COPIER ERROR: " + error + "\n";
      message += "Account: " + IntegerToString(AccountInfoInteger(ACCOUNT_LOGIN)) + "\n";
      message += "Time: " + TimeToString(TimeCurrent()) + "\n";
      message += "Action: Check connection and settings";
   }
   else
   {
      message = status + "\n";
      message += "Account: " + IntegerToString(AccountInfoInteger(ACCOUNT_LOGIN)) + "\n";
      message += "Last Update: " + TimeToString(TimeCurrent());
   }
   
   Comment(message);
   g_lastStatus = status;
   g_lastError = error;
   g_lastUpdate = TimeCurrent();
}

// Function to handle API responses
void HandleAPIResponse(int responseCode, string response)
{
   switch(responseCode)
   {
      case 200:
         if(StringFind(response, "pending") >= 0)
         {
            UpdateChartMessage("🔄 COPIER STATUS: PENDING CONFIGURATION\nContact administrator to setup account");
         }
         else if(StringFind(response, "master") >= 0)
         {
            UpdateChartMessage("✅ COPIER STATUS: MASTER ACTIVE\nSending trades to slaves");
         }
         else if(StringFind(response, "slave") >= 0)
         {
            UpdateChartMessage("📥 COPIER STATUS: SLAVE ACTIVE\nReceiving trades from master");
         }
         break;
         
      case 400:
         if(StringFind(response, "masterAccountId is required") >= 0)
         {
            UpdateChartMessage("", "BAD REQUEST\nMaster account ID is required");
         }
         else if(StringFind(response, "slaveAccountId is required") >= 0)
         {
            UpdateChartMessage("", "BAD REQUEST\nSlave account ID is required");
         }
         else if(StringFind(response, "Both slaveAccountId and masterAccountId are required") >= 0)
         {
            UpdateChartMessage("", "BAD REQUEST\nBoth slave and master account IDs are required");
         }
         else if(StringFind(response, "Account ID is required") >= 0)
         {
            UpdateChartMessage("", "BAD REQUEST\nAccount ID is required in headers");
         }
         else if(StringFind(response, "No data received") >= 0)
         {
            UpdateChartMessage("", "BAD REQUEST\nNo order data received");
         }
         else if(StringFind(response, "Platform") >= 0 && StringFind(response, "not supported") >= 0)
         {
            UpdateChartMessage("", "BAD REQUEST\nPlatform not supported");
         }
         else
         {
            UpdateChartMessage("", "BAD REQUEST\nCheck request parameters");
         }
         break;
         
      case 401:
         if(StringFind(response, "Account ID is required") >= 0)
         {
            UpdateChartMessage("", "AUTHENTICATION FAILED\nAccount ID is required in headers");
         }
         else if(StringFind(response, "Invalid or missing API key") >= 0)
         {
            UpdateChartMessage("", "AUTHENTICATION FAILED\nInvalid or missing API key");
         }
         else if(StringFind(response, "API Key required") >= 0)
         {
            UpdateChartMessage("", "AUTHENTICATION FAILED\nAPI key required for subscription");
         }
         else if(StringFind(response, "Authentication required") >= 0)
         {
            UpdateChartMessage("", "AUTHENTICATION FAILED\nAuthentication required");
         }
         else
         {
            UpdateChartMessage("", "AUTHENTICATION FAILED\nCheck account ID and API key");
         }
         break;
         
      case 403:
         if(StringFind(response, "Account pending configuration") >= 0)
         {
            UpdateChartMessage("", "ACCESS DENIED\nAccount pending configuration\nContact administrator");
         }
         else if(StringFind(response, "only available for master accounts") >= 0)
         {
            UpdateChartMessage("", "ACCESS DENIED\nThis endpoint is only for master accounts");
         }
         else if(StringFind(response, "only available for slave accounts") >= 0)
         {
            UpdateChartMessage("", "ACCESS DENIED\nThis endpoint is only for slave accounts");
         }
         else if(StringFind(response, "POST requests (sending trades) are only allowed for master accounts") >= 0)
         {
            UpdateChartMessage("", "ACCESS DENIED\nOnly master accounts can send trades");
         }
         else if(StringFind(response, "GET requests (receiving trades) are only allowed for slave accounts") >= 0)
         {
            UpdateChartMessage("", "ACCESS DENIED\nOnly slave accounts can receive trades");
         }
         else if(StringFind(response, "Account mismatch") >= 0)
         {
            UpdateChartMessage("", "ACCESS DENIED\nAccount mismatch\nCheck account configuration");
         }
         else if(StringFind(response, "Only master accounts can create orders") >= 0)
         {
            UpdateChartMessage("", "ACCESS DENIED\nOnly master accounts can create orders");
         }
         else if(StringFind(response, "Only slave accounts can retrieve orders") >= 0)
         {
            UpdateChartMessage("", "ACCESS DENIED\nOnly slave accounts can retrieve orders");
         }
         else if(StringFind(response, "is not registered") >= 0)
         {
            UpdateChartMessage("", "ACCESS DENIED\nAccount not registered\nContact administrator");
         }
         else
         {
            UpdateChartMessage("", "ACCESS DENIED\nAccount not configured for this operation");
         }
         break;
         
      case 404:
         if(StringFind(response, "Master account") >= 0 && StringFind(response, "not found") >= 0)
         {
            UpdateChartMessage("", "NOT FOUND\nMaster account not found");
         }
         else if(StringFind(response, "Slave account") >= 0 && StringFind(response, "not found") >= 0)
         {
            UpdateChartMessage("", "NOT FOUND\nSlave account not found");
         }
         else if(StringFind(response, "Pending account") >= 0 && StringFind(response, "not found") >= 0)
         {
            UpdateChartMessage("", "NOT FOUND\nPending account not found");
         }
         else
         {
            UpdateChartMessage("", "NOT FOUND\nAccount not registered");
         }
         break;
         
      case 409:
         if(StringFind(response, "Master account") >= 0 && StringFind(response, "already registered") >= 0)
         {
            UpdateChartMessage("", "CONFLICT\nMaster account already registered");
         }
         else if(StringFind(response, "Slave account") >= 0 && StringFind(response, "already registered") >= 0)
         {
            UpdateChartMessage("", "CONFLICT\nSlave account already registered");
         }
         else if(StringFind(response, "already exists as master or slave") >= 0)
         {
            UpdateChartMessage("", "CONFLICT\nAccount already exists as master or slave");
         }
         else
         {
            UpdateChartMessage("", "CONFLICT\nAccount already exists");
         }
         break;
         
      case 500:
         if(StringFind(response, "Failed to register master account") >= 0)
         {
            UpdateChartMessage("", "SERVER ERROR\nFailed to register master account\nRetrying in 30 seconds");
         }
         else if(StringFind(response, "Failed to register slave account") >= 0)
         {
            UpdateChartMessage("", "SERVER ERROR\nFailed to register slave account\nRetrying in 30 seconds");
         }
         else if(StringFind(response, "Failed to connect accounts") >= 0)
         {
            UpdateChartMessage("", "SERVER ERROR\nFailed to connect accounts\nRetrying in 30 seconds");
         }
         else if(StringFind(response, "Failed to save account configuration") >= 0)
         {
            UpdateChartMessage("", "SERVER ERROR\nFailed to save configuration\nRetrying in 30 seconds");
         }
         else if(StringFind(response, "Error writing CSV") >= 0)
         {
            UpdateChartMessage("", "SERVER ERROR\nError writing order data\nRetrying in 30 seconds");
         }
         else if(StringFind(response, "Error reading CSV") >= 0)
         {
            UpdateChartMessage("", "SERVER ERROR\nError reading order data\nRetrying in 30 seconds");
         }
         else if(StringFind(response, "Internal server error") >= 0)
         {
            UpdateChartMessage("", "SERVER ERROR\nInternal server error\nRetrying in 30 seconds");
         }
         else
         {
            UpdateChartMessage("", "SERVER ERROR\nRetrying in 30 seconds");
         }
         break;
         
      default:
         UpdateChartMessage("", "UNKNOWN ERROR\nCheck connection");
         break;
   }
}
```

### Status Update Frequency

- **Normal operation:** Update every 50 seconds (ping interval)
- **Error state:** Update immediately
- **Retry mode:** Update every 5 seconds
- **Pending account:** Update every 60 seconds

### Server Activity Monitoring

The server monitors account activity with the following settings:

- **Activity Timeout:** 60 seconds (accounts marked offline after 60s of inactivity)
- **Check Frequency:** Every 10 seconds (server checks activity status)
- **EA Ping Interval:** 50 seconds (EA sends ping every 50s)
- **Buffer Time:** 10 seconds (allows for network delays and timing variations)

This configuration ensures:
- EA has enough time to send pings (50s interval)
- Server detects offline accounts quickly (60s timeout)
- Minimal false positives from network delays

### Master Offline Handling

When a master account goes offline, the system automatically:

1. **Marks master as offline** after 60 seconds of inactivity
2. **Disables copy trading** for that master account
3. **Slaves receive "0" response** when requesting orders
4. **No new orders are processed** until master comes back online
5. **Existing orders remain** in the system but are not sent to slaves

### Position Management

**Important:** Both "0" and "[counter]" responses mean "no new orders available", NOT "close all positions".

**Slave EA should:**
- ✅ **Keep all existing positions open**
- ✅ **Continue monitoring for new orders**
- ✅ **Maintain stop loss and take profit levels**
- ✅ **Wait for master to come back online**

**Slave EA should NOT:**
- ❌ **Close existing positions when receiving "0" or "[counter]"**
- ❌ **Modify existing stop loss or take profit**
- ❌ **Stop monitoring for new orders**
- ❌ **Assume master is permanently offline**

### Response Types

| Response | Meaning | Master Status | Action |
|----------|---------|---------------|--------|
| `"0"` | No orders available | **Offline/Not Registered** | Keep positions, continue monitoring |
| `"[3]"` | No orders available | **Online but No Orders** | Keep positions, continue monitoring |
| `"[1]\n[order1]\n[order2]"` | Orders available | **Online with Orders** | Execute new orders |

**Slave EA Behavior when Master is Offline:**
- Receives `"0"` response from `/orders/neworder` endpoint
- Should display appropriate message to user
- Should continue polling but expect no orders
- Should **NOT** close existing positions (only no new orders)
- Should **NOT** attempt to execute any new trades
- Should wait for master to come back online
- Should maintain all existing open positions

**Master EA Behavior when Coming Back Online:**
- Sends ping to `/accounts/ping` endpoint
- System automatically re-enables copy trading
- New orders will be sent to slaves again
- Previous orders are not resent (only new orders)

### User-Friendly Messages

#### Connection Status:
```
🟢 CONNECTED - All systems operational
🟡 CONNECTING - Establishing connection
🔴 DISCONNECTED - Connection lost
```

#### Account Status:
```
📋 PENDING - Waiting for administrator setup
👑 MASTER - Sending trades to followers
📥 SLAVE - Following master trades
❌ ERROR - Check configuration
```

#### Trading Status:
```
📈 ACTIVE - Trading operations normal
⏸️ PAUSED - Trading temporarily stopped
🚫 DISABLED - Trading disabled by admin
```

---
