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