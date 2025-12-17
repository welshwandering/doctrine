# Gin Style Guide

> [Doctrine](../../README.md) > [Frameworks](../README.md) > Gin

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.txt).

Extends [Go style guide](../languages/go.md) with Gin-specific conventions.

**Target Version**: Gin 1.11+ with Go 1.23

## Quick Reference

All Go tooling applies. Additional considerations:

| Task | Tool | Command |
|------|------|---------|
| Install | go get | `go get -u github.com/gin-gonic/gin` |
| Run dev | go run | `go run main.go` |
| Test | go test + httptest | `go test -v ./...` |
| Build | go build | `go build -o bin/app` |

## Why Gin?

Gin[^1] is a high-performance HTTP web framework written in Go that provides:

- **High performance**: Up to 40x faster than Martini due to httprouter[^2] under the hood
- **Middleware ecosystem**: Rich selection of built-in and community middleware
- **Request validation**: Built-in binding and validation using struct tags
- **Error handling**: Centralized error management with custom error handlers
- **JSON rendering**: Fast JSON serialization with minimal allocations
- **Lightweight**: Minimal dependencies and small binary size

Use Gin for building RESTful APIs, microservices, and web applications requiring high throughput. Choose Echo[^3] for similar performance with different API design, or Fiber[^4] for Express.js-like syntax. Use the standard library net/http for maximum control and zero dependencies.

## Project Structure

Projects **SHOULD** organize Gin applications using a layered architecture:

```
my_app/
├── cmd/
│   └── server/
│       └── main.go           # Application entry point
├── internal/
│   ├── api/
│   │   ├── handler/          # HTTP handlers
│   │   │   ├── user.go
│   │   │   └── health.go
│   │   ├── middleware/       # Custom middleware
│   │   │   ├── auth.go
│   │   │   └── logging.go
│   │   └── router/           # Route definitions
│   │       └── router.go
│   ├── model/                # Domain models
│   │   └── user.go
│   ├── repository/           # Data access layer
│   │   └── user.go
│   ├── service/              # Business logic
│   │   └── user.go
│   └── config/               # Configuration
│       └── config.go
├── pkg/                      # Public libraries
│   └── validator/
├── migrations/               # Database migrations
├── go.mod
├── go.sum
└── .env
```

```go
// cmd/server/main.go
package main

import (
	"log"
	"myapp/internal/api/router"
	"myapp/internal/config"
)

func main() {
	cfg, err := config.Load()
	if err != nil {
		log.Fatalf("failed to load config: %v", err)
	}

	r := router.NewRouter(cfg)

	if err := r.Run(cfg.ServerAddress); err != nil {
		log.Fatalf("failed to start server: %v", err)
	}
}
```

**Why**: Separating concerns into handlers, services, and repositories improves testability, enables dependency injection, and makes the codebase maintainable as it grows. The `internal/` directory prevents accidental imports by other projects.

## Router Setup

Projects **MUST** configure the router in a dedicated function for testability:

```go
// internal/api/router/router.go
package router

import (
	"myapp/internal/api/handler"
	"myapp/internal/api/middleware"
	"myapp/internal/config"

	"github.com/gin-gonic/gin"
)

func NewRouter(cfg *config.Config) *gin.Engine {
	// Use release mode in production
	if cfg.Environment == "production" {
		gin.SetMode(gin.ReleaseMode)
	}

	r := gin.New()

	// Global middleware
	r.Use(gin.Recovery())
	r.Use(middleware.Logger())
	r.Use(middleware.CORS())

	// Health check (no auth required)
	r.GET("/health", handler.HealthCheck)

	// API v1 routes
	v1 := r.Group("/api/v1")
	{
		// Public routes
		v1.POST("/auth/login", handler.Login)
		v1.POST("/auth/register", handler.Register)

		// Protected routes
		authorized := v1.Group("")
		authorized.Use(middleware.Auth())
		{
			users := authorized.Group("/users")
			{
				users.GET("", handler.ListUsers)
				users.GET("/:id", handler.GetUser)
				users.POST("", handler.CreateUser)
				users.PUT("/:id", handler.UpdateUser)
				users.DELETE("/:id", handler.DeleteUser)
			}
		}
	}

	return r
}
```

**Why**: Centralizing route configuration improves discoverability and makes it easy to see the entire API surface. Using route groups enables logical organization and scoped middleware application.

## RESTful Routing Patterns

Projects **MUST** follow RESTful conventions for resource routing:

```go
// DO: Use standard HTTP methods and resource-based URLs
r.GET("/api/v1/posts", handler.ListPosts)          // List all posts
r.POST("/api/v1/posts", handler.CreatePost)        // Create new post
r.GET("/api/v1/posts/:id", handler.GetPost)        // Get specific post
r.PUT("/api/v1/posts/:id", handler.UpdatePost)     // Update entire post
r.PATCH("/api/v1/posts/:id", handler.PatchPost)    // Partial update
r.DELETE("/api/v1/posts/:id", handler.DeletePost)  // Delete post

// Nested resources
r.GET("/api/v1/posts/:id/comments", handler.ListPostComments)
r.POST("/api/v1/posts/:id/comments", handler.CreateComment)

// DON'T: Use verbs in URLs
r.GET("/api/v1/getPost/:id", handler.GetPost)      // Bad
r.POST("/api/v1/createPost", handler.CreatePost)   // Bad
r.DELETE("/api/v1/deletePost/:id", handler.DeletePost)  // Bad
```

## Handler Structure

Handlers **MUST** follow a consistent structure with clear separation of concerns:

```go
// internal/api/handler/user.go
package handler

import (
	"myapp/internal/model"
	"myapp/internal/service"
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
)

type UserHandler struct {
	userService service.UserService
}

func NewUserHandler(userService service.UserService) *UserHandler {
	return &UserHandler{
		userService: userService,
	}
}

// GetUser retrieves a user by ID
// @Summary Get user by ID
// @Description Get user details by user ID
// @Tags users
// @Accept json
// @Produce json
// @Param id path int true "User ID"
// @Success 200 {object} model.User
// @Failure 400 {object} ErrorResponse
// @Failure 404 {object} ErrorResponse
// @Router /api/v1/users/{id} [get]
func (h *UserHandler) GetUser(c *gin.Context) {
	id, err := strconv.ParseInt(c.Param("id"), 10, 64)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid user ID"})
		return
	}

	user, err := h.userService.GetByID(c.Request.Context(), id)
	if err != nil {
		c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
		return
	}

	c.JSON(http.StatusOK, user)
}

// CreateUser creates a new user
func (h *UserHandler) CreateUser(c *gin.Context) {
	var req model.CreateUserRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	user, err := h.userService.Create(c.Request.Context(), &req)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	c.JSON(http.StatusCreated, user)
}
```

**Why**: Dependency injection via handler structs enables testing with mock services. Using the request context allows request-scoped cancellation and tracing. Swagger annotations[^5] enable automatic API documentation generation.

## Middleware

### Built-in Middleware

Projects **MUST** use appropriate built-in middleware:

```go
import (
	"github.com/gin-gonic/gin"
)

func NewRouter() *gin.Engine {
	r := gin.New()

	// MUST: Recovery middleware prevents panics from crashing the server
	r.Use(gin.Recovery())

	// SHOULD: Logger middleware for request logging
	r.Use(gin.Logger())

	// MAY: Custom middleware for specific needs
	r.Use(CustomMiddleware())

	return r
}
```

**Why**: `gin.Recovery()` prevents panics from crashing the entire application. `gin.Logger()` provides request logging but **SHOULD** be replaced with structured logging in production.

### Custom Middleware

Projects **SHOULD** implement custom middleware for cross-cutting concerns:

```go
// internal/api/middleware/auth.go
package middleware

import (
	"net/http"
	"strings"

	"github.com/gin-gonic/gin"
)

// Auth validates JWT tokens
func Auth() gin.HandlerFunc {
	return func(c *gin.Context) {
		authHeader := c.GetHeader("Authorization")
		if authHeader == "" {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "missing authorization header"})
			c.Abort()
			return
		}

		parts := strings.SplitN(authHeader, " ", 2)
		if len(parts) != 2 || parts[0] != "Bearer" {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid authorization header"})
			c.Abort()
			return
		}

		token := parts[1]
		claims, err := validateToken(token)
		if err != nil {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
			c.Abort()
			return
		}

		// Store user info in context for handlers
		c.Set("userID", claims.UserID)
		c.Set("email", claims.Email)

		c.Next()
	}
}
```

```go
// internal/api/middleware/logging.go
package middleware

import (
	"log/slog"
	"time"

	"github.com/gin-gonic/gin"
)

// Logger provides structured logging for requests
func Logger() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		path := c.Request.URL.Path
		method := c.Request.Method

		c.Next()

		latency := time.Since(start)
		status := c.Writer.Status()

		slog.Info("request completed",
			"method", method,
			"path", path,
			"status", status,
			"latency", latency,
			"ip", c.ClientIP(),
		)
	}
}
```

**Why**: Custom middleware enables consistent handling of authentication, logging, and CORS across all routes. Using `c.Set()` and `c.Get()` allows middleware to pass data to handlers without global state.

## Request Validation and Binding

Projects **MUST** use Gin's binding and validation features:

```go
// internal/model/user.go
package model

type CreateUserRequest struct {
	Email    string `json:"email" binding:"required,email"`
	Username string `json:"username" binding:"required,min=3,max=50"`
	Password string `json:"password" binding:"required,min=8"`
	Age      int    `json:"age" binding:"omitempty,gte=18,lte=120"`
}

type UpdateUserRequest struct {
	Email    string `json:"email" binding:"omitempty,email"`
	Username string `json:"username" binding:"omitempty,min=3,max=50"`
	Age      int    `json:"age" binding:"omitempty,gte=18,lte=120"`
}

type User struct {
	ID        int64     `json:"id"`
	Email     string    `json:"email"`
	Username  string    `json:"username"`
	Age       int       `json:"age,omitempty"`
	CreatedAt time.Time `json:"created_at"`
}
```

```go
// Binding examples
func (h *UserHandler) CreateUser(c *gin.Context) {
	var req model.CreateUserRequest

	// ShouldBindJSON validates and binds JSON
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{
			"error": "validation failed",
			"details": err.Error(),
		})
		return
	}

	// Request is now validated
	user, err := h.userService.Create(c.Request.Context(), &req)
	// ...
}

// Query parameter binding
type ListUsersQuery struct {
	Page     int    `form:"page" binding:"omitempty,gte=1"`
	PageSize int    `form:"page_size" binding:"omitempty,gte=1,lte=100"`
	Sort     string `form:"sort" binding:"omitempty,oneof=name email created_at"`
}

func (h *UserHandler) ListUsers(c *gin.Context) {
	var query ListUsersQuery
	if err := c.ShouldBindQuery(&query); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	// Use validated query parameters
}

// URI parameter binding
type UserIDParam struct {
	ID int64 `uri:"id" binding:"required,gte=1"`
}

func (h *UserHandler) GetUser(c *gin.Context) {
	var param UserIDParam
	if err := c.ShouldBindUri(&param); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	// Use param.ID
}
```

**Why**: Gin uses the validator package[^6] under the hood, providing comprehensive validation rules via struct tags. This eliminates manual validation code and produces consistent error messages.

### Custom Validators

Projects **MAY** register custom validation functions:

```go
// pkg/validator/custom.go
package validator

import (
	"regexp"

	"github.com/go-playground/validator/v10"
)

var phoneRegex = regexp.MustCompile(`^\+?[1-9]\d{1,14}$`)

// ValidatePhone checks if the value is a valid phone number
func ValidatePhone(fl validator.FieldLevel) bool {
	return phoneRegex.MatchString(fl.Field().String())
}

// RegisterCustomValidators registers all custom validators
func RegisterCustomValidators(v *validator.Validate) error {
	if err := v.RegisterValidation("phone", ValidatePhone); err != nil {
		return err
	}
	return nil
}
```

```go
// Usage in model
type CreateUserRequest struct {
	Phone string `json:"phone" binding:"required,phone"`
}
```

## Error Handling

Projects **MUST** implement consistent error handling with custom error types:

```go
// internal/api/handler/error.go
package handler

import (
	"errors"
	"net/http"

	"github.com/gin-gonic/gin"
)

type ErrorResponse struct {
	Error   string `json:"error"`
	Message string `json:"message,omitempty"`
	Code    string `json:"code,omitempty"`
}

// Custom error types
var (
	ErrNotFound          = errors.New("resource not found")
	ErrUnauthorized      = errors.New("unauthorized")
	ErrForbidden         = errors.New("forbidden")
	ErrValidation        = errors.New("validation failed")
	ErrInternalServer    = errors.New("internal server error")
)

// HandleError processes errors and returns appropriate HTTP responses
func HandleError(c *gin.Context, err error) {
	switch {
	case errors.Is(err, ErrNotFound):
		c.JSON(http.StatusNotFound, ErrorResponse{
			Error: "not_found",
			Message: err.Error(),
		})
	case errors.Is(err, ErrUnauthorized):
		c.JSON(http.StatusUnauthorized, ErrorResponse{
			Error: "unauthorized",
			Message: err.Error(),
		})
	case errors.Is(err, ErrValidation):
		c.JSON(http.StatusBadRequest, ErrorResponse{
			Error: "validation_error",
			Message: err.Error(),
		})
	default:
		// Don't expose internal errors to clients
		c.JSON(http.StatusInternalServerError, ErrorResponse{
			Error: "internal_error",
			Message: "an unexpected error occurred",
		})
	}
}
```

```go
// Usage in handlers
func (h *UserHandler) GetUser(c *gin.Context) {
	id := c.Param("id")

	user, err := h.userService.GetByID(c.Request.Context(), id)
	if err != nil {
		HandleError(c, err)
		return
	}

	c.JSON(http.StatusOK, user)
}
```

**Why**: Centralized error handling ensures consistent error responses across the API. Using `errors.Is()` allows error wrapping while maintaining error type checking.

## Testing

Projects **MUST** test Gin handlers using httptest:

```go
// internal/api/handler/user_test.go
package handler

import (
	"bytes"
	"encoding/json"
	"errors"
	"myapp/internal/model"
	"myapp/internal/service/mocks"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/gin-gonic/gin"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

func TestGetUser(t *testing.T) {
	gin.SetMode(gin.TestMode)

	tests := []struct {
		name           string
		userID         string
		mockSetup      func(*mocks.UserService)
		expectedStatus int
		expectedBody   map[string]interface{}
	}{
		{
			name:   "success",
			userID: "1",
			mockSetup: func(m *mocks.UserService) {
				m.On("GetByID", mock.Anything, int64(1)).
					Return(&model.User{
						ID:       1,
						Email:    "test@example.com",
						Username: "testuser",
					}, nil)
			},
			expectedStatus: http.StatusOK,
		},
		{
			name:   "user not found",
			userID: "999",
			mockSetup: func(m *mocks.UserService) {
				m.On("GetByID", mock.Anything, int64(999)).
					Return(nil, ErrNotFound)
			},
			expectedStatus: http.StatusNotFound,
		},
		{
			name:           "invalid user ID",
			userID:         "abc",
			mockSetup:      func(m *mocks.UserService) {},
			expectedStatus: http.StatusBadRequest,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// Setup
			mockService := new(mocks.UserService)
			tt.mockSetup(mockService)

			handler := NewUserHandler(mockService)
			router := gin.New()
			router.GET("/users/:id", handler.GetUser)

			// Execute
			req := httptest.NewRequest(http.MethodGet, "/users/"+tt.userID, nil)
			w := httptest.NewRecorder()
			router.ServeHTTP(w, req)

			// Assert
			assert.Equal(t, tt.expectedStatus, w.Code)
			mockService.AssertExpectations(t)
		})
	}
}

func TestCreateUser(t *testing.T) {
	gin.SetMode(gin.TestMode)

	t.Run("success", func(t *testing.T) {
		mockService := new(mocks.UserService)
		mockService.On("Create", mock.Anything, mock.AnythingOfType("*model.CreateUserRequest")).
			Return(&model.User{
				ID:       1,
				Email:    "test@example.com",
				Username: "testuser",
			}, nil)

		handler := NewUserHandler(mockService)
		router := gin.New()
		router.POST("/users", handler.CreateUser)

		reqBody := model.CreateUserRequest{
			Email:    "test@example.com",
			Username: "testuser",
			Password: "password123",
		}
		body, _ := json.Marshal(reqBody)

		req := httptest.NewRequest(http.MethodPost, "/users", bytes.NewBuffer(body))
		req.Header.Set("Content-Type", "application/json")
		w := httptest.NewRecorder()

		router.ServeHTTP(w, req)

		assert.Equal(t, http.StatusCreated, w.Code)
		mockService.AssertExpectations(t)
	})

	t.Run("validation error", func(t *testing.T) {
		mockService := new(mocks.UserService)
		handler := NewUserHandler(mockService)
		router := gin.New()
		router.POST("/users", handler.CreateUser)

		// Invalid request (missing required fields)
		reqBody := map[string]interface{}{
			"email": "invalid-email",
		}
		body, _ := json.Marshal(reqBody)

		req := httptest.NewRequest(http.MethodPost, "/users", bytes.NewBuffer(body))
		req.Header.Set("Content-Type", "application/json")
		w := httptest.NewRecorder()

		router.ServeHTTP(w, req)

		assert.Equal(t, http.StatusBadRequest, w.Code)
	})
}
```

**Why**: Using httptest allows testing handlers without starting a server. Mocking the service layer isolates handler logic from business logic and database dependencies.

### Integration Testing

Projects **SHOULD** include integration tests for critical paths:

```go
// internal/api/integration_test.go
package api

import (
	"encoding/json"
	"myapp/internal/config"
	"myapp/internal/api/router"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestUserFlow(t *testing.T) {
	if testing.Short() {
		t.Skip("skipping integration test")
	}

	// Setup test database
	cfg := config.LoadTest()
	r := router.NewRouter(cfg)

	// Register user
	t.Run("register", func(t *testing.T) {
		reqBody := `{"email":"test@example.com","username":"testuser","password":"password123"}`
		req := httptest.NewRequest(http.MethodPost, "/api/v1/auth/register",
			bytes.NewBufferString(reqBody))
		req.Header.Set("Content-Type", "application/json")
		w := httptest.NewRecorder()

		r.ServeHTTP(w, req)

		assert.Equal(t, http.StatusCreated, w.Code)
	})

	// Login
	var token string
	t.Run("login", func(t *testing.T) {
		reqBody := `{"email":"test@example.com","password":"password123"}`
		req := httptest.NewRequest(http.MethodPost, "/api/v1/auth/login",
			bytes.NewBufferString(reqBody))
		req.Header.Set("Content-Type", "application/json")
		w := httptest.NewRecorder()

		r.ServeHTTP(w, req)

		require.Equal(t, http.StatusOK, w.Code)

		var resp map[string]string
		err := json.Unmarshal(w.Body.Bytes(), &resp)
		require.NoError(t, err)
		token = resp["token"]
		require.NotEmpty(t, token)
	})

	// Access protected resource
	t.Run("get user", func(t *testing.T) {
		req := httptest.NewRequest(http.MethodGet, "/api/v1/users/1", nil)
		req.Header.Set("Authorization", "Bearer "+token)
		w := httptest.NewRecorder()

		r.ServeHTTP(w, req)

		assert.Equal(t, http.StatusOK, w.Code)
	})
}
```

## Configuration

Projects **MUST** use environment-based configuration:

```go
// internal/config/config.go
package config

import (
	"fmt"
	"os"
	"strconv"

	"github.com/joho/godotenv"
)

type Config struct {
	Environment   string
	ServerAddress string
	DatabaseURL   string
	JWTSecret     string
	LogLevel      string
}

func Load() (*Config, error) {
	// Load .env file in development
	_ = godotenv.Load()

	cfg := &Config{
		Environment:   getEnv("ENVIRONMENT", "development"),
		ServerAddress: getEnv("SERVER_ADDRESS", ":8080"),
		DatabaseURL:   getEnv("DATABASE_URL", ""),
		JWTSecret:     getEnv("JWT_SECRET", ""),
		LogLevel:      getEnv("LOG_LEVEL", "info"),
	}

	if cfg.DatabaseURL == "" {
		return nil, fmt.Errorf("DATABASE_URL is required")
	}

	if cfg.JWTSecret == "" && cfg.Environment == "production" {
		return nil, fmt.Errorf("JWT_SECRET is required in production")
	}

	return cfg, nil
}

func LoadTest() *Config {
	return &Config{
		Environment:   "test",
		ServerAddress: ":8080",
		DatabaseURL:   "sqlite::memory:",
		JWTSecret:     "test-secret",
		LogLevel:      "debug",
	}
}

func getEnv(key, defaultValue string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
	if value := os.Getenv(key); value != "" {
		if intVal, err := strconv.Atoi(value); err == nil {
			return intVal
		}
	}
	return defaultValue
}
```

**Why**: Environment-based configuration follows twelve-factor app principles, prevents secrets in version control, and enables different settings per deployment environment.

## Logging and Observability

Projects **MUST** use Go's standard library `log/slog` package for structured logging. The `slog` package (introduced in Go 1.21) provides structured, leveled logging with excellent performance.

### Logger Setup

Configure the global logger at application startup:

```go
// internal/logger/logger.go
package logger

import (
	"log/slog"
	"os"
)

// Setup configures the global slog logger
func Setup(env string, level slog.Level) {
	var handler slog.Handler

	opts := &slog.HandlerOptions{
		Level: level,
		// Add source file information in debug mode
		AddSource: level == slog.LevelDebug,
	}

	if env == "production" {
		// JSON output for production (log aggregation systems)
		handler = slog.NewJSONHandler(os.Stdout, opts)
	} else {
		// Human-readable output for development
		handler = slog.NewTextHandler(os.Stdout, opts)
	}

	slog.SetDefault(slog.New(handler))
}

// ParseLevel converts a string to slog.Level
func ParseLevel(level string) slog.Level {
	switch level {
	case "debug":
		return slog.LevelDebug
	case "info":
		return slog.LevelInfo
	case "warn":
		return slog.LevelWarn
	case "error":
		return slog.LevelError
	default:
		return slog.LevelInfo
	}
}
```

```go
// cmd/server/main.go
package main

import (
	"myapp/internal/config"
	"myapp/internal/logger"
)

func main() {
	cfg, err := config.Load()
	if err != nil {
		log.Fatalf("failed to load config: %v", err)
	}

	// Initialize structured logging
	logger.Setup(cfg.Environment, logger.ParseLevel(cfg.LogLevel))

	slog.Info("starting server",
		"address", cfg.ServerAddress,
		"environment", cfg.Environment,
	)

	// ... rest of setup
}
```

### Request Logging Middleware

Projects **SHOULD** use structured logging with context propagation:

```go
// internal/api/middleware/logging.go
package middleware

import (
	"context"
	"log/slog"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
)

type contextKey string

const loggerKey contextKey = "logger"

// RequestID adds a unique request ID to each request
func RequestID() gin.HandlerFunc {
	return func(c *gin.Context) {
		requestID := c.GetHeader("X-Request-ID")
		if requestID == "" {
			requestID = uuid.New().String()
		}
		c.Set("requestID", requestID)
		c.Header("X-Request-ID", requestID)
		c.Next()
	}
}

// StructuredLogger provides structured logging with request context
func StructuredLogger() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		path := c.Request.URL.Path
		method := c.Request.Method
		requestID, _ := c.Get("requestID")

		// Create request-scoped logger with common attributes
		reqLogger := slog.With(
			"request_id", requestID,
			"method", method,
			"path", path,
		)

		// Store logger in context for use in handlers
		ctx := context.WithValue(c.Request.Context(), loggerKey, reqLogger)
		c.Request = c.Request.WithContext(ctx)

		c.Next()

		latency := time.Since(start)
		status := c.Writer.Status()
		errorMsg := c.Errors.ByType(gin.ErrorTypePrivate).String()

		attrs := []slog.Attr{
			slog.String("request_id", requestID.(string)),
			slog.String("method", method),
			slog.String("path", path),
			slog.Int("status", status),
			slog.Duration("latency", latency),
			slog.String("ip", c.ClientIP()),
			slog.String("user_agent", c.Request.UserAgent()),
			slog.Int("bytes_written", c.Writer.Size()),
		}

		if errorMsg != "" {
			attrs = append(attrs, slog.String("error", errorMsg))
		}

		level := slog.LevelInfo
		if status >= 500 {
			level = slog.LevelError
		} else if status >= 400 {
			level = slog.LevelWarn
		}

		slog.LogAttrs(c.Request.Context(), level, "request completed", attrs...)
	}
}

// LoggerFromContext retrieves the request-scoped logger
func LoggerFromContext(ctx context.Context) *slog.Logger {
	if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
		return logger
	}
	return slog.Default()
}
```

### Using the Logger in Handlers

Handlers **SHOULD** use the request-scoped logger for contextual logging:

```go
// internal/api/handler/user.go
package handler

import (
	"log/slog"
	"myapp/internal/api/middleware"
	"net/http"

	"github.com/gin-gonic/gin"
)

func (h *UserHandler) CreateUser(c *gin.Context) {
	logger := middleware.LoggerFromContext(c.Request.Context())

	var req model.CreateUserRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		logger.Warn("invalid request body", "error", err)
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	logger.Debug("creating user", "email", req.Email)

	user, err := h.userService.Create(c.Request.Context(), &req)
	if err != nil {
		logger.Error("failed to create user",
			"error", err,
			"email", req.Email,
		)
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	logger.Info("user created",
		"user_id", user.ID,
		"email", user.Email,
	)

	c.JSON(http.StatusCreated, user)
}
```

### Logging in Services

Services **SHOULD** accept a logger or use context-based logging:

```go
// internal/service/user.go
package service

import (
	"context"
	"log/slog"
)

type UserService struct {
	repo   UserRepository
	logger *slog.Logger
}

func NewUserService(repo UserRepository) *UserService {
	return &UserService{
		repo:   repo,
		logger: slog.Default().With("service", "user"),
	}
}

func (s *UserService) Create(ctx context.Context, req *model.CreateUserRequest) (*model.User, error) {
	s.logger.DebugContext(ctx, "creating user in database", "email", req.Email)

	user, err := s.repo.Create(ctx, req)
	if err != nil {
		s.logger.ErrorContext(ctx, "database error creating user",
			"error", err,
			"email", req.Email,
		)
		return nil, fmt.Errorf("create user: %w", err)
	}

	s.logger.InfoContext(ctx, "user created successfully",
		"user_id", user.ID,
	)

	return user, nil
}
```

### Logging with Groups

Use `slog.Group` to organize related attributes:

```go
func (h *OrderHandler) CreateOrder(c *gin.Context) {
	logger := middleware.LoggerFromContext(c.Request.Context())

	// Log with grouped attributes for better organization
	logger.Info("order created",
		slog.Group("order",
			slog.String("id", order.ID),
			slog.Float64("total", order.Total),
			slog.Int("item_count", len(order.Items)),
		),
		slog.Group("customer",
			slog.String("id", order.CustomerID),
			slog.String("email", order.CustomerEmail),
		),
	)
}
```

**Why slog**: Go's standard library `log/slog` provides structured logging without external dependencies. It offers excellent performance, type-safe attributes, log levels, and outputs JSON for production log aggregation systems. Context propagation maintains request-scoped logging attributes across service boundaries.

### Health Checks and Metrics

Projects **SHOULD** expose health check and metrics endpoints:

```go
// internal/api/handler/health.go
package handler

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type HealthHandler struct {
	db DBHealthChecker
}

type DBHealthChecker interface {
	Ping() error
}

func NewHealthHandler(db DBHealthChecker) *HealthHandler {
	return &HealthHandler{db: db}
}

// HealthCheck returns the service health status
func (h *HealthHandler) HealthCheck(c *gin.Context) {
	health := map[string]string{
		"status": "healthy",
	}

	if err := h.db.Ping(); err != nil {
		health["status"] = "unhealthy"
		health["database"] = "down"
		c.JSON(http.StatusServiceUnavailable, health)
		return
	}

	health["database"] = "up"
	c.JSON(http.StatusOK, health)
}

// ReadinessCheck determines if the service is ready to serve traffic
func (h *HealthHandler) ReadinessCheck(c *gin.Context) {
	if err := h.db.Ping(); err != nil {
		c.JSON(http.StatusServiceUnavailable, gin.H{
			"ready": false,
			"reason": "database not ready",
		})
		return
	}

	c.JSON(http.StatusOK, gin.H{"ready": true})
}
```

**Why**: Health checks enable orchestrators (Kubernetes, Docker) to monitor service health and restart unhealthy instances. Separating liveness and readiness checks prevents cascading failures during deployment.

## See Also

- [Go Style Guide](../languages/go.md) - Base Go conventions and tooling
- [Echo](https://echo.labstack.com/) - Alternative high-performance Go web framework
- [Fiber](https://gofiber.io/) - Express.js-inspired Go web framework

## References

[^1]: **Gin Web Framework** - High-performance HTTP web framework written in Go. Features martini-like API with 40x better performance. [https://gin-gonic.com/](https://gin-gonic.com/) | [GitHub](https://github.com/gin-gonic/gin)

[^2]: **httprouter** - High performance HTTP request router. Used as Gin's routing engine. [GitHub](https://github.com/julienschmidt/httprouter)

[^3]: **Echo** - High performance, minimalist Go web framework. Alternative to Gin with different design philosophy. [https://echo.labstack.com/](https://echo.labstack.com/) | [GitHub](https://github.com/labstack/echo)

[^4]: **Fiber** - Express.js-inspired web framework built on top of Fasthttp. Fastest HTTP engine for Go. [https://gofiber.io/](https://gofiber.io/) | [GitHub](https://github.com/gofiber/fiber)

[^5]: **swaggo/swag** - Automatically generate RESTful API documentation with Swagger 2.0 for Go. [GitHub](https://github.com/swaggo/swag)

[^6]: **go-playground/validator** - Go Struct and Field validation, including Cross Field, Cross Struct, Map, Slice and Array diving. [GitHub](https://github.com/go-playground/validator)
