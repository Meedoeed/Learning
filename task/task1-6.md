# Планы

Сегодня планирую закрыть несколько моментов и провести небольшой рефакторинг касаемо того, что сейчас пароли ложатся в БД в не шифрованном виде

План на сегодня:
1) хэширование паролей перед сохранением в БД
2) закрою service слой приложения - это будет нечто вроде валидирующей обёртки вчерашних репозиториев и их реализаций

# Хэширование паролей

Хэшировать буду AES-ом. Для этого написал небольшую утилиту

```go
package encryption

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/base64"
	"io"
)

type Encryptor struct {
	key []byte 
}

func NewEncryptor(encryptionKey string) *Encryptor {
	return &Encryptor{key: []byte(encryptionKey)}
}

func (e *Encryptor) Encrypt(plaintext string) (string, error) {
	block, err := aes.NewCipher(e.key)
	if err != nil {
		return "", err
	}

	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return "", err
	}

	nonce := make([]byte, gcm.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return "", err
	}

	ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
	return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func (e *Encryptor) Decrypt(ciphertext string) (string, error) {
	data, err := base64.StdEncoding.DecodeString(ciphertext)
	if err != nil {
		return "", err
	}

	block, err := aes.NewCipher(e.key)
	if err != nil {
		return "", err
	}

	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return "", err
	}

	nonceSize := gcm.NonceSize()
	nonce, ciphertextBytes := data[:nonceSize], data[nonceSize:]

	plaintext, err := gcm.Open(nil, nonce, ciphertextBytes, nil)
	if err != nil {
		return "", err
	}

	return string(plaintext), nil
}
```

Тут простая структура шифровальщика с функциями шифровки и дешифровки. 
Ключ для шифрования буду брать из .env

Добавил новый конфиг:

```go
type EncryptionCfg struct {
	Key string
}
// обновил
type Config struct {
	Server     ServerCfg
	Database   DatabaseCfg
	Sync       SyncCfg
	Log        LogCfg
	Encryption EncryptionCfg // добавил
}
```

Обновил с учётом шифрования server_repo

```go
// internal/storage/postgres/server_repo.go
package postgres

import (
	"context"

	"github.com/Meedoeed/ssh-sync-automation/internal/domain"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/encryption"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/google/uuid"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

type ServerRepo struct {
	db        *pgxpool.Pool
	encryptor *encryption.Encryptor // Добавляем
}

func NewServerRepo(db *pgxpool.Pool, encryptor *encryption.Encryptor) *ServerRepo {
	return &ServerRepo{
		db:        db,
		encryptor: encryptor,
	}
}

func (r *ServerRepo) Create(ctx context.Context, server *domain.Server) error {
	// Шифруем пароль и ключ перед сохранением
	var encryptedPassword, encryptedPrivateKey *string

	if server.Password != nil && *server.Password != "" {
		encrypted, err := r.encryptor.Encrypt(*server.Password)
		if err != nil {
			logger.Get().Error().Err(err).Msg("Failed to encrypt password")
			return err
		}
		encryptedPassword = &encrypted
	}

	if server.PrivateKey != nil && *server.PrivateKey != "" {
		encrypted, err := r.encryptor.Encrypt(*server.PrivateKey)
		if err != nil {
			logger.Get().Error().Err(err).Msg("Failed to encrypt private key")
			return err
		}
		encryptedPrivateKey = &encrypted
	}

	query := `
        INSERT INTO servers (id, name, host, port, username, auth_type, password, private_key, is_active)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING created_at, updated_at
    `

	if server.ID == uuid.Nil {
		server.ID = uuid.New()
	}

	err := r.db.QueryRow(ctx, query,
		server.ID,
		server.Name,
		server.Host,
		server.Port,
		server.Username,
		server.AuthType,
		encryptedPassword,  
		encryptedPrivateKey, 
		server.IsActive,
	).Scan(&server.CreatedAt, &server.UpdatedAt)

	if err != nil {
		logger.Get().Error().
			Err(err).
			Str("server_name", server.Name).
			Msg("Failed to create server")
		return err
	}

	server.Password = nil
	server.PrivateKey = nil

	return nil
}

func (r *ServerRepo) GetByID(ctx context.Context, id uuid.UUID) (*domain.Server, error) {
	query := `
        SELECT id, name, host, port, username, auth_type, password, private_key,
               created_at, updated_at, last_seen, is_active
        FROM servers
        WHERE id = $1
    `

	var server domain.Server
	var encryptedPassword, encryptedPrivateKey *string

	err := r.db.QueryRow(ctx, query, id).Scan(
		&server.ID,
		&server.Name,
		&server.Host,
		&server.Port,
		&server.Username,
		&server.AuthType,
		&encryptedPassword,
		&encryptedPrivateKey,
		&server.CreatedAt,
		&server.UpdatedAt,
		&server.LastSeen,
		&server.IsActive,
	)

	if err != nil {
		if err == pgx.ErrNoRows {
			return nil, nil
		}
		return nil, err
	}

	if encryptedPassword != nil && *encryptedPassword != "" {
		decrypted, err := r.encryptor.Decrypt(*encryptedPassword)
		if err != nil {
			logger.Get().Error().Err(err).Msg("Failed to decrypt password")
			return nil, err
		}
		server.Password = &decrypted
	}

	if encryptedPrivateKey != nil && *encryptedPrivateKey != "" {
		decrypted, err := r.encryptor.Decrypt(*encryptedPrivateKey)
		if err != nil {
			logger.Get().Error().Err(err).Msg("Failed to decrypt private key")
			return nil, err
		}
		server.PrivateKey = &decrypted
	}

	return &server, nil
}

func (r *ServerRepo) GetByName(ctx context.Context, name string) (*domain.Server, error) {
	query := `
        SELECT id, name, host, port, username, auth_type, password, private_key,
               created_at, updated_at, last_seen, is_active
        FROM servers
        WHERE name = $1
    `

	var server domain.Server
	var encryptedPassword, encryptedPrivateKey *string

	err := r.db.QueryRow(ctx, query, name).Scan(
		&server.ID,
		&server.Name,
		&server.Host,
		&server.Port,
		&server.Username,
		&server.AuthType,
		&encryptedPassword,
		&encryptedPrivateKey,
		&server.CreatedAt,
		&server.UpdatedAt,
		&server.LastSeen,
		&server.IsActive,
	)

	if err != nil {
		if err == pgx.ErrNoRows {
			return nil, nil
		}
		return nil, err
	}

	if encryptedPassword != nil && *encryptedPassword != "" {
		decrypted, err := r.encryptor.Decrypt(*encryptedPassword)
		if err != nil {
			return nil, err
		}
		server.Password = &decrypted
	}

	if encryptedPrivateKey != nil && *encryptedPrivateKey != "" {
		decrypted, err := r.encryptor.Decrypt(*encryptedPrivateKey)
		if err != nil {
			return nil, err
		}
		server.PrivateKey = &decrypted
	}

	return &server, nil
}

func (r *ServerRepo) Update(ctx context.Context, server *domain.Server) error {
	var encryptedPassword, encryptedPrivateKey *string

	if server.Password != nil && *server.Password != "" {
		encrypted, err := r.encryptor.Encrypt(*server.Password)
		if err != nil {
			return err
		}
		encryptedPassword = &encrypted
	}

	if server.PrivateKey != nil && *server.PrivateKey != "" {
		encrypted, err := r.encryptor.Encrypt(*server.PrivateKey)
		if err != nil {
			return err
		}
		encryptedPrivateKey = &encrypted
	}

	query := `
        UPDATE servers
        SET name = $2, host = $3, port = $4, username = $5,
            auth_type = $6, password = $7, private_key = $8, is_active = $9
        WHERE id = $1
    `

	_, err := r.db.Exec(ctx, query,
		server.ID,
		server.Name,
		server.Host,
		server.Port,
		server.Username,
		server.AuthType,
		encryptedPassword,
		encryptedPrivateKey,
		server.IsActive,
	)

	if err != nil {
		logger.Get().Error().
			Err(err).
			Str("server_id", server.ID.String()).
			Msg("Failed to update server")
		return err
	}

	return nil
}

func (r *ServerRepo) List(ctx context.Context, activeOnly bool) ([]*domain.Server, error) {
	query := `
        SELECT id, name, host, port, username, auth_type, 
               created_at, updated_at, last_seen, is_active
        FROM servers
    `
	if activeOnly {
		query += " WHERE is_active = true"
	}
	query += " ORDER BY name"

	rows, err := r.db.Query(ctx, query)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var servers []*domain.Server
	for rows.Next() {
		var server domain.Server
		err := rows.Scan(
			&server.ID,
			&server.Name,
			&server.Host,
			&server.Port,
			&server.Username,
			&server.AuthType,
			&server.CreatedAt,
			&server.UpdatedAt,
			&server.LastSeen,
			&server.IsActive,
		)
		if err != nil {
			return nil, err
		}
		servers = append(servers, &server)
	}

	return servers, nil
}

func (r *ServerRepo) Delete(ctx context.Context, id uuid.UUID) error {
	query := `DELETE FROM servers WHERE id = $1`
	_, err := r.db.Exec(ctx, query, id)
	return err
}

func (r *ServerRepo) UpdateLastSeen(ctx context.Context, id uuid.UUID) error {
	query := `UPDATE servers SET last_seen = NOW() WHERE id = $1`
	_, err := r.db.Exec(ctx, query, id)
	return err
}
```

Также обновил handler и main.go для поддержки encryptor

```go
//main.go
// cmd/server/main.go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/config"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/encryption"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/Meedoeed/ssh-sync-automation/internal/server"
	"github.com/Meedoeed/ssh-sync-automation/internal/storage/postgres"
	"github.com/joho/godotenv"
)

func main() {
	if err := godotenv.Load(); err != nil {
		log.Println("No .env file found")
	}

	cfg := config.Load()

	if cfg.Encryption.Key == "" {
		log.Fatal("ENCRYPTION_KEY is required")
	}
	if len(cfg.Encryption.Key) != 32 {
		log.Fatal("ENCRYPTION_KEY must be 32 bytes for AES-256")
	}

	logger.Init(cfg.Log.Level, true)
	logger.Get().Info().Msg("Starting SSH-SYNC-AUTOMATION service")

	encryptor := encryption.NewEncryptor(cfg.Encryption.Key)
	logger.Get().Info().Msg("Encryption initialized")

	ctx := context.Background()
	db, err := postgres.NewDB(ctx, &cfg.Database)
	if err != nil {
		logger.Get().Fatal().Err(err).Msg("Failed to connect to database")
	}
	defer db.Close()

	httpServer := server.NewHTTP(cfg, db, encryptor)

	go func() {
		if err := httpServer.Start(); err != nil {
			logger.Get().Fatal().Err(err).Msg("Failed to start HTTP server")
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := httpServer.Shutdown(shutdownCtx); err != nil {
		logger.Get().Error().Err(err).Msg("HTTP server shutdown error")
	}

	logger.Get().Info().Msg("Service stopped")
}
// http.go
package server

import (
	"context"
	"fmt"

	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"

	"github.com/Meedoeed/ssh-sync-automation/internal/config"
	"github.com/Meedoeed/ssh-sync-automation/internal/handler"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/encryption"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	myMiddleware "github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/middleware"
	"github.com/Meedoeed/ssh-sync-automation/internal/storage/postgres"
)

type HTTPServer struct {
	echo     *echo.Echo
	config   *config.Config
	db       *postgres.DB
	encrytor *encryption.Encryptor
}

func NewHTTP(cfg *config.Config, db *postgres.DB, encrytor *encryption.Encryptor) *HTTPServer {
	e := echo.New()
	e.HideBanner = true
	e.HidePort = true

	e.Use(middleware.Recover())
	e.Use(middleware.RequestID())
	e.Use(myMiddleware.EchoLogger())

	healthHandler := handler.NewHealthHandler(db)
	handler.RegisterRoutes(e, healthHandler)

	return &HTTPServer{
		echo:     e,
		config:   cfg,
		db:       db,
		encrytor: encrytor,
	}
}

func (s *HTTPServer) Start() error {
	addr := fmt.Sprintf(":%s", s.config.Server.Port)
	logger.Get().Info().Msgf("Starting HTTP server on %s", addr)
	return s.echo.Start(addr)
}

func (s *HTTPServer) Shutdown(ctx context.Context) error {
	logger.Get().Info().Msg("Shutting down HTTP server...")
	return s.echo.Shutdown(ctx)
}
```
Добавил ключ в .env
На этом рефакторинг закончен

# Service layer

Реализовал сценарии использования для серверов и для тасок. Это прослойка между rep слоем и слоем handler по большей части выполняющая задачи валидации данных.

```go
//server_service.go
// internal/service/server_service.go
package service

import (
	"context"
	"fmt"
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/domain"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/Meedoeed/ssh-sync-automation/internal/repository"
	"github.com/google/uuid"
)

type ServerService struct {
	serverRepo repository.ServerRepository
	statusRepo repository.ServerStatusRepository
}

func NewServerService(
	serverRepo repository.ServerRepository,
	statusRepo repository.ServerStatusRepository,
) *ServerService {
	return &ServerService{
		serverRepo: serverRepo,
		statusRepo: statusRepo,
	}
}

func (s *ServerService) CreateServer(ctx context.Context, server *domain.Server) error {
	// Валидация
	if err := s.validateServer(server); err != nil {
		return fmt.Errorf("validation failed: %w", err)
	}

	existing, err := s.serverRepo.GetByName(ctx, server.Name)
	if err != nil {
		return fmt.Errorf("failed to check server name: %w", err)
	}
	if existing != nil {
		return fmt.Errorf("server with name %s already exists", server.Name)
	}

	if err := s.serverRepo.Create(ctx, server); err != nil {
		return fmt.Errorf("failed to create server: %w", err)
	}

	logger.Get().Info().
		Str("server_id", server.ID.String()).
		Str("server_name", server.Name).
		Msg("Server created successfully")

	return nil
}

func (s *ServerService) GetServer(ctx context.Context, id uuid.UUID) (*domain.Server, error) {
	server, err := s.serverRepo.GetByID(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("failed to get server: %w", err)
	}
	if server == nil {
		return nil, fmt.Errorf("server not found")
	}
	return server, nil
}

func (s *ServerService) ListServers(ctx context.Context, activeOnly bool) ([]*domain.Server, error) {
	servers, err := s.serverRepo.List(ctx, activeOnly)
	if err != nil {
		return nil, fmt.Errorf("failed to list servers: %w", err)
	}
	return servers, nil
}

func (s *ServerService) UpdateServer(ctx context.Context, server *domain.Server) error {
	existing, err := s.serverRepo.GetByID(ctx, server.ID)
	if err != nil {
		return fmt.Errorf("failed to get server: %w", err)
	}
	if existing == nil {
		return fmt.Errorf("server not found")
	}

	if err := s.validateServer(server); err != nil {
		return fmt.Errorf("validation failed: %w", err)
	}

	if existing.Name != server.Name {
		nameExists, err := s.serverRepo.GetByName(ctx, server.Name)
		if err != nil {
			return fmt.Errorf("failed to check server name: %w", err)
		}
		if nameExists != nil {
			return fmt.Errorf("server with name %s already exists", server.Name)
		}
	}

	if err := s.serverRepo.Update(ctx, server); err != nil {
		return fmt.Errorf("failed to update server: %w", err)
	}

	logger.Get().Info().
		Str("server_id", server.ID.String()).
		Str("server_name", server.Name).
		Msg("Server updated successfully")

	return nil
}

func (s *ServerService) DeleteServer(ctx context.Context, id uuid.UUID) error {
	existing, err := s.serverRepo.GetByID(ctx, id)
	if err != nil {
		return fmt.Errorf("failed to get server: %w", err)
	}
	if existing == nil {
		return fmt.Errorf("server not found")
	}

	// Удаление
	if err := s.serverRepo.Delete(ctx, id); err != nil {
		return fmt.Errorf("failed to delete server: %w", err)
	}

	logger.Get().Info().
		Str("server_id", id.String()).
		Str("server_name", existing.Name).
		Msg("Server deleted successfully")

	return nil
}

func (s *ServerService) UpdateServerStatus(ctx context.Context, serverID uuid.UUID, status string, errMsg *string) error {
	serverStatus := &domain.ServerStatus{
		ID:           uuid.New(),
		ServerID:     serverID,
		Status:       status,
		LastChecked:  time.Now(),
		ErrorMessage: errMsg,
	}

	if err := s.statusRepo.Create(ctx, serverStatus); err != nil {
		return fmt.Errorf("failed to update server status: %w", err)
	}

	if status == "online" {
		if err := s.serverRepo.UpdateLastSeen(ctx, serverID); err != nil {
			logger.Get().Warn().
				Err(err).
				Str("server_id", serverID.String()).
				Msg("Failed to update last_seen")
		}
	}

	return nil
}

func (s *ServerService) GetServerStatus(ctx context.Context, serverID uuid.UUID) (*domain.ServerStatus, error) {
	status, err := s.statusRepo.GetLatestForServer(ctx, serverID)
	if err != nil {
		return nil, fmt.Errorf("failed to get server status: %w", err)
	}
	return status, nil
}

func (s *ServerService) validateServer(server *domain.Server) error {
	if server.Name == "" {
		return fmt.Errorf("server name is required")
	}
	if server.Host == "" {
		return fmt.Errorf("host is required")
	}
	if server.Port < 1 || server.Port > 65535 {
		return fmt.Errorf("invalid port: %d", server.Port)
	}
	if server.Username == "" {
		return fmt.Errorf("username is required")
	}

	switch server.AuthType {
	case "password":
		if server.Password == nil || *server.Password == "" {
			return fmt.Errorf("password is required for password auth")
		}
	case "key":
		if server.PrivateKey == nil || *server.PrivateKey == "" {
			return fmt.Errorf("private key is required for key auth")
		}
	default:
		return fmt.Errorf("invalid auth type: %s, must be 'password' or 'key'", server.AuthType)
	}

	return nil
}

//task_service.go
// internal/service/task_service.go
package service

import (
	"context"
	"fmt"
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/domain"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/Meedoeed/ssh-sync-automation/internal/repository"
	"github.com/google/uuid"
)

type TaskService struct {
	taskRepo repository.TaskRepository
}

func NewTaskService(taskRepo repository.TaskRepository) *TaskService {
	return &TaskService{
		taskRepo: taskRepo,
	}
}

func (s *TaskService) CreateTask(ctx context.Context, task *domain.SyncTask) error {
	if err := s.validateTask(task); err != nil {
		return fmt.Errorf("validation failed: %w", err)
	}

	if task.ID == uuid.Nil {
		task.ID = uuid.New()
	}
	if task.Status == "" {
		task.Status = domain.StatusPending
	}
	if task.MaxAttempts == 0 {
		task.MaxAttempts = 5
	}

	if err := s.taskRepo.Create(ctx, task); err != nil {
		return fmt.Errorf("failed to create task: %w", err)
	}

	logger.Get().Info().
		Str("task_id", task.ID.String()).
		Str("server_id", task.ServerID.String()).
		Str("direction", string(task.Direction)).
		Str("file_name", task.FileName).
		Msg("Task created successfully")

	return nil
}

func (s *TaskService) GetTask(ctx context.Context, id uuid.UUID) (*domain.SyncTask, error) {
	task, err := s.taskRepo.GetByID(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("failed to get task: %w", err)
	}
	if task == nil {
		return nil, fmt.Errorf("task not found")
	}
	return task, nil
}

func (s *TaskService) ListServerTasks(ctx context.Context, serverID uuid.UUID, status *domain.SyncStatus, limit int) ([]*domain.SyncTask, error) {
	if limit <= 0 || limit > 100 {
		limit = 50
	}

	tasks, err := s.taskRepo.ListByServer(ctx, serverID, status, limit)
	if err != nil {
		return nil, fmt.Errorf("failed to list tasks: %w", err)
	}
	return tasks, nil
}

func (s *TaskService) GetPendingTasks(ctx context.Context, limit int) ([]*domain.SyncTask, error) {
	if limit <= 0 || limit > 100 {
		limit = 10
	}

	tasks, err := s.taskRepo.ListPending(ctx, limit)
	if err != nil {
		return nil, fmt.Errorf("failed to get pending tasks: %w", err)
	}
	return tasks, nil
}

func (s *TaskService) StartTask(ctx context.Context, id uuid.UUID) error {
	task, err := s.taskRepo.GetByID(ctx, id)
	if err != nil {
		return fmt.Errorf("failed to get task: %w", err)
	}
	if task == nil {
		return fmt.Errorf("task not found")
	}

	if task.Status != domain.StatusPending {
		return fmt.Errorf("task status is %s, cannot start", task.Status)
	}

	now := time.Now()
	task.Status = domain.StatusProcessing
	task.StartedAt = &now

	if err := s.taskRepo.Update(ctx, task); err != nil {
		return fmt.Errorf("failed to start task: %w", err)
	}

	logger.Get().Info().
		Str("task_id", id.String()).
		Msg("Task started")

	return nil
}

func (s *TaskService) UpdateTaskProgress(ctx context.Context, id uuid.UUID, bytesTransferred int64) error {
	if err := s.taskRepo.UpdateProgress(ctx, id, bytesTransferred); err != nil {
		return fmt.Errorf("failed to update progress: %w", err)
	}
	return nil
}

func (s *TaskService) CompleteTask(ctx context.Context, id uuid.UUID) error {
	task, err := s.taskRepo.GetByID(ctx, id)
	if err != nil {
		return fmt.Errorf("failed to get task: %w", err)
	}
	if task == nil {
		return fmt.Errorf("task not found")
	}

	now := time.Now()
	task.Status = domain.StatusCompleted
	task.CompletedAt = &now
	task.ErrorMessage = nil

	if err := s.taskRepo.Update(ctx, task); err != nil {
		return fmt.Errorf("failed to complete task: %w", err)
	}

	logger.Get().Info().
		Str("task_id", id.String()).
		Int64("total_bytes", task.BytesTransferred).
		Msg("Task completed successfully")

	return nil
}

func (s *TaskService) FailTask(ctx context.Context, id uuid.UUID, errMsg string) error {
	task, err := s.taskRepo.GetByID(ctx, id)
	if err != nil {
		return fmt.Errorf("failed to get task: %w", err)
	}
	if task == nil {
		return fmt.Errorf("task not found")
	}

	task.AttemptCount++

	if task.AttemptCount >= task.MaxAttempts {
		now := time.Now()
		task.Status = domain.StatusFailed
		task.CompletedAt = &now
		task.ErrorMessage = &errMsg

		logger.Get().Warn().
			Str("task_id", id.String()).
			Int("attempts", task.AttemptCount).
			Int("max_attempts", task.MaxAttempts).
			Str("error", errMsg).
			Msg("Task failed permanently")
	} else {
		task.Status = domain.StatusPending
		task.ErrorMessage = &errMsg

		logger.Get().Info().
			Str("task_id", id.String()).
			Int("attempt", task.AttemptCount).
			Int("max_attempts", task.MaxAttempts).
			Msg("Task will be retried")
	}

	if err := s.taskRepo.Update(ctx, task); err != nil {
		return fmt.Errorf("failed to fail task: %w", err)
	}

	return nil
}

func (s *TaskService) CancelTask(ctx context.Context, id uuid.UUID) error {
	task, err := s.taskRepo.GetByID(ctx, id)
	if err != nil {
		return fmt.Errorf("failed to get task: %w", err)
	}
	if task == nil {
		return fmt.Errorf("task not found")
	}

	if task.Status == domain.StatusCompleted {
		return fmt.Errorf("cannot cancel completed task")
	}

	now := time.Now()
	task.Status = domain.StatusCancelled
	task.CompletedAt = &now

	if err := s.taskRepo.Update(ctx, task); err != nil {
		return fmt.Errorf("failed to cancel task: %w", err)
	}

	logger.Get().Info().
		Str("task_id", id.String()).
		Msg("Task cancelled")

	return nil
}

func (s *TaskService) CleanupOldTasks(ctx context.Context, olderThan time.Duration) (int64, error) {
	count, err := s.taskRepo.DeleteCompletedOlderThan(ctx, olderThan)
	if err != nil {
		return 0, fmt.Errorf("failed to cleanup tasks: %w", err)
	}

	if count > 0 {
		logger.Get().Info().
			Int64("deleted_count", count).
			Dur("older_than", olderThan).
			Msg("Cleaned up old tasks")
	}

	return count, nil
}

func (s *TaskService) validateTask(task *domain.SyncTask) error {
	if task.ServerID == uuid.Nil {
		return fmt.Errorf("server_id is required")
	}
	if task.Direction != domain.DirectionUpload && task.Direction != domain.DirectionDownload {
		return fmt.Errorf("invalid direction: %s", task.Direction)
	}
	if task.FileName == "" {
		return fmt.Errorf("file_name is required")
	}
	if task.RemotePath == "" {
		return fmt.Errorf("remote_path is required")
	}
	if task.LocalPath == "" {
		return fmt.Errorf("local_path is required")
	}
	return nil
}

```

Помимо бизнес-логики валидации у сервиса есть более глобальные цели заключающиеся в синхронизации удалённых репозиториев и для реализации этих сценариев использования я написал sync_service.go - сценарии использования синхронизации

```go
// internal/service/sync_service.go
package service

import (
	"context"
	"fmt"
	"os"
	"path/filepath"

	"github.com/Meedoeed/ssh-sync-automation/internal/domain"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/Meedoeed/ssh-sync-automation/internal/repository"
	"github.com/google/uuid"
)

type SyncService struct {
	serverRepo   repository.ServerRepository
	taskRepo     repository.TaskRepository
	statusRepo   repository.ServerStatusRepository
	baseLocalDir string
}

func NewSyncService(
	serverRepo repository.ServerRepository,
	taskRepo repository.TaskRepository,
	statusRepo repository.ServerStatusRepository,
	baseLocalDir string,
) *SyncService {
	return &SyncService{
		serverRepo:   serverRepo,
		taskRepo:     taskRepo,
		statusRepo:   statusRepo,
		baseLocalDir: baseLocalDir,
	}
}

func (s *SyncService) SyncServer(ctx context.Context, serverID uuid.UUID, sshClient infrastructure.SSHClientInterface) error {
	server, err := s.serverRepo.GetByID(ctx, serverID)
	if err != nil {
		return fmt.Errorf("failed to get server: %w", err)
	}
	if server == nil {
		return fmt.Errorf("server not found")
	}

	logger.Get().Info().
		Str("server_id", serverID.String()).
		Str("server_name", server.Name).
		Msg("Starting sync for server")


	if err := s.syncDownload(ctx, server, sshClient); err != nil {
		logger.Get().Error().
			Err(err).
			Str("server_id", serverID.String()).
			Msg("Download sync failed")
	}

	if err := s.syncUpload(ctx, server, sshClient); err != nil {
		logger.Get().Error().
			Err(err).
			Str("server_id", serverID.String()).
			Msg("Upload sync failed")
		return err
	}

	logger.Get().Info().
		Str("server_id", serverID.String()).
		Str("server_name", server.Name).
		Msg("Sync completed for server")

	return nil
}

func (s *SyncService) syncDownload(ctx context.Context, server *domain.Server, sshClient infrastructure.SSHClientInterface) error {
	remotePath := "done/"
	localPath := filepath.Join(s.baseLocalDir, "done", server.Name)

	if err := os.MkdirAll(localPath, 0755); err != nil {
		return fmt.Errorf("failed to create local directory: %w", err)
	}

	files, err := sshClient.ListFiles(remotePath)
	if err != nil {
		errMsg := err.Error()
		_ = s.updateServerStatus(ctx, server.ID, "error", &errMsg)
		return fmt.Errorf("failed to list files: %w", err)
	}

	if len(files) == 0 {
		logger.Get().Debug().
			Str("server", server.Name).
			Msg("No files to download")
		return nil
	}

	for _, file := range files {
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
		}

		remoteFilePath := filepath.Join(remotePath, file)
		localFilePath := filepath.Join(localPath, file)

		// Создаем задачу
		task := &domain.SyncTask{
			ID:         uuid.New(),
			ServerID:   server.ID,
			Direction:  domain.DirectionDownload,
			FileName:   file,
			RemotePath: remoteFilePath,
			LocalPath:  localFilePath,
			Status:     domain.StatusPending,
		}

		if err := s.taskRepo.Create(ctx, task); err != nil {
			logger.Get().Error().
				Err(err).
				Str("file", file).
				Msg("Failed to create download task")
			continue
		}

		if err := sshClient.DownloadWithRetry(remoteFilePath, localFilePath); err != nil {
			logger.Get().Error().
				Err(err).
				Str("file", file).
				Msg("Failed to download file")

			errMsg := err.Error()
			if err := s.taskRepo.UpdateStatus(ctx, task.ID, domain.StatusFailed, &errMsg); err != nil {
				logger.Get().Warn().Err(err).Msg("Failed to update task status")
			}
			continue
		}

		if err := sshClient.DeleteFile(remoteFilePath); err != nil {
			logger.Get().Warn().
				Err(err).
				Str("file", file).
				Msg("Failed to delete remote file after download")
		}

		if err := s.taskRepo.UpdateStatus(ctx, task.ID, domain.StatusCompleted, nil); err != nil {
			logger.Get().Warn().Err(err).Msg("Failed to update task status")
		}

		logger.Get().Info().
			Str("server", server.Name).
			Str("file", file).
			Msg("File downloaded successfully")
	}

	return nil
}

func (s *SyncService) syncUpload(ctx context.Context, server *domain.Server, sshClient infrastructure.SSHClientInterface) error {
	localPath := filepath.Join(s.baseLocalDir, "tasks", server.Name)
	remotePath := "tasks/"

	if _, err := os.Stat(localPath); os.IsNotExist(err) {
		logger.Get().Debug().
			Str("server", server.Name).
			Msg("No tasks directory, nothing to upload")
		return nil
	}

	files, err := os.ReadDir(localPath)
	if err != nil {
		return fmt.Errorf("failed to read local directory: %w", err)
	}

	if len(files) == 0 {
		logger.Get().Debug().
			Str("server", server.Name).
			Msg("No files to upload")
		return nil
	}

	for _, file := range files {
		if file.IsDir() {
			continue
		}

		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
		}

		localFilePath := filepath.Join(localPath, file.Name())
		remoteFilePath := filepath.Join(remotePath, file.Name())

		task := &domain.SyncTask{
			ID:         uuid.New(),
			ServerID:   server.ID,
			Direction:  domain.DirectionUpload,
			FileName:   file.Name(),
			RemotePath: remoteFilePath,
			LocalPath:  localFilePath,
			Status:     domain.StatusPending,
		}

		if err := s.taskRepo.Create(ctx, task); err != nil {
			logger.Get().Error().
				Err(err).
				Str("file", file.Name()).
				Msg("Failed to create upload task")
			continue
		}

		if err := sshClient.UploadWithRetry(localFilePath, remoteFilePath); err != nil {
			logger.Get().Error().
				Err(err).
				Str("file", file.Name()).
				Msg("Failed to upload file")

			errMsg := err.Error()
			if err := s.taskRepo.UpdateStatus(ctx, task.ID, domain.StatusFailed, &errMsg); err != nil {
				logger.Get().Warn().Err(err).Msg("Failed to update task status")
			}
			continue
		}

		if err := os.Remove(localFilePath); err != nil {
			logger.Get().Warn().
				Err(err).
				Str("file", file.Name()).
				Msg("Failed to delete local file after upload")
		}

		if err := s.taskRepo.UpdateStatus(ctx, task.ID, domain.StatusCompleted, nil); err != nil {
			logger.Get().Warn().Err(err).Msg("Failed to update task status")
		}

		logger.Get().Info().
			Str("server", server.Name).
			Str("file", file.Name()).
			Msg("File uploaded successfully")
	}

	return nil
}

func (s *SyncService) updateServerStatus(ctx context.Context, serverID uuid.UUID, status string, errMsg *string) error {
	serverStatus := &domain.ServerStatus{
		ID:           uuid.New(),
		ServerID:     serverID,
		Status:       status,
		ErrorMessage: errMsg,
	}

	return s.statusRepo.Create(ctx, serverStatus)
}
```

Обновил http.go и main.go

```go
//http.go
package server

import (
	"context"
	"fmt"

	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"

	"github.com/Meedoeed/ssh-sync-automation/internal/config"
	"github.com/Meedoeed/ssh-sync-automation/internal/handler"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/encryption"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	myMiddleware "github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/middleware"
	"github.com/Meedoeed/ssh-sync-automation/internal/service"
	"github.com/Meedoeed/ssh-sync-automation/internal/storage/postgres"
)

type HTTPServer struct {
	echo          *echo.Echo
	config        *config.Config
	db            *postgres.DB
	encrytor      *encryption.Encryptor
	serverService *service.ServerService
	taskService   *service.TaskService
	syncService   *service.SyncService
}

func NewHTTP(
	cfg *config.Config,
	db *postgres.DB,
	encrytor *encryption.Encryptor,
	serverService *service.ServerService,
	taskService *service.TaskService,
	syncService *service.SyncService,
) *HTTPServer {
	e := echo.New()
	e.HideBanner = true
	e.HidePort = true

	e.Use(middleware.Recover())
	e.Use(middleware.RequestID())
	e.Use(myMiddleware.EchoLogger())

	healthHandler := handler.NewHealthHandler(db)
	handler.RegisterRoutes(e, healthHandler)

	return &HTTPServer{
		echo:          e,
		config:        cfg,
		db:            db,
		encrytor:      encrytor,
		serverService: serverService,
		taskService:   taskService,
		syncService:   syncService,
	}
}

func (s *HTTPServer) Start() error {
	addr := fmt.Sprintf(":%s", s.config.Server.Port)
	logger.Get().Info().Msgf("Starting HTTP server on %s", addr)
	return s.echo.Start(addr)
}

func (s *HTTPServer) Shutdown(ctx context.Context) error {
	logger.Get().Info().Msg("Shutting down HTTP server...")
	return s.echo.Shutdown(ctx)
}
//main.go
// cmd/server/main.go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/config"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/encryption"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/Meedoeed/ssh-sync-automation/internal/server"
	"github.com/Meedoeed/ssh-sync-automation/internal/service"
	"github.com/Meedoeed/ssh-sync-automation/internal/storage/postgres"
	"github.com/joho/godotenv"
)

func main() {
	if err := godotenv.Load(); err != nil {
		log.Println("No .env file found")
	}

	cfg := config.Load()

	if cfg.Encryption.Key == "" {
		log.Fatal("ENCRYPTION_KEY is required")
	}
	if len(cfg.Encryption.Key) != 32 {
		log.Fatal("ENCRYPTION_KEY must be 32 bytes for AES-256")
	}

	logger.Init(cfg.Log.Level, true)
	logger.Get().Info().Msg("Starting SSH-SYNC-AUTOMATION service")

	encryptor := encryption.NewEncryptor(cfg.Encryption.Key)
	logger.Get().Info().Msg("Encryption initialized")

	ctx := context.Background()
	db, err := postgres.NewDB(ctx, &cfg.Database)
	if err != nil {
		logger.Get().Fatal().Err(err).Msg("Failed to connect to database")
	}
	defer db.Close()

	serverRepo := postgres.NewServerRepo(db.Pool, encryptor)
	taskRepo := postgres.NewTaskRepo(db.Pool)
	statusRepo := postgres.NewServerStatusRepo(db.Pool)

	serverService := service.NewServerService(serverRepo, statusRepo)
	taskService := service.NewTaskService(taskRepo)
	syncService := service.NewSyncService(serverRepo, taskRepo, statusRepo, "./data")

	httpServer := server.NewHTTP(cfg, db, encryptor, serverService, taskService, syncService)

	go func() {
		if err := httpServer.Start(); err != nil {
			logger.Get().Fatal().Err(err).Msg("Failed to start HTTP server")
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := httpServer.Shutdown(shutdownCtx); err != nil {
		logger.Get().Error().Err(err).Msg("HTTP server shutdown error")
	}

	logger.Get().Info().Msg("Service stopped")
}
```