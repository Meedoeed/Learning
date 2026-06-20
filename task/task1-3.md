# Задачи на сегодня

На сегодня стоят следующие задачи:
1) определение структуры SSH клиента
2) Интерфейс SSH клиента
3) Определить конфигурацию подключения
В совокупности это - определение интерфейса клиента ssh

# Интерфейс ssh-client

Интерфейс буду определять в infrastructure/ssh_client.go

Для sftp использую github/pkg/sftp


``` go 

type SSHClient struct {
	sshClint   *ssh.Client
	sftpClient *sftp.Client
	config     *config.SyncCfg
}


```

Для контракта клиента нужна доменная сущность сервера (для подключения). Я определил её так:
``` go
// internal/domain/server.go
package domain

type Server struct {
	ID         string
	Name       string
	Host       string
	Port       int
	Username   string
	AuthType   string
	Password   *string
	PrivateKey *string
}
```
Теперь определил набор методов, необходимый для клиента:
1) Открытие подключения
2) закрытие подключения
3) Определить список файлов
4) Скачать файлы
5) Загрузить файлы
6) Удалить файлы 

``` go
type SSHClientInterface interface {
	Connect(server *domain.Server) error
	Close() error
	ListFiles(remotePath string) ([]string, error)
	DownloadFile(remotePath, localPath string) error
	UploadFile(localPath, remotePath string) error
	DeleteFile(remotePath string) error
}
```

На сегодня поставленные задачи выполнены, завтра приступлю к реализации методов shh-client