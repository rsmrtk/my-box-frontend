# 🚀 Backend API Documentation

## 📚 Зміст
1. [Архітектура](#архітектура)
2. [Структура проекту](#структура-проекту)
3. [API Endpoints](#api-endpoints)
4. [Моделі даних](#моделі-даних)
5. [Authentication](#authentication)
6. [Error Handling](#error-handling)
7. [Go Best Practices](#go-best-practices)
8. [Testing](#testing)

## 🏗 Архітектура

### Clean Architecture Layers

```
┌────────────────────────────────────────┐
│          HTTP Layer (Handlers)         │
├────────────────────────────────────────┤
│         Service Layer (Business)       │
├────────────────────────────────────────┤
│      Repository Layer (Database)       │
├────────────────────────────────────────┤
│         Infrastructure (DB, Cache)     │
└────────────────────────────────────────┘
```

## 📁 Структура проекту

```
backend/
├── cmd/
│   └── server/
│       └── main.go              # Entry point
├── internal/
│   ├── config/                  # Configuration
│   │   └── config.go
│   ├── domain/                  # Domain models
│   │   ├── transaction.go
│   │   ├── user.go
│   │   └── category.go
│   ├── handler/                 # HTTP handlers
│   │   ├── auth.go
│   │   ├── transaction.go
│   │   ├── user.go
│   │   └── middleware.go
│   ├── service/                 # Business logic
│   │   ├── auth_service.go
│   │   ├── transaction_service.go
│   │   └── statistics_service.go
│   ├── repository/              # Data access
│   │   ├── postgres/
│   │   │   ├── transaction_repo.go
│   │   │   └── user_repo.go
│   │   └── interfaces.go
│   └── utils/                   # Utilities
│       ├── validator.go
│       └── response.go
├── pkg/                          # Public packages
│   ├── auth/
│   │   └── jwt.go
│   └── database/
│       └── postgres.go
├── migrations/                   # DB migrations
├── docs/                         # API documentation
└── tests/                        # Integration tests
```

## 🔗 API Endpoints

### Base URL
```
Production: https://api.finance-dashboard.com
Development: http://localhost:8080
```

### Authentication Endpoints

#### POST /api/auth/register
```json
Request:
{
  "email": "user@example.com",
  "password": "securePassword123",
  "name": "John Doe"
}

Response: 201 Created
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "John Doe",
    "created_at": "2024-01-01T00:00:00Z"
  },
  "token": "jwt-token-here"
}
```

#### POST /api/auth/login
```json
Request:
{
  "email": "user@example.com",
  "password": "securePassword123"
}

Response: 200 OK
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "token": "jwt-token-here",
  "refresh_token": "refresh-token-here"
}
```

#### POST /api/auth/refresh
```json
Request:
{
  "refresh_token": "refresh-token-here"
}

Response: 200 OK
{
  "token": "new-jwt-token",
  "refresh_token": "new-refresh-token"
}
```

### Transaction Endpoints

#### GET /api/transactions
```
Query Parameters:
- page (int): Page number (default: 1)
- limit (int): Items per page (default: 20)
- category (string): Filter by category
- from_date (string): Start date (YYYY-MM-DD)
- to_date (string): End date (YYYY-MM-DD)
- sort (string): Sort field (amount, date, category)
- order (string): Sort order (asc, desc)

Headers:
Authorization: Bearer <token>

Response: 200 OK
{
  "data": [
    {
      "id": "uuid",
      "amount": 150.00,
      "category": "food",
      "description": "Grocery shopping",
      "type": "expense",
      "date": "2024-01-15",
      "created_at": "2024-01-15T10:30:00Z",
      "updated_at": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "pages": 8
  }
}
```

#### GET /api/transactions/:id
```
Headers:
Authorization: Bearer <token>

Response: 200 OK
{
  "id": "uuid",
  "amount": 150.00,
  "category": "food",
  "description": "Grocery shopping",
  "type": "expense",
  "date": "2024-01-15",
  "tags": ["groceries", "weekly"],
  "receipt_url": "https://storage.example.com/receipts/uuid.pdf",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

#### POST /api/transactions
```json
Request:
Headers:
Authorization: Bearer <token>

Body:
{
  "amount": 150.00,
  "category": "food",
  "description": "Grocery shopping",
  "type": "expense",
  "date": "2024-01-15",
  "tags": ["groceries", "weekly"]
}

Response: 201 Created
{
  "id": "uuid",
  "amount": 150.00,
  "category": "food",
  "description": "Grocery shopping",
  "type": "expense",
  "date": "2024-01-15",
  "tags": ["groceries", "weekly"],
  "created_at": "2024-01-15T10:30:00Z"
}
```

#### PUT /api/transactions/:id
```json
Request:
Headers:
Authorization: Bearer <token>

Body:
{
  "amount": 175.00,
  "description": "Grocery shopping - updated"
}

Response: 200 OK
{
  "id": "uuid",
  "amount": 175.00,
  "category": "food",
  "description": "Grocery shopping - updated",
  "type": "expense",
  "date": "2024-01-15",
  "updated_at": "2024-01-15T11:00:00Z"
}
```

#### DELETE /api/transactions/:id
```
Headers:
Authorization: Bearer <token>

Response: 204 No Content
```

### Statistics Endpoints

#### GET /api/statistics/overview
```
Query Parameters:
- period (string): month, quarter, year
- year (int): Year (default: current)
- month (int): Month (1-12)

Headers:
Authorization: Bearer <token>

Response: 200 OK
{
  "period": "2024-01",
  "income": {
    "total": 5000.00,
    "count": 2,
    "average": 2500.00
  },
  "expenses": {
    "total": 3200.00,
    "count": 45,
    "average": 71.11
  },
  "balance": 1800.00,
  "savings_rate": 36.00,
  "top_categories": [
    {
      "category": "food",
      "amount": 800.00,
      "percentage": 25.00,
      "count": 15
    }
  ],
  "trend": {
    "income_change": 10.5,
    "expense_change": -5.2
  }
}
```

#### GET /api/statistics/monthly
```
Query Parameters:
- year (int): Year (default: current)

Headers:
Authorization: Bearer <token>

Response: 200 OK
{
  "year": 2024,
  "months": [
    {
      "month": 1,
      "income": 5000.00,
      "expenses": 3200.00,
      "balance": 1800.00,
      "categories": {
        "food": 800.00,
        "transport": 400.00,
        "utilities": 300.00
      }
    }
  ]
}
```

## 📊 Моделі даних

### Go Domain Models

```go
// domain/user.go
package domain

import (
    "time"
    "github.com/google/uuid"
)

type User struct {
    ID           uuid.UUID  `json:"id" db:"id"`
    Email        string     `json:"email" db:"email"`
    Name         string     `json:"name" db:"name"`
    PasswordHash string     `json:"-" db:"password_hash"`
    Currency     string     `json:"currency" db:"currency"`
    CreatedAt    time.Time  `json:"created_at" db:"created_at"`
    UpdatedAt    time.Time  `json:"updated_at" db:"updated_at"`
    DeletedAt    *time.Time `json:"-" db:"deleted_at"`
}

func (u *User) Validate() error {
    if u.Email == "" {
        return ErrInvalidEmail
    }
    if u.Name == "" {
        return ErrInvalidName
    }
    return nil
}
```

```go
// domain/transaction.go
package domain

import (
    "time"
    "github.com/google/uuid"
    "github.com/shopspring/decimal"
)

type TransactionType string

const (
    TransactionTypeIncome  TransactionType = "income"
    TransactionTypeExpense TransactionType = "expense"
)

type Transaction struct {
    ID          uuid.UUID       `json:"id" db:"id"`
    UserID      uuid.UUID       `json:"user_id" db:"user_id"`
    Amount      decimal.Decimal `json:"amount" db:"amount"`
    Category    string          `json:"category" db:"category"`
    Description string          `json:"description" db:"description"`
    Type        TransactionType `json:"type" db:"type"`
    Date        time.Time       `json:"date" db:"date"`
    Tags        []string        `json:"tags" db:"tags"`
    ReceiptURL  *string         `json:"receipt_url,omitempty" db:"receipt_url"`
    CreatedAt   time.Time       `json:"created_at" db:"created_at"`
    UpdatedAt   time.Time       `json:"updated_at" db:"updated_at"`
}

func (t *Transaction) Validate() error {
    if t.Amount.LessThanOrEqual(decimal.Zero) {
        return ErrInvalidAmount
    }
    if t.Category == "" {
        return ErrInvalidCategory
    }
    if t.Type != TransactionTypeIncome && t.Type != TransactionTypeExpense {
        return ErrInvalidTransactionType
    }
    return nil
}
```

## 🔐 Authentication

### JWT Implementation

```go
// pkg/auth/jwt.go
package auth

import (
    "errors"
    "time"
    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

type JWTManager struct {
    secretKey     string
    tokenDuration time.Duration
}

func NewJWTManager(secretKey string, duration time.Duration) *JWTManager {
    return &JWTManager{
        secretKey:     secretKey,
        tokenDuration: duration,
    }
}

func (j *JWTManager) Generate(userID, email string) (string, error) {
    claims := &Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(j.tokenDuration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(j.secretKey))
}

func (j *JWTManager) Verify(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return []byte(j.secretKey), nil
    })

    if err != nil {
        return nil, err
    }

    if !token.Valid {
        return nil, errors.New("invalid token")
    }

    claims, ok := token.Claims.(*Claims)
    if !ok {
        return nil, errors.New("invalid claims")
    }

    return claims, nil
}
```

### Middleware

```go
// internal/handler/middleware.go
package handler

import (
    "net/http"
    "strings"
    "github.com/gin-gonic/gin"
)

func AuthMiddleware(jwtManager *auth.JWTManager) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Authorization header required"})
            c.Abort()
            return
        }

        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid authorization header"})
            c.Abort()
            return
        }

        claims, err := jwtManager.Verify(parts[1])
        if err != nil {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid token"})
            c.Abort()
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("email", claims.Email)
        c.Next()
    }
}

func RateLimitMiddleware(limit int) gin.HandlerFunc {
    // Implement rate limiting logic
    return func(c *gin.Context) {
        // Rate limiting implementation
        c.Next()
    }
}

func CORSMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
        c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }

        c.Next()
    }
}
```

## ❌ Error Handling

### Standard Error Response

```go
// internal/utils/response.go
package utils

import (
    "github.com/gin-gonic/gin"
)

type ErrorResponse struct {
    Error   string `json:"error"`
    Message string `json:"message,omitempty"`
    Code    string `json:"code,omitempty"`
}

func SendError(c *gin.Context, status int, err error, message string) {
    c.JSON(status, ErrorResponse{
        Error:   err.Error(),
        Message: message,
    })
}

func SendSuccess(c *gin.Context, status int, data interface{}) {
    c.JSON(status, gin.H{
        "data": data,
    })
}
```

### Custom Errors

```go
// internal/domain/errors.go
package domain

import "errors"

var (
    ErrUserNotFound           = errors.New("user not found")
    ErrInvalidCredentials     = errors.New("invalid credentials")
    ErrInvalidEmail          = errors.New("invalid email")
    ErrInvalidAmount         = errors.New("amount must be positive")
    ErrInvalidCategory       = errors.New("category is required")
    ErrInvalidTransactionType = errors.New("invalid transaction type")
    ErrUnauthorized          = errors.New("unauthorized")
    ErrForbidden             = errors.New("forbidden")
)

type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

type ValidationErrors []ValidationError

func (v ValidationErrors) Error() string {
    return "validation failed"
}
```

## 📋 Go Best Practices

### Service Layer Pattern

```go
// internal/service/transaction_service.go
package service

import (
    "context"
    "github.com/google/uuid"
    "finance-dashboard/internal/domain"
    "finance-dashboard/internal/repository"
)

type TransactionService interface {
    Create(ctx context.Context, tx *domain.Transaction) error
    GetByID(ctx context.Context, id uuid.UUID) (*domain.Transaction, error)
    GetByUser(ctx context.Context, userID uuid.UUID, filter Filter) ([]*domain.Transaction, error)
    Update(ctx context.Context, id uuid.UUID, tx *domain.Transaction) error
    Delete(ctx context.Context, id uuid.UUID) error
    GetStatistics(ctx context.Context, userID uuid.UUID, period string) (*Statistics, error)
}

type transactionService struct {
    repo repository.TransactionRepository
    cache CacheService
}

func NewTransactionService(repo repository.TransactionRepository, cache CacheService) TransactionService {
    return &transactionService{
        repo:  repo,
        cache: cache,
    }
}

func (s *transactionService) Create(ctx context.Context, tx *domain.Transaction) error {
    // Validate
    if err := tx.Validate(); err != nil {
        return err
    }

    // Generate ID
    tx.ID = uuid.New()

    // Save to database
    if err := s.repo.Create(ctx, tx); err != nil {
        return err
    }

    // Invalidate cache
    s.cache.Delete(ctx, getCacheKey(tx.UserID))

    return nil
}

func (s *transactionService) GetStatistics(ctx context.Context, userID uuid.UUID, period string) (*Statistics, error) {
    // Check cache
    cacheKey := fmt.Sprintf("stats:%s:%s", userID, period)
    if cached, err := s.cache.Get(ctx, cacheKey); err == nil {
        return cached.(*Statistics), nil
    }

    // Calculate statistics
    transactions, err := s.repo.GetByUserAndPeriod(ctx, userID, period)
    if err != nil {
        return nil, err
    }

    stats := calculateStatistics(transactions)

    // Cache result
    s.cache.Set(ctx, cacheKey, stats, 15*time.Minute)

    return stats, nil
}
```

### Repository Pattern

```go
// internal/repository/postgres/transaction_repo.go
package postgres

import (
    "context"
    "database/sql"
    "github.com/google/uuid"
    "github.com/jmoiron/sqlx"
    "finance-dashboard/internal/domain"
)

type TransactionRepository struct {
    db *sqlx.DB
}

func NewTransactionRepository(db *sqlx.DB) *TransactionRepository {
    return &TransactionRepository{db: db}
}

func (r *TransactionRepository) Create(ctx context.Context, tx *domain.Transaction) error {
    query := `
        INSERT INTO transactions (
            id, user_id, amount, category, description,
            type, date, tags, receipt_url, created_at, updated_at
        ) VALUES (
            $1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11
        )
    `

    _, err := r.db.ExecContext(ctx, query,
        tx.ID, tx.UserID, tx.Amount, tx.Category, tx.Description,
        tx.Type, tx.Date, pq.Array(tx.Tags), tx.ReceiptURL,
        tx.CreatedAt, tx.UpdatedAt,
    )

    return err
}

func (r *TransactionRepository) GetByID(ctx context.Context, id uuid.UUID) (*domain.Transaction, error) {
    var tx domain.Transaction
    query := `
        SELECT id, user_id, amount, category, description,
               type, date, tags, receipt_url, created_at, updated_at
        FROM transactions
        WHERE id = $1
    `

    err := r.db.GetContext(ctx, &tx, query, id)
    if err == sql.ErrNoRows {
        return nil, domain.ErrTransactionNotFound
    }

    return &tx, err
}

func (r *TransactionRepository) GetByUser(ctx context.Context, userID uuid.UUID, filter Filter) ([]*domain.Transaction, error) {
    query := `
        SELECT id, user_id, amount, category, description,
               type, date, tags, receipt_url, created_at, updated_at
        FROM transactions
        WHERE user_id = $1
    `

    args := []interface{}{userID}
    argCount := 1

    if filter.Category != "" {
        argCount++
        query += fmt.Sprintf(" AND category = $%d", argCount)
        args = append(args, filter.Category)
    }

    if !filter.FromDate.IsZero() {
        argCount++
        query += fmt.Sprintf(" AND date >= $%d", argCount)
        args = append(args, filter.FromDate)
    }

    if !filter.ToDate.IsZero() {
        argCount++
        query += fmt.Sprintf(" AND date <= $%d", argCount)
        args = append(args, filter.ToDate)
    }

    // Add sorting
    query += fmt.Sprintf(" ORDER BY %s %s", filter.SortBy, filter.Order)

    // Add pagination
    query += fmt.Sprintf(" LIMIT %d OFFSET %d", filter.Limit, filter.Offset)

    var transactions []*domain.Transaction
    err := r.db.SelectContext(ctx, &transactions, query, args...)

    return transactions, err
}
```

### Handler Layer

```go
// internal/handler/transaction.go
package handler

import (
    "net/http"
    "strconv"
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
    "finance-dashboard/internal/service"
)

type TransactionHandler struct {
    service service.TransactionService
}

func NewTransactionHandler(service service.TransactionService) *TransactionHandler {
    return &TransactionHandler{service: service}
}

func (h *TransactionHandler) Create(c *gin.Context) {
    var req CreateTransactionRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        utils.SendError(c, http.StatusBadRequest, err, "Invalid request")
        return
    }

    userID, _ := c.Get("user_id")
    tx := &domain.Transaction{
        UserID:      uuid.MustParse(userID.(string)),
        Amount:      req.Amount,
        Category:    req.Category,
        Description: req.Description,
        Type:        req.Type,
        Date:        req.Date,
        Tags:        req.Tags,
    }

    if err := h.service.Create(c, tx); err != nil {
        utils.SendError(c, http.StatusInternalServerError, err, "Failed to create transaction")
        return
    }

    utils.SendSuccess(c, http.StatusCreated, tx)
}

func (h *TransactionHandler) GetList(c *gin.Context) {
    userID, _ := c.Get("user_id")

    // Parse query parameters
    filter := service.Filter{
        Category: c.Query("category"),
        Page:     parseInt(c.Query("page"), 1),
        Limit:    parseInt(c.Query("limit"), 20),
        SortBy:   c.DefaultQuery("sort", "date"),
        Order:    c.DefaultQuery("order", "desc"),
    }

    transactions, err := h.service.GetByUser(c, uuid.MustParse(userID.(string)), filter)
    if err != nil {
        utils.SendError(c, http.StatusInternalServerError, err, "Failed to fetch transactions")
        return
    }

    utils.SendSuccess(c, http.StatusOK, gin.H{
        "data":       transactions,
        "pagination": filter.ToPagination(),
    })
}

func (h *TransactionHandler) GetStatistics(c *gin.Context) {
    userID, _ := c.Get("user_id")
    period := c.DefaultQuery("period", "month")

    stats, err := h.service.GetStatistics(c, uuid.MustParse(userID.(string)), period)
    if err != nil {
        utils.SendError(c, http.StatusInternalServerError, err, "Failed to calculate statistics")
        return
    }

    utils.SendSuccess(c, http.StatusOK, stats)
}
```

## 🧪 Testing

### Unit Test Example

```go
// internal/service/transaction_service_test.go
package service_test

import (
    "context"
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "finance-dashboard/internal/domain"
    "finance-dashboard/internal/service"
)

type MockTransactionRepo struct {
    mock.Mock
}

func (m *MockTransactionRepo) Create(ctx context.Context, tx *domain.Transaction) error {
    args := m.Called(ctx, tx)
    return args.Error(0)
}

func TestTransactionService_Create(t *testing.T) {
    repo := new(MockTransactionRepo)
    cache := new(MockCache)
    svc := service.NewTransactionService(repo, cache)

    tx := &domain.Transaction{
        Amount:      decimal.NewFromFloat(100.50),
        Category:    "food",
        Description: "Lunch",
        Type:        domain.TransactionTypeExpense,
    }

    repo.On("Create", mock.Anything, mock.Anything).Return(nil)
    cache.On("Delete", mock.Anything, mock.Anything).Return(nil)

    err := svc.Create(context.Background(), tx)

    assert.NoError(t, err)
    assert.NotEqual(t, uuid.Nil, tx.ID)
    repo.AssertExpectations(t)
    cache.AssertExpectations(t)
}
```

### Integration Test Example

```go
// tests/integration/transaction_test.go
package integration_test

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestCreateTransaction(t *testing.T) {
    // Setup test database
    db := setupTestDB(t)
    defer db.Close()

    // Create test server
    router := setupRouter(db)

    // Create test user and get token
    token := createTestUser(t, router)

    // Test creating transaction
    payload := map[string]interface{}{
        "amount":      150.00,
        "category":    "food",
        "description": "Test transaction",
        "type":        "expense",
        "date":        "2024-01-15",
    }

    body, _ := json.Marshal(payload)
    req := httptest.NewRequest("POST", "/api/transactions", bytes.NewBuffer(body))
    req.Header.Set("Authorization", "Bearer "+token)
    req.Header.Set("Content-Type", "application/json")

    rec := httptest.NewRecorder()
    router.ServeHTTP(rec, req)

    assert.Equal(t, http.StatusCreated, rec.Code)

    var response map[string]interface{}
    json.Unmarshal(rec.Body.Bytes(), &response)

    assert.NotNil(t, response["id"])
    assert.Equal(t, 150.0, response["amount"])
    assert.Equal(t, "food", response["category"])
}
```

## 🚀 Running the Backend

### Configuration

```go
// internal/config/config.go
package config

import (
    "github.com/spf13/viper"
)

type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    JWT      JWTConfig
    Redis    RedisConfig
}

type ServerConfig struct {
    Port         string
    Mode         string // debug, release
    ReadTimeout  time.Duration
    WriteTimeout time.Duration
}

type DatabaseConfig struct {
    Host     string
    Port     int
    User     string
    Password string
    DBName   string
    SSLMode  string
}

func Load() (*Config, error) {
    viper.SetConfigFile(".env")
    viper.AutomaticEnv()

    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }

    var config Config
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }

    return &config, nil
}
```

### Main Entry Point

```go
// cmd/server/main.go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"
    "finance-dashboard/internal/config"
    "finance-dashboard/internal/handler"
    "finance-dashboard/internal/repository/postgres"
    "finance-dashboard/internal/service"
    "finance-dashboard/pkg/database"
)

func main() {
    // Load configuration
    cfg, err := config.Load()
    if err != nil {
        log.Fatal("Failed to load config:", err)
    }

    // Connect to database
    db, err := database.NewPostgresDB(cfg.Database)
    if err != nil {
        log.Fatal("Failed to connect to database:", err)
    }
    defer db.Close()

    // Run migrations
    if err := database.RunMigrations(db, "migrations"); err != nil {
        log.Fatal("Failed to run migrations:", err)
    }

    // Initialize repositories
    userRepo := postgres.NewUserRepository(db)
    txRepo := postgres.NewTransactionRepository(db)

    // Initialize services
    authService := service.NewAuthService(userRepo)
    txService := service.NewTransactionService(txRepo)

    // Initialize handlers
    authHandler := handler.NewAuthHandler(authService)
    txHandler := handler.NewTransactionHandler(txService)

    // Setup router
    router := setupRouter(cfg, authHandler, txHandler)

    // Start server
    srv := &http.Server{
        Addr:         ":" + cfg.Server.Port,
        Handler:      router,
        ReadTimeout:  cfg.Server.ReadTimeout,
        WriteTimeout: cfg.Server.WriteTimeout,
    }

    // Graceful shutdown
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal("Failed to start server:", err)
        }
    }()

    log.Printf("Server started on port %s", cfg.Server.Port)

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down server...")

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    log.Println("Server exited")
}

func setupRouter(cfg *config.Config, authHandler *handler.AuthHandler, txHandler *handler.TransactionHandler) *gin.Engine {
    gin.SetMode(cfg.Server.Mode)

    router := gin.New()
    router.Use(gin.Recovery())
    router.Use(handler.LoggerMiddleware())
    router.Use(handler.CORSMiddleware())

    // Health check
    router.GET("/health", func(c *gin.Context) {
        c.JSON(200, gin.H{"status": "healthy"})
    })

    api := router.Group("/api")
    {
        // Auth routes
        auth := api.Group("/auth")
        {
            auth.POST("/register", authHandler.Register)
            auth.POST("/login", authHandler.Login)
            auth.POST("/refresh", authHandler.Refresh)
        }

        // Protected routes
        protected := api.Group("/")
        protected.Use(handler.AuthMiddleware(jwtManager))
        {
            // Transaction routes
            protected.GET("/transactions", txHandler.GetList)
            protected.GET("/transactions/:id", txHandler.GetByID)
            protected.POST("/transactions", txHandler.Create)
            protected.PUT("/transactions/:id", txHandler.Update)
            protected.DELETE("/transactions/:id", txHandler.Delete)

            // Statistics routes
            protected.GET("/statistics/overview", txHandler.GetStatistics)
            protected.GET("/statistics/monthly", txHandler.GetMonthlyStats)
        }
    }

    return router
}
```

## 📋 Development Commands

```bash
# Install dependencies
go mod download

# Run tests
go test ./...

# Run with coverage
go test -cover ./...

# Run specific test
go test -run TestTransactionService ./internal/service

# Build application
go build -o bin/server cmd/server/main.go

# Run application
./bin/server

# Run with hot reload (using air)
air

# Generate mocks
mockery --all --output=mocks

# Run linter
golangci-lint run

# Format code
go fmt ./...
goimports -w .
```

## 🔗 Useful Links

- [Gin Web Framework](https://gin-gonic.com/)
- [GORM](https://gorm.io/)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Best Practices](https://dave.cheney.net/practical-go/presentations/qcon-china.html)