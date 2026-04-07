# Проблематика

Для начала, перед реализацией сервиса хочу обозначить проблематику, чтобы в дальнейшем сформировать цели работы и иметь чёткое представление о задачах

Есть несколько удалённых SSH-серверов, которые работают в нестабильной среде:
1) они редко выходят на связь
2) могут пропадать из онлайна на часы или даже дни
3) соединения обрываются 

На каждом сервере есть две директории:

1) done/ - сюда кладутся результаты работы (забирать отсюда)
2) tasks/ - сюда нужно класть новые задания (отправлять сюда)

Почему нужна автоматизация?
Сейчас человеку, работающему с такой системой необходимо:
- Мониторить появление сервера в сети
- Подключаться по ssh
- производить много ручных операций: проверять директории, качать файлы, удалять другие файлы. Также руками класть файлы в tasks/
- Повторять такие операции нужно для каждого сервера - тяжело масштабировать

Все эти причины определяют текущую боль - создают нужду в автоматизации.

## Цели

Что нужно сделать? Автоматизировать процесс обмена файлами между локальной машиной и удалёнными SSH-серверами, а именно:

1) Мониторить, когда сервера появляются в сети
2) Автоматически забирать файлы из done - класть их в локальную директорию done/<server_name>
3) Автоматически при появлении серверов в онлайне отправлять на них файлы из локальной директории tasks/<server_name> - на соответствующий сервер в директорию tasks/
4) После передачи файлов их следует удалять
5) Для каждого отдельного сервера нужен свой воркер
6) Необходимо производить полное логирование в zerolog
7) WEB GUI, а в нём:
- статус серверов
- какие задачи в процессе
- какие задачи выполнены
- история синхронизаций

От себя:
Из отдельных фич мне хотелось бы здесь реализовать fault-tolerant download для файлов. Это довольно очевидная функция - она позволит при внезапном отключении сервера в процессе скачивания, в виду его нестабильной среды, сделать ряд retry-ев на заданных интервалах и продолжить скачивание с момента остановки - эдакий graceful degradation в контексте данного сервиса. Это одна из самых важных функций в контексте выполнения. 
По ходу реализации буду добавлять фичи, которые покажутся мне важными для этой реализации, подобно этой

# Стек

Бэк:
1) Go
2) Веб-сервер на labstack/echo 
3) Логгер: zerolog
4) Метрики: Prometheus
5) БД: Postgres
6) Вызовов ssh, scp через shell быть не должно. Для ssh - go

Фронт:
1) React 
2) Tailwind CSS


# ROADMAP

При выполнении задачи определил себе следующий курс:

1) Скелет сервиса: echo, zerolog, probes, metrics
2) SSH-SFTP клиент с resume-download
3) Postgres схемы
4) Worker для одного сервера 
5) Scheduler и Worker Pool
6) Rest API
7) React UI
8) Интеграция, тесты 
9) Документация

План по ходу скорее всего будет несколько изменяться, я накидал его за короткий промежуток времени, я вынесу его в файл ROADMAP-task1.md в этой директории и там буду постепенно по необходимости вносить правки

# Начало работы 

Реализовывать буду в виде слоистой архитектуры. Сразу создам архитектуру проекта - я, вероятно буду её видоизменять в прооцессе работы, но такие заглушки позволят мне в будущем не отвлекаться на особенности структурной организации и больше работать с кодом лишь изредка отвлекаясь на архитектуру. Итак:

Создаю GitHub-реп к нему подвязываю локальную директорию ssh-sync-automation
Создаю модуль проекта go
Ссылка на реп-проекта https://github.com/Meedoeed/ssh-sync-automation

## Слоистая архитектура

![архитекутура](../images/d04042.png)

Здесь main - точка входа
internal - приватный код
config - конфигурация приложения
domain - Бизнес-сущности
repository - интерфейсы для работы с БД
storage - Реализация repository
service - Сценарии использования
handler - HTTP - обработчики - слой доставки
infrastructure - Внешние зависимости (инструменты или utils)
migrations - SQL-миграции (схема Базы данных )

По стеку мы определили echo, zerolog, prometheus, postgres. Покрою необходимыми зависимостями:

Посмотрев на проект сверху легко понять какие компоненты сейчас наиболее необходимы:
``` go 
package main

func main() {
	// Загрузка конфига
	// Инициализация логера
	// Запуск http=сервера
}
```
Поочерёдно начинаю их писать. Это первый пункт ROADMAP-а и по сути сейчас я собираю каркас (скелет) сервиса

Начну с конфига. Поразмыслив над тем , что именно нужно конфигурировать в проекте пишу следующие структуры, определяющие общий конфиг сервиса, конфиг сервера (фактически только порт), конфиг для БД и конфиг для шедулера воркеров - думаю в дальнейшем будет полезен, но возможно его поля нужно будет чуть поправить.
``` go 
package config

import (
	"os"
	"strconv"
	"time"
)

type Config struct {
	Server   ServerCfg
	Database DatabaseCfg
	Sync     SyncCfg
	Log      LogCfg
}

type ServerCfg struct {
	Port string
}

type DatabaseCfg struct {
	Host     string
	Port     string
	User     string
	Password string
	DBName   string
	SSLMode  string
}

type SyncCfg struct {
	Interval      time.Duration
	RetryMaxAtmpt int
	RetryDelay    time.Duration
	SSHConTimeout time.Duration
	SSHKeepAlive  time.Duration
}

type LogCfg struct {
	Level string
}

func Load() *Config {
	return &Config{
		Server: ServerCfg{
			Port: getEnv("SERVER_PORT", "8080"),
		},
		Database: DatabaseCfg{
			Host:     getEnv("DB_HOST", "localhost"),
			Port:     getEnv("DB_PORT", "5432"),
			User:     getEnv("DB_USER", "postgres"),
			Password: getEnv("DB_PASSWORD", ""),
			DBName:   getEnv("DB_NAME", "ssh_sync"),
			SSLMode:  getEnv("DB_SSLMODE", "disable"),
		},
		Sync: SyncCfg{
			Interval:      getEnvDuration("SYNC_INTERVAL", 5*time.Minute),
			RetryMaxAtmpt: getEnvInt("RETRY_MAX_ATTEMPTS", 5),
			RetryDelay:    getEnvDuration("RETRY_DELAY", 5*time.Second),
			SSHConTimeout: getEnvDuration("SSH_CONNECT_TIMEOUT", 10*time.Second),
			SSHKeepAlive:  getEnvDuration("SSH_KEEPALIVE", 30*time.Second),
		},
		Log: LogCfg{
			Level: getEnv("LOG_LEVEL", "info"),
		},
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

func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
	if value := os.Getenv(key); value != "" {
		if duration, err := time.ParseDuration(value); err == nil {
			return duration
		}
	}
	return defaultValue
}

```

Выбирал между YAML-конфигом и классическим .env - выбор, как уже понятно остановился на втором поскольку он больше пригоден для коллективной разработки, более удобен для k8s. Помимо этого также при YAML-конфиге секреты улетят в git - оно не надо.

На этом конфиг готов, сразу вкину его в main.go 

``` go
package main

import (
	"github.com/Meedoeed/ssh-sync-automation/internal/config"
)

func main() {
	cfg := config.Load()
	// Инициализация логера
	// Запуск http=сервера
}
```

Теперь задача на следующий раз - созать Logger. Архитектурно он утилита - внешняя зависимость, поэтому место ему в internal/infrastructure/logger/logger.go