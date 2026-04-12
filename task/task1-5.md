# БД и миграции

Сегодня напимасал sql-код для миграций БД
Пока на инициализацию всей бд и её полный дроп

``` sql
-- init up
CREATE TABLE IF NOT EXISTS servers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    host VARCHAR(255) NOT NULL,
    port INT NOT NULL DEFAULT 22,
    username VARCHAR(255) NOT NULL,
    auth_type VARCHAR(20) NOT NULL CHECK (auth_type IN ('password', 'key')),
    password TEXT,
    private_key TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_seen TIMESTAMP WITH TIME ZONE,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_servers_name ON servers(name);
CREATE INDEX idx_servers_active ON servers(is_active);

CREATE TABLE IF NOT EXISTS server_status (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    server_id UUID NOT NULL REFERENCES servers(id) ON DELETE CASCADE,
    status VARCHAR(20) NOT NULL CHECK (status IN ('online', 'offline', 'syncing', 'error')),
    last_checked TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_server_status_server_id ON server_status(server_id);
CREATE INDEX idx_server_status_last_checked ON server_status(last_checked DESC);
CREATE INDEX idx_server_status_status ON server_status(status);

CREATE TABLE IF NOT EXISTS sync_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    server_id UUID NOT NULL REFERENCES servers(id) ON DELETE CASCADE,
    direction VARCHAR(10) NOT NULL CHECK (direction IN ('upload', 'download')),
    file_name VARCHAR(512) NOT NULL,
    remote_path TEXT NOT NULL,
    local_path TEXT NOT NULL,
    file_size BIGINT DEFAULT 0,
    bytes_transferred BIGINT DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'cancelled')),
    attempt_count INT DEFAULT 0,
    max_attempts INT DEFAULT 5,
    error_message TEXT,
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_sync_tasks_server_id ON sync_tasks(server_id);
CREATE INDEX idx_sync_tasks_status ON sync_tasks(status);
CREATE INDEX idx_sync_tasks_created_at ON sync_tasks(created_at DESC);
CREATE INDEX idx_sync_tasks_server_status ON sync_tasks(server_id, status);

CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_servers_updated_at
    BEFORE UPDATE ON servers
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_sync_tasks_updated_at
    BEFORE UPDATE ON sync_tasks
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

``` sql
-- init down
DROP TRIGGER IF EXISTS update_sync_tasks_updated_at ON sync_tasks;
DROP TRIGGER IF EXISTS update_servers_updated_at ON servers;
DROP FUNCTION IF EXISTS update_updated_at_column();

DROP TABLE IF EXISTS sync_tasks;
DROP TABLE IF EXISTS server_status;
DROP TABLE IF EXISTS servers;
```

Понял, что необходимо обновить конфиг для БД - добавил туда максимальное количество подключений (простаивающих и в принципе открытых).
Открытые = активные + простаивающие 
Также добавил в конфиг lifetime максимальное время жизни подключения и максимальное время жизни простаивающего подключения
``` go
type DatabaseCfg struct {
	Host     string
	Port     string
	User     string
	Password string
	DBName   string
	SSLMode  string
	// Пул соединений
	MaxOpenConns    int
	MaxIdleConns    int
	ConnMaxLifetime time.Duration
	ConnMaxIdleTime time.Duration
}
```

добавил в конфиг dsn - функцию-конструктор строки для подключения к БД

Перешёл к реализации storage, где написал код-обёртку для подключения к Postgres. Использовал pgxpool

``` go
type DB struct {
    Pool   *pgxpool.Pool
    config *config.DatabaseCfg
}
```

Во многом прои поключении я руководствовался атрибутами пула соединений pgxpool и их документацией 

``` go
	p                     *puddle.Pool[*connResource]
	config                *Config
	beforeConnect         func(context.Context, *pgx.ConnConfig) error
	afterConnect          func(context.Context, *pgx.Conn) error
	prepareConn           func(context.Context, *pgx.Conn) (bool, error)
	afterRelease          func(*pgx.Conn) bool
	beforeClose           func(*pgx.Conn)
	shouldPing            func(context.Context, ShouldPingParams) bool
	minConns              int32
	minIdleConns          int32
	maxConns              int32
	maxConnLifetime       time.Duration
	maxConnLifetimeJitter time.Duration
	maxConnIdleTime       time.Duration
	healthCheckPeriod     time.Duration
	pingTimeout           time.Duration
```
Отсюда как раз позаимствовал MaxConns и IdleConns 

NewDB() - функция конструктор БД

``` go

type DB struct {
	Pool   *pgxpool.Pool
	config *config.DatabaseCfg
}

func NewDB(ctx context.Context, cfg *config.DatabaseCfg) (*DB, error) {
	poolConfig, err := pgxpool.ParseConfig(cfg.DSN())
	if err != nil {
		return nil, fmt.Errorf("failed to parse database config: %w", err)
	}

	poolConfig.MaxConns = int32(cfg.MaxOpenConns)
	poolConfig.MinConns = int32(cfg.MaxIdleConns)
	poolConfig.MaxConnLifetime = cfg.ConnMaxLifetime
	poolConfig.MaxConnIdleTime = cfg.ConnMaxIdleTime

	connectCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
	defer cancel()

	pool, err := pgxpool.NewWithConfig(connectCtx, poolConfig)
	if err != nil {
		return nil, fmt.Errorf("failed to create connection pool: %w", err)
	}

	if err := pool.Ping(connectCtx); err != nil {
		pool.Close()
		return nil, fmt.Errorf("failed to ping database: %w", err)
	}

	logger.Get().Info().
		Str("host", cfg.Host).
		Str("port", cfg.Port).
		Str("database", cfg.DBName).
		Msg("Connected to PostgreSQL")

	return &DB{
		Pool:   pool,
		config: cfg,
	}, nil
}

func (db *DB) Close() {
	if db.Pool != nil {
		db.Pool.Close()
		logger.Get().Info().Msg("PostgreSQL connection pool closed")
	}
}

func (db *DB) Ping(ctx context.Context) error {
	return db.Pool.Ping(ctx)
}

```

С учётом миграций и интеграции с БД немного правлю доменные сущности: в частности server и добавляю новую - ServerStatus в файле domain/server.go
А в domain/task.go формирую доменные сущности для задач синхронизации

``` go
package domain

import (
	"time"

	"github.com/google/uuid"
)

type Server struct {
	ID         uuid.UUID
	Name       string
	Host       string
	Port       int
	Username   string
	AuthType   string
	Password   *string
	PrivateKey *string
	CreatedAt  time.Time
	UpdatedAt  time.Time
	LastSeen   *time.Time
	IsActive   bool
}

type ServerStatus struct {
	ID           uuid.UUID
	ServerID     uuid.UUID
	Status       string //  TODO "online" или "offline" или "syncing" или "error"
	LastChecked  time.Time
	ErrorMessage *string
	CreatedAt    time.Time
}

```

``` go
package domain

import (
	"time"

	"github.com/google/uuid"
)

type SyncDirection string

const (
	DirectionUpload   SyncDirection = "upload"
	DirectionDownload SyncDirection = "download"
)

type SyncStatus string

const (
	StatusPending    SyncStatus = "pending"
	StatusProcessing SyncStatus = "processing"
	StatusCompleted  SyncStatus = "completed"
	StatusFailed     SyncStatus = "failed"
	StatusCancelled  SyncStatus = "cancelled"
)

type SyncTask struct {
	ID               uuid.UUID
	ServerID         uuid.UUID
	Direction        SyncDirection
	FileName         string
	RemotePath       string
	LocalPath        string
	FileSize         int64
	BytesTransferred int64
	Status           SyncStatus
	AttemptCount     int
	MaxAttempts      int
	ErrorMessage     *string
	StartedAt        *time.Time
	CompletedAt      *time.Time
	CreatedAt        time.Time
	UpdatedAt        time.Time
}

func (t *SyncTask) Progress() float64 {
	if t.FileSize == 0 {
		return 0
	}
	return float64(t.BytesTransferred) / float64(t.FileSize) * 100
}

```

Далее написал в repository контракты для бд - они могут пригодится, если вдруг нужно будет переходить к каким-то более продвинутым нежеди pg решениям

``` go 
// интерфейсы для сервера

type ServerRepository interface {
	Create(ctx context.Context, server *domain.Server) error
	GetByID(ctx context.Context, id uuid.UUID) (*domain.Server, error)
	GetByName(ctx context.Context, name string) (*domain.Server, error)
	List(ctx context.Context, activeOnly bool) ([]*domain.Server, error)
	Update(ctx context.Context, server *domain.Server) error
	Delete(ctx context.Context, id uuid.UUID) error
	UpdateLastSeen(ctx context.Context, id uuid.UUID) error
}

type ServerStatusRepository interface {
	Create(ctx context.Context, status *domain.ServerStatus) error
	GetLatestForServer(ctx context.Context, serverID uuid.UUID) (*domain.ServerStatus, error)
	ListByServer(ctx context.Context, serverID uuid.UUID, limit int) ([]*domain.ServerStatus, error)
}
```

``` go
// интерфейсы для таски

type TaskRepository interface {
	Create(ctx context.Context, task *domain.SyncTask) error
	GetByID(ctx context.Context, id uuid.UUID) (*domain.SyncTask, error)
	ListByServer(ctx context.Context, serverID uuid.UUID, status *domain.SyncStatus, limit int) ([]*domain.SyncTask, error)
	ListPending(ctx context.Context, limit int) ([]*domain.SyncTask, error)
	Update(ctx context.Context, task *domain.SyncTask) error
	UpdateStatus(ctx context.Context, id uuid.UUID, status domain.SyncStatus, errMsg *string) error
	UpdateProgress(ctx context.Context, id uuid.UUID, bytesTransferred int64) error
	IncrementAttempt(ctx context.Context, id uuid.UUID) error
	Delete(ctx context.Context, id uuid.UUID) error
	DeleteCompletedOlderThan(ctx context.Context, olderThan time.Duration) (int64, error)
}

```

Настало время покрыть эти интерфейсы реализациями для Postgres в storage (это уже фактически как бизнес-логика)
Поделил реализации на 3 таблицы и 3 соответствующих файла

```go 
//server_repo.go
package postgres

import (
	"context"

	"github.com/Meedoeed/ssh-sync-automation/internal/domain"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/google/uuid"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

type ServerRepo struct {
	db *pgxpool.Pool
}

func NewServerRepo(db *pgxpool.Pool) *ServerRepo {
	return &ServerRepo{db: db}
}

func (r *ServerRepo) Create(ctx context.Context, server *domain.Server) error {
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
		server.Password,
		server.PrivateKey,
		server.IsActive,
	).Scan(&server.CreatedAt, &server.UpdatedAt)

	if err != nil {
		logger.Get().Error().
			Err(err).
			Str("server_name", server.Name).
			Msg("Failed to create server")
		return err
	}

	logger.Get().Info().
		Str("server_id", server.ID.String()).
		Str("server_name", server.Name).
		Msg("Server created")

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
	err := r.db.QueryRow(ctx, query, id).Scan(
		&server.ID,
		&server.Name,
		&server.Host,
		&server.Port,
		&server.Username,
		&server.AuthType,
		&server.Password,
		&server.PrivateKey,
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
	err := r.db.QueryRow(ctx, query, name).Scan(
		&server.ID,
		&server.Name,
		&server.Host,
		&server.Port,
		&server.Username,
		&server.AuthType,
		&server.Password,
		&server.PrivateKey,
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

	return &server, nil
}

func (r *ServerRepo) List(ctx context.Context, activeOnly bool) ([]*domain.Server, error) {
	query := `
		SELECT id, name, host, port, username, auth_type, password, private_key,
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
			&server.Password,
			&server.PrivateKey,
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

func (r *ServerRepo) Update(ctx context.Context, server *domain.Server) error {
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
		server.Password,
		server.PrivateKey,
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

func (r *ServerRepo) Delete(ctx context.Context, id uuid.UUID) error {
	query := `DELETE FROM servers WHERE id = $1`

	_, err := r.db.Exec(ctx, query, id)
	if err != nil {
		logger.Get().Error().
			Err(err).
			Str("server_id", id.String()).
			Msg("Failed to delete server")
		return err
	}

	logger.Get().Info().
		Str("server_id", id.String()).
		Msg("Server deleted")

	return nil
}

func (r *ServerRepo) UpdateLastSeen(ctx context.Context, id uuid.UUID) error {
	query := `UPDATE servers SET last_seen = NOW() WHERE id = $1`

	_, err := r.db.Exec(ctx, query, id)
	return err
}
```

```go 
//server_status_repo.go
package postgres

import (
	"context"
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/domain"
	"github.com/google/uuid"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

type ServerStatusRepo struct {
	db *pgxpool.Pool
}

func NewServerStatusRepo(db *pgxpool.Pool) *ServerStatusRepo {
	return &ServerStatusRepo{db: db}
}

func (r *ServerStatusRepo) Create(ctx context.Context, status *domain.ServerStatus) error {
	query := `
		INSERT INTO server_status (id, server_id, status, last_checked, error_message)
		VALUES ($1, $2, $3, $4, $5)
		RETURNING created_at
	`

	if status.ID == uuid.Nil {
		status.ID = uuid.New()
	}

	if status.LastChecked.IsZero() {
		status.LastChecked = time.Now()
	}

	return r.db.QueryRow(ctx, query,
		status.ID,
		status.ServerID,
		status.Status,
		status.LastChecked,
		status.ErrorMessage,
	).Scan(&status.CreatedAt)
}

func (r *ServerStatusRepo) GetLatestForServer(ctx context.Context, serverID uuid.UUID) (*domain.ServerStatus, error) {
	query := `
		SELECT id, server_id, status, last_checked, error_message, created_at
		FROM server_status
		WHERE server_id = $1
		ORDER BY last_checked DESC
		LIMIT 1
	`

	var status domain.ServerStatus
	err := r.db.QueryRow(ctx, query, serverID).Scan(
		&status.ID,
		&status.ServerID,
		&status.Status,
		&status.LastChecked,
		&status.ErrorMessage,
		&status.CreatedAt,
	)

	if err != nil {
		if err == pgx.ErrNoRows {
			return nil, nil
		}
		return nil, err
	}

	return &status, nil
}

func (r *ServerStatusRepo) ListByServer(ctx context.Context, serverID uuid.UUID, limit int) ([]*domain.ServerStatus, error) {
	query := `
		SELECT id, server_id, status, last_checked, error_message, created_at
		FROM server_status
		WHERE server_id = $1
		ORDER BY last_checked DESC
		LIMIT $2
	`

	rows, err := r.db.Query(ctx, query, serverID, limit)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var statuses []*domain.ServerStatus
	for rows.Next() {
		var status domain.ServerStatus
		err := rows.Scan(
			&status.ID,
			&status.ServerID,
			&status.Status,
			&status.LastChecked,
			&status.ErrorMessage,
			&status.CreatedAt,
		)
		if err != nil {
			return nil, err
		}
		statuses = append(statuses, &status)
	}

	return statuses, nil
}

```

```go 
//task_repo.go
package postgres

import (
	"context"
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/domain"
	"github.com/google/uuid"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

type TaskRepo struct {
	db *pgxpool.Pool
}

func NewTaskRepo(db *pgxpool.Pool) *TaskRepo {
	return &TaskRepo{db: db}
}

func (r *TaskRepo) Create(ctx context.Context, task *domain.SyncTask) error {
	query := `
		INSERT INTO sync_tasks (
			id, server_id, direction, file_name, remote_path, local_path,
			file_size, status, max_attempts
		)
		VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
		RETURNING created_at, updated_at
	`

	if task.ID == uuid.Nil {
		task.ID = uuid.New()
	}

	if task.MaxAttempts == 0 {
		task.MaxAttempts = 5
	}

	return r.db.QueryRow(ctx, query,
		task.ID,
		task.ServerID,
		task.Direction,
		task.FileName,
		task.RemotePath,
		task.LocalPath,
		task.FileSize,
		task.Status,
		task.MaxAttempts,
	).Scan(&task.CreatedAt, &task.UpdatedAt)
}

func (r *TaskRepo) GetByID(ctx context.Context, id uuid.UUID) (*domain.SyncTask, error) {
	query := `
		SELECT id, server_id, direction, file_name, remote_path, local_path,
		       file_size, bytes_transferred, status, attempt_count, max_attempts,
		       error_message, started_at, completed_at, created_at, updated_at
		FROM sync_tasks
		WHERE id = $1
	`

	var task domain.SyncTask
	err := r.db.QueryRow(ctx, query, id).Scan(
		&task.ID,
		&task.ServerID,
		&task.Direction,
		&task.FileName,
		&task.RemotePath,
		&task.LocalPath,
		&task.FileSize,
		&task.BytesTransferred,
		&task.Status,
		&task.AttemptCount,
		&task.MaxAttempts,
		&task.ErrorMessage,
		&task.StartedAt,
		&task.CompletedAt,
		&task.CreatedAt,
		&task.UpdatedAt,
	)

	if err != nil {
		if err == pgx.ErrNoRows {
			return nil, nil
		}
		return nil, err
	}

	return &task, nil
}

func (r *TaskRepo) ListByServer(ctx context.Context, serverID uuid.UUID, status *domain.SyncStatus, limit int) ([]*domain.SyncTask, error) {
	query := `
		SELECT id, server_id, direction, file_name, remote_path, local_path,
		       file_size, bytes_transferred, status, attempt_count, max_attempts,
		       error_message, started_at, completed_at, created_at, updated_at
		FROM sync_tasks
		WHERE server_id = $1
	`
	args := []any{serverID}

	if status != nil {
		query += " AND status = $2"
		args = append(args, *status)
	}

	query += " ORDER BY created_at DESC LIMIT $"
	if status != nil {
		query += "3"
	} else {
		query += "2"
	}
	args = append(args, limit)

	rows, err := r.db.Query(ctx, query, args...)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var tasks []*domain.SyncTask
	for rows.Next() {
		var task domain.SyncTask
		err := rows.Scan(
			&task.ID,
			&task.ServerID,
			&task.Direction,
			&task.FileName,
			&task.RemotePath,
			&task.LocalPath,
			&task.FileSize,
			&task.BytesTransferred,
			&task.Status,
			&task.AttemptCount,
			&task.MaxAttempts,
			&task.ErrorMessage,
			&task.StartedAt,
			&task.CompletedAt,
			&task.CreatedAt,
			&task.UpdatedAt,
		)
		if err != nil {
			return nil, err
		}
		tasks = append(tasks, &task)
	}

	return tasks, nil
}

func (r *TaskRepo) ListPending(ctx context.Context, limit int) ([]*domain.SyncTask, error) {
	query := `
		SELECT id, server_id, direction, file_name, remote_path, local_path,
		       file_size, bytes_transferred, status, attempt_count, max_attempts,
		       error_message, started_at, completed_at, created_at, updated_at
		FROM sync_tasks
		WHERE status = $1 AND attempt_count < max_attempts
		ORDER BY created_at
		LIMIT $2
		FOR UPDATE SKIP LOCKED
	`

	rows, err := r.db.Query(ctx, query, domain.StatusPending, limit)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var tasks []*domain.SyncTask
	for rows.Next() {
		var task domain.SyncTask
		err := rows.Scan(
			&task.ID,
			&task.ServerID,
			&task.Direction,
			&task.FileName,
			&task.RemotePath,
			&task.LocalPath,
			&task.FileSize,
			&task.BytesTransferred,
			&task.Status,
			&task.AttemptCount,
			&task.MaxAttempts,
			&task.ErrorMessage,
			&task.StartedAt,
			&task.CompletedAt,
			&task.CreatedAt,
			&task.UpdatedAt,
		)
		if err != nil {
			return nil, err
		}
		tasks = append(tasks, &task)
	}

	return tasks, nil
}

func (r *TaskRepo) Update(ctx context.Context, task *domain.SyncTask) error {
	query := `
		UPDATE sync_tasks
		SET file_size = $2, bytes_transferred = $3, status = $4,
		    attempt_count = $5, error_message = $6, started_at = $7, completed_at = $8
		WHERE id = $1
	`

	_, err := r.db.Exec(ctx, query,
		task.ID,
		task.FileSize,
		task.BytesTransferred,
		task.Status,
		task.AttemptCount,
		task.ErrorMessage,
		task.StartedAt,
		task.CompletedAt,
	)

	return err
}

func (r *TaskRepo) UpdateStatus(ctx context.Context, id uuid.UUID, status domain.SyncStatus, errMsg *string) error {
	query := `
		UPDATE sync_tasks
		SET status = $2, error_message = $3,
		    completed_at = CASE WHEN $2 IN ('completed', 'failed', 'cancelled') THEN NOW() ELSE completed_at END
		WHERE id = $1
	`

	_, err := r.db.Exec(ctx, query, id, status, errMsg)
	return err
}

func (r *TaskRepo) UpdateProgress(ctx context.Context, id uuid.UUID, bytesTransferred int64) error {
	query := `
		UPDATE sync_tasks
		SET bytes_transferred = $2
		WHERE id = $1
	`

	_, err := r.db.Exec(ctx, query, id, bytesTransferred)
	return err
}

func (r *TaskRepo) IncrementAttempt(ctx context.Context, id uuid.UUID) error {
	query := `
		UPDATE sync_tasks
		SET attempt_count = attempt_count + 1,
		    started_at = COALESCE(started_at, NOW())
		WHERE id = $1
	`

	_, err := r.db.Exec(ctx, query, id)
	return err
}

func (r *TaskRepo) Delete(ctx context.Context, id uuid.UUID) error {
	query := `DELETE FROM sync_tasks WHERE id = $1`

	_, err := r.db.Exec(ctx, query, id)
	return err
}

func (r *TaskRepo) DeleteCompletedOlderThan(ctx context.Context, olderThan time.Duration) (int64, error) {
	query := `
		DELETE FROM sync_tasks
		WHERE status IN ('completed', 'failed', 'cancelled')
		  AND completed_at < NOW() - $1
	`

	result, err := r.db.Exec(ctx, query, olderThan)
	if err != nil {
		return 0, err
	}

	return result.RowsAffected(), nil
}
```

После этого я обновил main.go
обновил readiness_check и закрыл в нём TODO
обновил http.go

```go
//main.go
package main

import (
	"context"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/config"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/Meedoeed/ssh-sync-automation/internal/server"
	"github.com/Meedoeed/ssh-sync-automation/internal/storage/postgres"
)

func main() {
	cfg := config.Load()
	logger.Init(cfg.Log.Level, true)
	logger.Get().Info().Msg("Starting SSH-SYNC-AUTOMATION service")

	ctx := context.Background()
	db, err := postgres.NewDB(ctx, &cfg.Database)
	if err != nil {
		logger.Get().Fatal().Err(err).Msg("Failed to connect to database")
	}
	defer db.Close()

	httpServer := server.NewHTTP(cfg, db)

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
```go
//healt.go
package handler

import (
	"net/http"

	"github.com/Meedoeed/ssh-sync-automation/internal/storage/postgres"
	"github.com/labstack/echo/v4"
)

type HealthHandler struct {
	db *postgres.DB
}

func NewHealthHandler(db *postgres.DB) *HealthHandler {
	return &HealthHandler{db: db}
}

func (h *HealthHandler) Liveness(c echo.Context) error {
	return c.JSON(http.StatusOK, map[string]string{"status": "alive"})
}

func (h *HealthHandler) Readiness(c echo.Context) error {
	if h.db == nil {
		return c.JSON(http.StatusServiceUnavailable, map[string]string{
			"status": "not ready",
			"reason": "database not initialized",
		})
	}

	if err := h.db.Ping(c.Request().Context()); err != nil {
		return c.JSON(http.StatusServiceUnavailable, map[string]string{
			"status": "not ready",
			"reason": "database unavailable: " + err.Error(),
		})
	}

	return c.JSON(http.StatusOK, map[string]string{"status": "ready"})
}
```

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
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	myMiddleware "github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/middleware"
	"github.com/Meedoeed/ssh-sync-automation/internal/storage/postgres"
)

type HTTPServer struct {
	echo   *echo.Echo
	config *config.Config
	db     *postgres.DB
}

func NewHTTP(cfg *config.Config, db *postgres.DB) *HTTPServer {
	e := echo.New()
	e.HideBanner = true
	e.HidePort = true

	e.Use(middleware.Recover())
	e.Use(middleware.RequestID())
	e.Use(myMiddleware.EchoLogger())

	healthHandler := handler.NewHealthHandler(db)
	handler.RegisterRoutes(e, healthHandler)

	return &HTTPServer{
		echo:   e,
		config: cfg,
		db:     db,
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

В целом код работает, но есть существенный минус по миграциям:
Мне не нравится, что до сих пор они применяются вручную и поэтому я немноого переписал storage.go для автоматического их применения.
Миграции у меня в embedFS 
Для миграций использовал:

go get github.com/golang-migrate/migrate/v4
go get github.com/golang-migrate/migrate/v4/database/postgres
go get github.com/golang-migrate/migrate/v4/source/file

```go
// migrate.go
package postgres

import (
	"embed"
	"errors"
	"fmt"

	"github.com/Meedoeed/ssh-sync-automation/internal/config"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/golang-migrate/migrate/v4"
	_ "github.com/golang-migrate/migrate/v4/database/postgres"
	"github.com/golang-migrate/migrate/v4/source/iofs"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

func RunMigrations(cfg *config.DatabaseCfg) error {
	sourceDriver, err := iofs.New(migrationsFS, "migrations")
	if err != nil {
		return fmt.Errorf("failed to create migration source: %w", err)
	}

	dbURL := fmt.Sprintf(
		"postgres://%s:%s@%s:%s/%s?sslmode=%s",
		cfg.User, cfg.Password, cfg.Host, cfg.Port, cfg.DBName, cfg.SSLMode,
	)

	m, err := migrate.NewWithSourceInstance("iofs", sourceDriver, dbURL)
	if err != nil {
		return fmt.Errorf("failed to create migrator: %w", err)
	}
	defer m.Close()

	if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
		return fmt.Errorf("failed to apply migrations: %w", err)
	}

	version, dirty, err := m.Version()
	if err != nil && !errors.Is(err, migrate.ErrNilVersion) {
		return fmt.Errorf("failed to get migration version: %w", err)
	}

	if !dirty {
		logger.Get().Info().
			Uint("version", uint(version)).
			Msg("Database migrations applied successfully")
	} else {
		logger.Get().Warn().
			Uint("version", uint(version)).
			Msg("Database is in dirty state, manual intervention may be required")
	}

	return nil
}
```
Теперь storage.go NewDB() начинается с запуска миграций:
``` go
if err := RunMigrations(cfg); err != nil {
    return nil, fmt.Errorf("failed to run migrations: %w", err)
}
```

На этом этапе хотел протестить и понял, что словил баг ещё в самом начале, на этапе парсера конфигов. 
Я просто не гружу .env в переменное окружение, исправил с помощью импорта пакета "github.com/joho/godotenv"
Теперь в main гружу через него .env:
```go
// cmd/server/main.go
package main

import (
    "log"
    "github.com/joho/godotenv"  // ← добавить импорт
    "github.com/Meedoeed/ssh-sync-automation/internal/config"
)

func main() {
    if err := godotenv.Load(); err != nil {
        log.Println("No .env file found")
    }
    
    cfg := config.Load()
    // ...
}
```

Сверху ещё один баг - посмотрел документацию и увидел, что по стандартному DSN pg подключаться не умеет
``` go
# Example URL
//	postgres://jack:secret@pg.example.com:5432/mydb?sslmode=verify-ca&pool_max_conns=10&pool_max_conn_lifetime=1h30m
``` 
вырезка из документации. Поэтому я дописал функцию URL в конфиг для коннекта к БД
``` go
// config.go
func (c *DatabaseCfg) URL() string {
	return fmt.Sprintf("postgres://%s:%s@%s:%s/%s?sslmode=%s",
		c.User,
		c.Password,
		c.Host,
		c.Port,
		c.DBName,
		c.SSLMode,
	)
}
```
С этих пор dsn не нужен и я полностью удалил его метод