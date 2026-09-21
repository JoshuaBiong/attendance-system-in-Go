This document serves as the Complete Golang Backend Implementation Blueprint. It provides the AI with the exact structure, dependencies, environment configuration, and detailed handler logic required to build the high-performance Gin API that manages both Google Sheets and Discord integration.

🛠️ Golang Backend Implementation Blueprint (Go + Gin)
1. ⚙️ Tech Stack & Dependencies
Component	Library/Package	Purpose
Framework	github.com/gin-gonic/gin	HTTP routing, middleware, and request/response handling.
Google Sheets	google.golang.org/api/sheets/v4	Accessing and writing data to Google Sheets.
Google Auth	golang.org/x/oauth2/google	Managing Service Account authentication for Sheets API.
JWT	github.com/golang-jwt/jwt/v5	Creating, signing, and validating JWTs for user session.
Configuration	github.com/joho/godotenv	Loading secret environment variables from .env file.
Validation	github.com/go-playground/validator/v10	Struct field validation (e.g., required, oneof).
UUID	github.com/google/uuid	Generating unique EntryID for each log.

Export to Sheets

2. 📝 Environment Variables (.env File)
The backend must securely load these configuration variables. These must not be hardcoded.

Variable Name	Purpose	Example Value/Notes
PORT	API listening port.	8080
JWT_SECRET	Secret key for signing/verifying JWTs.	my-secure-32-byte-secret
SHEET_ID	The ID of the Google Spreadsheet.	1ABc2DeF-gHIjK-3LMnoP_4QRsT5U6VWx
SHEET_SERVICE_ACCOUNT_PATH	Path to the Service Account JSON file.	./credentials.json
DISCORD_WEBHOOK_URL	The URL for the specific Discord channel webhook.	https://discord.com/api/webhooks/...

Export to Sheets

3. 📂 Project Structure (Golang)
/backend
├── cmd/
│   └── main.go           # Server entry point
├── config/
│   └── config.go         # Loads .env variables
├── internal/
│   ├── auth/
│   │   ├── auth.go       # JWT & Google OAuth logic
│   │   └── middleware.go # JWT validation middleware
│   ├── handlers/
│   │   └── log_handler.go# The core POST /log logic
│   ├── services/
│   │   ├── sheets.go     # Google Sheets API client and append logic
│   │   └── discord.go    # Discord Webhook/Bot API client
│   └── models/
│       ├── log_models.go # Structs for request/response/database rows
│       └── user_models.go# Struct for authenticated user (JWT payload)
├── go.mod
└── go.sum
4. 🔗 Core Data Structures (models/log_models.go)
This defines what the API receives and what it writes to the spreadsheet.

Go

// LogRequest defines the JSON payload from the React frontend
type LogRequest struct {
    // FullName is retrieved from the JWT payload, but included here for completeness
    FullName string `json:"fullName" binding:"required"` 
    LogType  string `json:"logType" binding:"required,oneof=Log In Log Out Overtime Break"`
    Date     string `json:"date" binding:"required,datetime=2006-01-02"` // YYYY-MM-DD
    Time     string `json:"time" binding:"required,datetime=15:04"`      // HH:MM
    Todo     string `json:"todo" binding:"required"`
    Remarks  string `json:"remarks"`
}

// SheetRow represents the data structure for one row in the Google Sheet (10 columns)
// This will be converted to []interface{} for the Sheets API.
type SheetRow struct {
    Timestamp        string
    FullName         string
    LogType          string
    Date             string
    Time             string
    CombinedDateTime string
    Task             string
    Remarks          string
    EntryID          string
    IPAddress        string
}
5. 🔌 Detailed Backend Workflow (handlers/log_handler.go)
This is the central logic for the POST /log endpoint.

A. Handler Initialization and Setup
Dependencies: The handler function must receive initialized services (e.g., sheetsService, discordService).

Middleware: Apply JWT Validation Middleware to ensure the request has a valid token and the user's details (e.g., FullName, UserID) are available in the Gin context.

Binding: Use c.ShouldBindJSON(&logRequest) to parse the incoming JSON payload into the LogRequest struct.

B. Data Preparation & Validation
Input Validation: Check for binding errors. If errors exist, return a 400 Bad Request with validation details.

Compute Fields:

Timestamp (Server): time.Now().Format(time.RFC3339)

EntryID: uuid.New().String()

IPAddress: c.ClientIP()

CombinedDateTime: Parse and merge logRequest.Date and logRequest.Time into a single, Go-native time.Time object, then format it (e.g., 2006-01-02 15:04:05).

Row Assembly: Populate the SheetRow struct with the calculated and received values, ensuring all 10 columns are present.

C. Concurrent External Operations (The Dual Write)
The core function uses Goroutines and sync.WaitGroup to execute the two API calls concurrently, minimizing latency.

Initialize WaitGroup: var wg sync.WaitGroup

Sheets Append (Action A - Goroutine):

wg.Add(1)

Launch a Goroutine that calls sheetsService.AppendRow(row).

The Goroutine ensures wg.Done() is called upon completion (or error).

Store the result/error in a channel or shared variable.

Discord Post (Action B - Goroutine):

wg.Add(1)

Launch a Goroutine that formats the message string and calls discordService.SendMessage(formattedMessage).

The Goroutine ensures wg.Done() is called upon completion (or error).

Store the result/error in a channel or shared variable.

Wait: wg.Wait() blocks until both Goroutines complete.

D. Final Response
Error Handling: Check the results from both Action A and Action B.

If Action A (Sheets) fails, return a 500 Internal Server Error stating the log failed to save.

If Action B (Discord) fails, log the error internally (high priority) but still return a 200 OK success response (the log is saved, notification is non-critical).

Success: If the Sheets write is successful, return a 201 Created status with a success JSON payload:

JSON

{
    "success": true,
    "entryID": "...",
    "message": "Log recorded. Discord notification status: success/failed" 
}
6. 🌐 Service Clients
A. Google Sheets Client (services/sheets.go)
Initialization: Use option.WithCredentialsFile() and sheets.NewService() to create the Sheets client, ensuring the necessary scopes are requested.

AppendRow(row SheetRow) function: Takes the structured data, converts it to []interface{}, and executes the sheets.Service.Spreadsheets.Values.Append() call.

B. Discord Client (services/discord.go)
Initialization: Stores the Discord Webhook URL/Bot Token loaded from the config.

SendMessage(message string) function:

Constructs the JSON payload: {"content": message}.

Uses the standard Go net/http package to send a POST request to the Discord endpoint.

Handles the required Authorization: Bot <TOKEN> header if using a bot token, or simply sends to the Webhook URL.

This context provides a direct, highly implementable blueprint for the AI to generate the Go code with correct dependencies, structure, and concurrency logic.

Which part of the implementation would you like to see the code for first? (e.g., The POST /log Handler, or The Sheets Service).