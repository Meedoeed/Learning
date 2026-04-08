# Logger 

Стояла задача написать Logger на zerolog
Мной были в реализации предприняты следующие решения:
Я ещё на этапе создания конфигуратора сервиса задал структуру LoggerCfg с полем Level

Я хочу инициализировать своим логгером глобальный логгер zerolog чтобы избежать множественных объектов zerolog.Logger и оставить только один - переинициализированный стоковый логгер

Я для этого сделал две функции - конструктор и геттер

## New feature 

Помимо этого решил сделать фичу, которая уже по сути предусмотрена zerolog-ом: дать возможность выбора между JSON-логами (вариант для прода и человеко-читаемыми цветными логами). Для этого конструктор на входе имеет на входе помимо level string поле pretty bool. Я не стал выносить его в .env конфиг и обновлять парсер конфигов, вот почему:
1) параметр редко меняется и в проде логгер всегда JSON
2) На логику приложения pretty никак не влияет - это чисто визуально приятнее для пользователя - потенциально, отладчика

## Реализация

Моя реализация логгера на zerolog такая:
``` go
package logger

import (
	"os"
	"time"

	"github.com/rs/zerolog"
	"github.com/rs/zerolog/log"
)

var SuperLogger *zerolog.Logger

func Init(level string, pretty bool) { // функция-конструктор для глобального логгера SuperLogger
	zerolog.TimeFieldFormat = time.RFC3339Nano

	logLevel, err := zerolog.ParseLevel(level)
	if err != nil {
		logLevel = zerolog.InfoLevel
	}
	zerolog.SetGlobalLevel(logLevel)

	var logger zerolog.Logger

	if pretty { // для отладки человеко-читаемый
		output := zerolog.ConsoleWriter{
			Out:        os.Stdout,
			TimeFormat: "15:04:05.000",
			NoColor:    false,
		}
		logger = zerolog.New(output).With().Timestamp().Logger()
	} else {
		logger = zerolog.New(os.Stdout).With().Timestamp().Logger()
	}

	SuperLogger = &logger
	log.Logger = logger // глобальный логгер поменял на настроенный с дефолтного
}

func Get() *zerolog.Logger { // геттер для логгера
	if SuperLogger == nil {
		Init("Info", true) // если не инициализировали логгер ранее - создаём дефолтный отладочный
	}
	return SuperLogger
}
```

# Точка входа

Добавил логеер в main.go, следующим этапом будет http-сервер для которого в коде ниже я оставил TODO - часто прибегаю к этому чтобы не забыть чего-нибудь сделать.

Я заранее прописал заглушку для graceful shutdown сделал контекст с 10-секундным таймаутом и заглушкой _ = ctx для будущей реализации

``` go
package main

import (
    ...
)

func main() {
	cfg := config.Load()
	logger.Init(cfg.Log.Level, true) // в разработке true, в проде false
	logger.Get().Info().Msg("Starting SSH-SYNC-AUTOMATION service")
	// Запуск http=сервера

	// TODO http-server

	// сразу сделал задел на graceful shutdown сервиса - пока что _=ctx - заглушка
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	_ = ctx

	logger.Get().Info().Msg("Service stopped")
}
```

# Архитектура

Было принято решение создать новые слои:

handler - здесь буду держать обработчики
middleware - тут соответственно мидлвари
server - тут инициализирую HTTP-сервер с путями и мидлварями

middleware к бизнес-логике не относится и выполняет сквозные технические задачи - заложил поэтому его в infrastructure. Остальное в internal


## Handler
Обработчики сделал следующие:

1) health:
тут и обработчик для /heathz и для /live

С заделом на последующий DI с бд оформил в видде heathhandler в виде структуры

Оставил для Readiness - заглушку с TODO до реализации storage с БД

``` go
package handler

import (
	...
)

type HealthHandler struct{}

func NewHealthHandler() *HealthHandler {
	return &HealthHandler{}
}

func (h *HealthHandler) Liveness(c echo.Context) error {
	return c.JSON(http.StatusOK, map[string]string{"status": "alive"})
}

func (h *HealthHandler) Readiness(c echo.Context) error {
	// TODO после реализации БД
	// проверять статус БД
	return c.JSON(http.StatusOK, map[string]string{"status": "ready"})
}

```

# Middleware
В мидлвари логично - добавил вышенаписанный логер:

``` go
package middleware

import (
	"time"

	"github.com/Meedoeed/ssh-sync-automation/internal/infrastructure/logger"
	"github.com/labstack/echo/v4"
)

func EchoLogger() echo.MiddlewareFunc {
	return func(next echo.HandlerFunc) echo.HandlerFunc {
		return func(c echo.Context) error {
			start := time.Now()
			err := next(c)
			latency := time.Since(start)

			logger.Get().Info().
				Str("method", c.Request().Method).
				Str("uri", c.Request().RequestURI).
				Int("status", c.Response().Status).
				Dur("latency", latency).
				Str("ip", c.RealIP()).
				Msg("HTTP request")

			return err
		}
	}
}
```

Тут в целом всё понятно

# Сборка и создание HTTP-server

Всё написанное выше собираю в internal/server/http.go

В цепочку мидлварей добавил Recovery и новый для себя RequsetID - интересная мидлварь позволяет создавать уникальный сквозной id для запроса на всех слоях приложения, что очень удобно для отладки.

В остальном всё более понятно:
Функция start для запуска сервера
Функция shutdown - для graceful shutdown

``` go 
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
)

type HTTPServer struct {
	echo   *echo.Echo
	config *config.Config
}

func NewHTTP(cfg *config.Config) *HTTPServer {
	e := echo.New()
	e.HideBanner = true
	e.HidePort = true

	e.Use(middleware.Recover())
	e.Use(middleware.RequestID())
	e.Use(myMiddleware.EchoLogger())

	healthHandler := handler.NewHealthHandler()
	handler.RegisterRoutes(e, healthHandler)

	return &HTTPServer{
		echo:   e,
		config: cfg,
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

В файле routes настраиваю роутер маршрутов:
``` go
package handler

import (
	"github.com/labstack/echo/v4"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func RegisterRoutes(e *echo.Echo, healthHandler *HealthHandler) {
	e.GET("/live", healthHandler.Liveness)
	e.GET("/healthz", healthHandler.Readiness)
	e.GET("/metrics", echo.WrapHandler(promhttp.Handler()))
}
```

# Заканчиваю со скелетом и собираю в main.go

``` go
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
)

func main() {
	cfg := config.Load()
	logger.Init(cfg.Log.Level, true) // в разработке true, в проде false
	logger.Get().Info().Msg("Starting SSH-SYNC-AUTOMATION service")

	httpServer := server.NewHTTP(cfg)

	go func() {
		if err := httpServer.Start(); err != nil {
			logger.Get().Fatal().Err(err).Msg("Failed to start HTTP server")
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := httpServer.Shutdown(ctx); err != nil {
		logger.Get().Error().Err(err).Msg("HTTP server shutdown error")
	}

	logger.Get().Info().Msg("Service stopped")
}
```