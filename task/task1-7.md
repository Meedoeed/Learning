# Задачи

Текущие задачи по реализации:
1) Worker для одного сервиса 
2) WorkerPool как планировщик работы воркеров

# Worker

Методы:
NewWorker() - конструктор
Start() - запуск воркера
Stop() - остановка воркера
GetState() - текущее состояние для метрик
GetServerID() - ID сервера привязанного к воркеру
Run() - основной цикл ворера
runSync() - выполняет 1 цикл синхронизации
updateStatus() - обновляет статус сервера в БД с помощью сервиса, созданного ранее

``` go
package worker

import (
	"context"
	"fmt"
	"sync"
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/domain"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/Meedoeed/ssh-sync-automation/internal/repository"
	"github.com/Meedoeed/ssh-sync-automation/internal/service"
	"github.com/google/uuid"
)

type State string

const (
	StateStopped State = "stopped"
	StateRunning State = "running"
	StateError   State = "error"
)

type Worker struct {
	mu          sync.RWMutex
	id          uuid.UUID
	serverID    uuid.UUID
	serverName  string
	state       State
	sshClient   infrastructure.SSHClientInterface
	syncService *service.SyncService
	statusRepo  repository.ServerStatusRepository

	interval   time.Duration
	cancelFunc context.CancelFunc
	wg         sync.WaitGroup

	lastSync   time.Time
	syncCount  int64
	errorCount int64
	lastError  string
}

type Config struct {
	Interval    time.Duration
	SyncService *service.SyncService
	StatusRepo  repository.ServerStatusRepository
}

func NewWorker(
	serverID uuid.UUID,
	serverName string,
	sshClient infrastructure.SSHClientInterface,
	cfg *Config,
) *Worker {
	return &Worker{
		id:          uuid.New(),
		serverID:    serverID,
		serverName:  serverName,
		state:       StateStopped,
		sshClient:   sshClient,
		syncService: cfg.SyncService,
		statusRepo:  cfg.StatusRepo,
		interval:    cfg.Interval,
	}
}

func (w *Worker) Start(ctx context.Context) error {
	w.mu.Lock()
	defer w.mu.Unlock()

	if w.state == StateRunning {
		return fmt.Errorf("worker already running")
	}

	ctx, cancel := context.WithCancel(ctx)
	w.cancelFunc = cancel
	w.state = StateRunning

	w.wg.Add(1)
	go w.run(ctx)

	logger.Get().Info().
		Str("worker_id", w.id.String()).
		Str("server_id", w.serverID.String()).
		Str("server_name", w.serverName).
		Dur("interval", w.interval).
		Msg("Worker started")

	return nil
}

func (w *Worker) Stop() error {
	w.mu.Lock()
	defer w.mu.Unlock()

	if w.state != StateRunning {
		return fmt.Errorf("worker is not running")
	}

	w.state = StateStopped
	if w.cancelFunc != nil {
		w.cancelFunc()
	}
	w.wg.Wait()

	if err := w.sshClient.Close(); err != nil {
		logger.Get().Warn().
			Err(err).
			Str("server_name", w.serverName).
			Msg("Error closing SSH connection")
	}

	logger.Get().Info().
		Str("worker_id", w.id.String()).
		Str("server_name", w.serverName).
		Msg("Worker stopped")

	return nil
}

func (w *Worker) GetState() State {
	w.mu.RLock()
	defer w.mu.RUnlock()
	return w.state
}

func (w *Worker) GetStats() (lastSync time.Time, syncCount, errorCount int64, lastError string) {
	w.mu.RLock()
	defer w.mu.RUnlock()
	return w.lastSync, w.syncCount, w.errorCount, w.lastError
}

func (w *Worker) GetServerID() uuid.UUID {
	return w.serverID
}

func (w *Worker) GetServerName() string {
	return w.serverName
}

func (w *Worker) run(ctx context.Context) {
	defer w.wg.Done()

	ticker := time.NewTicker(w.interval)
	defer ticker.Stop()

	w.runSync(ctx)

	for {
		select {
		case <-ctx.Done():
			w.updateStatus(ctx, "offline", nil)
			return

		case <-ticker.C:
			w.mu.RLock()
			state := w.state
			w.mu.RUnlock()

			if state != StateRunning {
				continue
			}

			w.runSync(ctx)
		}
	}
}

func (w *Worker) runSync(ctx context.Context) {
	startTime := time.Now()

	logger.Get().Debug().
		Str("server_name", w.serverName).
		Msg("Starting sync cycle")

	w.updateStatus(ctx, "syncing", nil)

	if !w.sshClient.IsConnected() {
		logger.Get().Warn().
			Str("server_name", w.serverName).
			Msg("SSH connection lost")

		w.updateStatus(ctx, "offline", nil)

		w.mu.Lock()
		w.errorCount++
		w.lastError = "SSH connection lost"
		w.mu.Unlock()
		return
	}

	err := w.syncService.SyncServer(ctx, w.serverID, w.sshClient)

	w.mu.Lock()
	w.lastSync = time.Now()
	if err != nil {
		w.errorCount++
		w.lastError = err.Error()
		w.updateStatus(ctx, "error", &w.lastError)
		logger.Get().Error().
			Err(err).
			Str("server_name", w.serverName).
			Dur("duration", time.Since(startTime)).
			Msg("Sync cycle failed")
	} else {
		w.syncCount++
		w.lastError = ""
		w.updateStatus(ctx, "online", nil)
		logger.Get().Info().
			Str("server_name", w.serverName).
			Dur("duration", time.Since(startTime)).
			Msg("Sync cycle completed successfully")
	}
	w.mu.Unlock()
}

func (w *Worker) updateStatus(ctx context.Context, status string, errMsg *string) {
	serverStatus := &domain.ServerStatus{
		ID:           uuid.New(),
		ServerID:     w.serverID,
		Status:       status,
		LastChecked:  time.Now(),
		ErrorMessage: errMsg,
	}

	if err := w.statusRepo.Create(ctx, serverStatus); err != nil {
		logger.Get().Warn().
			Err(err).
			Str("server_name", w.serverName).
			Msg("Failed to update server status")
	}
}
```
# WorkerPool

Методы:
1) Start() - запускает работу/ создаёт воркеров для активных серверов
2) StopAll() - метод для graceful shutdown
3) addWorker() - добавление нового воркера для сервера
4) removeWorker() - удаление воркера с остановкой
5) GetWorker() - геттер для определенного воркера
6) ListWorkers() - возвращает срез всех воркеров
7) Для статистики и метрик сделал GetStats() - собирает статистику по воркерам.
``` go 
package worker

import (
	"context"
	"fmt"
	"sync"
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/config"
	"github.com/Meedoeed/ssh-sync-automation/internal/domain"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure"
	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/Meedoeed/ssh-sync-automation/internal/repository"
	"github.com/Meedoeed/ssh-sync-automation/internal/service"
	"github.com/google/uuid"
)

type Pool struct {
	mu          sync.RWMutex
	workers     map[uuid.UUID]*Worker
	serverRepo  repository.ServerRepository
	statusRepo  repository.ServerStatusRepository
	syncService *service.SyncService
	syncCfg     *config.SyncCfg
	interval    time.Duration
}

type PoolConfig struct {
	Interval    time.Duration
	SyncCfg     *config.SyncCfg
	ServerRepo  repository.ServerRepository
	StatusRepo  repository.ServerStatusRepository
	SyncService *service.SyncService
}

func NewPool(cfg *PoolConfig) *Pool {
	return &Pool{
		workers:     make(map[uuid.UUID]*Worker),
		serverRepo:  cfg.ServerRepo,
		statusRepo:  cfg.StatusRepo,
		syncService: cfg.SyncService,
		syncCfg:     cfg.SyncCfg,
		interval:    cfg.Interval,
	}
}

func (p *Pool) Start(ctx context.Context) error {
	logger.Get().Info().Msg("Starting worker pool")

	servers, err := p.serverRepo.List(ctx, true)
	if err != nil {
		return fmt.Errorf("failed to list active servers: %w", err)
	}

	logger.Get().Info().
		Int("server_count", len(servers)).
		Msg("Found active servers, creating workers")

	for _, server := range servers {
		if err := p.AddWorker(ctx, server); err != nil {
			logger.Get().Error().
				Err(err).
				Str("server_name", server.Name).
				Msg("Failed to create worker for server")
		}
	}

	logger.Get().Info().
		Int("worker_count", len(p.workers)).
		Msg("Worker pool started")

	return nil
}

func (p *Pool) AddWorker(ctx context.Context, server *domain.Server) error {
	p.mu.Lock()
	defer p.mu.Unlock()

	if _, exists := p.workers[server.ID]; exists {
		return fmt.Errorf("worker for server %s already exists", server.Name)
	}

	sshClient := infrastructure.NewSSHClient(&config.SyncCfg{
		SSHConTimeout: p.syncCfg.SSHConTimeout,
		SSHKeepAlive:  p.syncCfg.SSHKeepAlive,
		RetryMaxAtmpt: p.syncCfg.RetryMaxAtmpt,
		RetryDelay:    p.syncCfg.RetryDelay,
	})

	if err := sshClient.Connect(server); err != nil {
		return fmt.Errorf("failed to connect to server: %w", err)
	}

	workerCfg := &Config{
		Interval:    p.interval,
		SyncService: p.syncService,
		StatusRepo:  p.statusRepo,
	}

	worker := NewWorker(server.ID, server.Name, sshClient, workerCfg)

	if err := worker.Start(ctx); err != nil {
		sshClient.Close()
		return fmt.Errorf("failed to start worker: %w", err)
	}

	p.workers[server.ID] = worker

	logger.Get().Info().
		Str("server_id", server.ID.String()).
		Str("server_name", server.Name).
		Msg("Worker added for server")

	return nil
}

func (p *Pool) RemoveWorker(serverID uuid.UUID) error {
	p.mu.Lock()
	defer p.mu.Unlock()

	worker, exists := p.workers[serverID]
	if !exists {
		return fmt.Errorf("worker for server %s not found", serverID)
	}

	if err := worker.Stop(); err != nil {
		return fmt.Errorf("failed to stop worker: %w", err)
	}

	delete(p.workers, serverID)

	logger.Get().Info().
		Str("server_id", serverID.String()).
		Str("server_name", worker.GetServerName()).
		Msg("Worker removed")

	return nil
}

func (p *Pool) GetWorker(serverID uuid.UUID) (*Worker, error) {
	p.mu.RLock()
	defer p.mu.RUnlock()

	worker, exists := p.workers[serverID]
	if !exists {
		return nil, fmt.Errorf("worker not found for server %s", serverID)
	}

	return worker, nil
}

func (p *Pool) ListWorkers() []*Worker {
	p.mu.RLock()
	defer p.mu.RUnlock()

	workers := make([]*Worker, 0, len(p.workers))
	for _, w := range p.workers {
		workers = append(workers, w)
	}

	return workers
}

func (p *Pool) StopAll() {
	p.mu.Lock()
	defer p.mu.Unlock()

	for serverID, worker := range p.workers {
		if err := worker.Stop(); err != nil {
			logger.Get().Warn().
				Err(err).
				Str("server_id", serverID.String()).
				Msg("Error stopping worker")
		}
	}

	p.workers = make(map[uuid.UUID]*Worker)

	logger.Get().Info().Msg("All workers stopped")
}

func (p *Pool) GetStats() map[string]interface{} {
	p.mu.RLock()
	defer p.mu.RUnlock()

	stats := map[string]interface{}{
		"total_workers": len(p.workers),
		"workers":       make([]map[string]interface{}, 0),
	}

	workersStats := make([]map[string]interface{}, 0, len(p.workers))
	for _, worker := range p.workers {
		lastSync, syncCount, errorCount, lastError := worker.GetStats()
		workerStats := map[string]interface{}{
			"server_id":   worker.GetServerID().String(),
			"server_name": worker.GetServerName(),
			"state":       worker.GetState(),
			"last_sync":   lastSync,
			"sync_count":  syncCount,
			"error_count": errorCount,
			"last_error":  lastError,
		}
		workersStats = append(workersStats, workerStats)
	}
	stats["workers"] = workersStats

	return stats
}
```