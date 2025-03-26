# Anvaya Backend Implementation Reference

## Project Structure
```
anvaya-backend/
├── cmd/            # Application entry points
│   └── server/     # Main server application
├── internal/       # Private application code
│   ├── auth/       # Authentication logic
│   ├── models/     # Data models
│   ├── handlers/   # API handlers
│   ├── services/   # Business logic
│   └── database/   # Database interaction
├── pkg/            # Public libraries that can be used by external applications
│   ├── utils/      # Utility functions
│   └── middleware/ # Middleware components
├── configs/        # Configuration files
├── migrations/     # Database migrations
├── scripts/        # Utility scripts
├── .env            # Environment variables (not committed to version control)
└── go.mod          # Go module file
```

## Technology Stack Decisions

### 1. HTTP Routing
**Selected:** Gorilla Mux

**Rationale:**
- Flexible URL routing with named parameters
- Robust pattern matching for routes
- Built-in middleware support
- Compatible with standard library's http.Handler interface

**Implementation Notes:**
```go
// Example usage will be added here
```

### 2. Database Access
**Selected:** sqlx

**Rationale:**
- Extends the standard database/sql package with convenient features
- Maintains close connection to SQL without heavy abstraction
- Efficient query execution with named parameters
- Streamlined scanning of query results into structs
- Better performance than full ORMs while reducing boilerplate code

**Implementation Notes:**
```go
// Example of using sqlx to connect to PostgreSQL
import (
    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
    "log"
)

// Define a struct that maps to a database table
type User struct {
    ID        int    `db:"id"`
    FirstName string `db:"first_name"`
    LastName  string `db:"last_name"`
    Phone     string `db:"phone"`
    Email     string `db:"email"`
}

// Connect to the database
func connectDB() (*sqlx.DB, error) {
    connStr := "host=your-supabase-host port=5432 user=postgres password=your-password dbname=postgres sslmode=require"
    db, err := sqlx.Connect("postgres", connStr)
    if err != nil {
        return nil, err
    }
    return db, nil
}

// Query example with named parameters
func getUserByPhone(db *sqlx.DB, phone string) (User, error) {
    var user User
    err := db.Get(&user, "SELECT * FROM users WHERE phone = $1", phone)
    return user, err
}

// Insert example
func createUser(db *sqlx.DB, user User) (int, error) {
    query := `
        INSERT INTO users (first_name, last_name, phone, email)
        VALUES ($1, $2, $3, $4)
        RETURNING id
    `
    var id int
    err := db.QueryRow(query, user.FirstName, user.LastName, user.Phone, user.Email).Scan(&id)
    return id, err
}
```

### 3. Authentication
**Selected:** JWT (JSON Web Tokens)

**Rationale:**
- Stateless authentication mechanism
- Well-suited for REST APIs
- Efficient for service-to-service communication
- Supports role-based access control needed for patient/psychologist roles

**Implementation Notes:**
```go
// Example usage will be added here
```

### 4. Configuration Management
**Selected:** Viper

**Rationale:**
- Supports multiple configuration formats (JSON, TOML, YAML, HCL, env vars)
- Hierarchical configuration structure
- Environment variable binding and overrides
- Live watching and re-reading of config files
- Reading from remote config systems (etcd, Consul)
- Explicit configuration validation

**Implementation Notes:**
```go
// Example of using Viper for configuration
import (
    "fmt"
    "github.com/spf13/viper"
)

func initConfig() error {
    // Set default values
    viper.SetDefault("server.port", 8080)
    viper.SetDefault("database.max_connections", 20)
    
    // Look for config in the config directory
    viper.AddConfigPath("./configs")
    viper.SetConfigName("config") // config file name without extension
    viper.SetConfigType("yaml")   // YAML format
    
    // Read the config file
    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); ok {
            // Config file not found; use defaults and env vars
            fmt.Println("No config file found, using defaults and environment variables")
        } else {
            // Config file was found but another error occurred
            return err
        }
    }
    
    // Override from environment variables
    // Use environment variables like APP_SERVER_PORT
    viper.SetEnvPrefix("APP")  // Prefix for env vars
    viper.AutomaticEnv()       // Read environment variables
    viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    
    return nil
}

// Usage example
func getConfig() {
    port := viper.GetInt("server.port")
    dbHost := viper.GetString("database.host")
    maxConn := viper.GetInt("database.max_connections")
    
    fmt.Printf("Server will run on port %d\n", port)
    fmt.Printf("Database host: %s with %d max connections\n", dbHost, maxConn)
}

// Example config.yaml file:
/*
server:
  port: 8080
  timeout: 30
database:
  host: your-supabase-host
  port: 5432
  user: postgres
  max_connections: 20
security:
  jwt_secret: your-secret-key
  token_expiry: 24h
*/
```

**Viper Deep Dive:**
Viper is powerful because it:

1. **Provides a single source of truth** for configuration across your application
2. **Supports hierarchical configurations** with nested keys
3. **Allows for flexible environment overrides** - you can override any config value with an env var
4. **Handles live reloading** - your app can respond to config changes without restart
5. **Works with remote config systems** like etcd or Consul for distributed setups
6. **Can unmarshal config directly into Go structs**:
   ```go
   type DBConfig struct {
       Host     string
       Port     int
       User     string
       Password string
       Name     string
   }
   
   var dbConfig DBConfig
   viper.UnmarshalKey("database", &dbConfig)
   ```

### 5. Logging
**Selected:** logrus

**Rationale:**
- Structured logging with support for fields
- Flexible output formatting (JSON, text)
- Multiple log levels with customizable thresholds
- Hook system for sending logs to multiple destinations
- Middleware integration for HTTP request logging
- Widely adopted in the Go community

**Implementation Notes:**
```go
// Example of using logrus for logging
import (
    "os"
    "github.com/sirupsen/logrus"
    "github.com/gorilla/mux"
    "net/http"
)

// Initialize the logger
func initLogger() *logrus.Logger {
    logger := logrus.New()
    
    // Set log format to JSON for better parsing in log management systems
    logger.SetFormatter(&logrus.JSONFormatter{
        TimestampFormat: "2006-01-02 15:04:05",
        PrettyPrint:     false,
    })
    
    // Output to stdout
    logger.SetOutput(os.Stdout)
    
    // Set log level based on environment
    if os.Getenv("APP_ENV") == "production" {
        logger.SetLevel(logrus.InfoLevel)
    } else {
        logger.SetLevel(logrus.DebugLevel)
    }
    
    return logger
}

// Example middleware for HTTP request logging
func LoggingMiddleware(logger *logrus.Logger) mux.MiddlewareFunc {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Log request details
            logger.WithFields(logrus.Fields{
                "method":     r.Method,
                "path":       r.URL.Path,
                "remote_ip":  r.RemoteAddr,
                "user_agent": r.UserAgent(),
            }).Info("HTTP request received")
            
            // Call the next handler
            next.ServeHTTP(w, r)
        })
    }
}

// Usage examples in application code
func exampleUsage(logger *logrus.Logger) {
    // Simple logging
    logger.Info("Application started")
    
    // Logging with fields for better context
    logger.WithFields(logrus.Fields{
        "user_id":   123,
        "component": "appointment_service",
    }).Info("Appointment created successfully")
    
    // Error logging with stack trace
    err := someFunction()
    if err != nil {
        logger.WithError(err).Error("Failed to process appointment")
    }
    
    // Different log levels
    logger.Debug("This is debug information")
    logger.Info("This is informational")
    logger.Warn("This is a warning")
    logger.Error("This is an error")
    
    // You can also create child loggers with preset fields
    appointmentLogger := logger.WithFields(logrus.Fields{
        "component": "appointment_service",
    })
    
    appointmentLogger.Info("This log entry is related to appointments")
}
```

## Core API Endpoints

### Patient APIs
- TBD

### Psychologist APIs
- TBD

### WhatsApp Integration APIs
- TBD

## Database Interaction
- TBD

## Authentication Flow
- TBD

## Deployment Considerations
- TBD
