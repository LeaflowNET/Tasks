# 开胃菜结束了

前面的任务都是开胃菜，这次让我们搭建一点更有挑战性的：部署多副本 WordPress。

这个任务不难，但绝对不会那么简单。不过我们还是简化了一些操作，任务要求是循序渐进，所以这里我们不部署读写分离的 MySQL 和集群 Redis。所以我们这里会部署一个单机的 MySQL， Redis 和多副本的 WordPress 然后并设置 HPA，高负载时自动扩容，随后自动缩容，实现成本控制和负载均衡。

期间你将学会

1. 部署有状态的 MySQL 和 Redis
2. 通过配置管理来不修改容器文件的情况下挂载数据到容器中
3. 如何配置 PHP 的 Session 保存到 Redis 中
4. 通过存储卷持久化 MySQL、Redis、WordPress 的数据
5. 通过服务管理，让同一个工作区的服务可以互相通信
6. 通过网站管理，创建一个 WordPress 网站并对外发布。

但是由于目前只有十堰集群具备多个节点，所以你的域名可能要备案后才能使用。

让我们来开始

# 1. 创建 MySQL、WordPress 和 Redis 的存储卷

首先到存储中下面的几个存储

1. mysql-data，容量 512Mi
2. redis-data，容量 512Mi
3. wordpress-data，容量 512Mi

当然你也可以按需求创建，但是为了完成这个任务，请务必保证名称一模一样。

# 2. 创建配置、密钥

到“配置映射”中，创建以下配置

1. MySQL

创建一个配置映射，名称为 `mysql-config`

| 配置键名       | 说明                   | 配置值    |
| -------------- | ---------------------- | --------- |
| MYSQL_DATABASE | 启动后创建的数据库名称 | wordpress |
| MYSQL_USER     | 启动后创建的数据库用户 | wordpress |

2. Redis

创建一个配置映射，名叫 redis-config，然后键名称为 `redis.conf`
值为以下内容

```conf
requirepass 请设置一个高强度的密码
```

然后我们还需要一份自定义的 php.ini 的配置，用于扩展 PHP 镜像中的 php 配置，到时候会放到 `/usr/local/etc/php/conf.d/` 下。

3. php-config
   然后我们还需要配置 php，输入名称为 php-config，键名称为 `php.ini`，值为以下内容

```ini
file_uploads = On
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
max_input_time = 1000
session.save_handler = redis
session.save_path = "tcp://:[密码]@redis-headless:6379"
```

注意，密码要去掉 “[]” 两个符号，但是不能去除前面的冒号和后面的@。

4. vhost 配置

之后还需要到 Apache2 中配置一个虚拟主机。创建一个 apache2-config。
第一个键名称为 `default.conf`，内容：

```conf
<VirtualHost *:80>
	#ServerName www.example.com
	ServerAdmin webmaster@localhost

  <If "%{HTTP:X-Forwarded-Proto} == 'https'">
      SetEnvIf X-Forwarded-Proto "https" HTTPS=on
  </If>

	DocumentRoot /var/www/html
	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

# 3. 创建密钥来存储密码

到“密钥管理”中，创建一个通用密钥，名叫 mysql-password

| 名称                | 说明                                          | 值                 |
| ------------------- | --------------------------------------------- | ------------------ |
| MYSQL_ROOT_PASSWORD | 超级用户的密码，请填写一个高强度的密码        | YourRandomPassword |
| MYSQL_PASSWORD      | 启动后创建的用户的密码,请填写一个高强度的密码 | wordpress          |

# 4. 创建工作负载

我们将会创建一个有状态的 MySQL 和 Redis 以及一个无状态的 WordPress。

下面是镜像
|说明|镜像|
|----|----|
|MySQL|m.daocloud.io/docker.io/mysql:9.4.0|
|Redis|redis:8.2.0-alpine|
|PHP With Apache2|ccr.ccs.tencentyun.com/leafphp/leafphp:8.4-apache|

但是这次，我们并没有使用 WordPress 官方镜像，因为官方镜像并没有打包 Redis 扩展，不过 Leaflow 准备了一个：`ccr.ccs.tencentyun.com/leafphp/leafphp:8.4-apache`。

下面是工作负载的配置

1. MySQL

到`创建应用`中，创建一个`有状态应用`，名叫 `mysql`，副本设置为 1，服务名称不用填。
到下方 `容器 1` 设置中，将容器名称设置为 `mysql`，镜像填写 `m.daocloud.io/docker.io/mysql:9.4.0` 。

资源限制中，内存设置为 1024，CPU 设置为 2000。

网络配置中，添加一个端口，端口名称填写 tcp-mysql，端口填写 3306，协议选择 TCP。

`从配置映射引用`中，添加一个从 `mysql-config` 的配置映射。
`从密钥映射引用`中，添加一个从 `mysql-password` 的密钥。

健康检查部分，添加一个 `就绪探针`，探针类型选择 `Exec Command`，命令列表需要添加 5 个，依次填写 `mysqladmin`, `ping`, `-h`, `127.0.0.1`, `--silent`。初始延迟设置为 30。

`从存储卷引用`中，添加一个从 `mysql-data` 的存储卷，挂载路径填写 `/var/lib/mysql`，子路径填写 `data`。

2. Redis

到`创建应用`中，创建一个`有状态应用`，名叫 `redis`，副本设置为 1，服务名称不用填。

然后将上方的`选择适合您经验水平的配置模式`设置为`专业`，否则有些参数找不到！
然后将上方的`选择适合您经验水平的配置模式`设置为`专业`，否则有些参数找不到！
然后将上方的`选择适合您经验水平的配置模式`设置为`专业`，否则有些参数找不到！

到下方 `容器 1` 设置中，将容器名称设置为 `redis`，镜像填写 `redis:8.2.0-alpine` 。

资源限制中，内存设置为 512 ，CPU 设置为 1000 。

网络配置中，添加一个端口，端口名称填写 `tcp-redis`，端口填写 `6379`，协议选择 `TCP`。

`从配置映射引用`中，添加一个从 `redis-config` 的配置映射。

健康检查部分，添加一个 `就绪探针`，探针类型选择 `Exec Command`，命令列表需要添加 5 个，依次填写 `redis-cli`, `-a`, `你设置的 Redis 密码`, `ping`。初始延迟设置为 10。

`从存储卷引用`中，添加一个从 `redis-data` 的存储卷，挂载路径填写 `/data`，子路径写 `data`。

接着从这个下方的“从配置映射挂载文件”中，将配置映射源设置为 `redis-config`，挂载路径写 `/usr/local/etc/redis/redis.conf`，`源键名`写`redis.conf`，目标路径为 `redis.conf`，子路径也为 `redis.conf`。

3. Apache2 + PHP

到`创建应用`中，创建一个`无状态应用`，名叫 `apache2`，副本设置为 1。

然后将上方的`选择适合您经验水平的配置模式`设置为`专业`，否则有些参数找不到！
然后将上方的`选择适合您经验水平的配置模式`设置为`专业`，否则有些参数找不到！
然后将上方的`选择适合您经验水平的配置模式`设置为`专业`，否则有些参数找不到！

到下方 `容器 1` 设置中，将容器名称设置为 `apache2`，镜像填写 `ccr.ccs.tencentyun.com/leafphp/leafphp:8.4-apache` 。

资源限制中，内存设置为 1024 ，CPU 设置为 2000 。

网络配置中，添加一个端口，端口名称填写 `http-80`，端口填写 `80`，协议选择 `TCP`。

`从存储卷引用`中，添加一个从 `wordpress-data` 的存储卷，挂载路径填写 `/var/www/html`，子路径填写 `html`。

接着从这个下方的“从配置映射挂载文件”中，将配置映射源设置为 `php-config`，挂载路径写 `/usr/local/etc/php/conf.d/99-custom.ini`，`源键名`写`php.ini`，目标路径为 `99-custom.ini`，子路径也为 `99-custom.ini`。

然后我们配置虚拟主机的映射，再次点击“添加”（注意，不是在 php-config 上添加），映射源选择 apache2-config，路径输入 /etc/apache2/sites-enabled

# 5. 创建服务

我们刚刚创建了一个基本的 LAMP 环境，但是这里我们为了测试一下 PHP Session 配置是否正确，所以需要暴露一下 Apache2 到公网中，方便我们测试。

到服务管理中，新建一个 `apache2` 的服务，服务类型选择 `负载均衡`。
目标工作负载选择无状态的 apache2，然后创建即可，之后我们会在服务那一栏看到一个外部的 IP，这就是临时测试用的 IP 地址。

# 6. 验证 Redis Session 配置情况。

下面的我们的重头戏了，我们需要写一个简单的 php 脚本用于获取当前主机名并启动一个 Session 然后获取 Session ID，并扩容工作负载，生成多个 apache2 容器组。

到文件管理中，选择 apache2 开头的容器组，然后导航到 `/var/www/html` 下，新建一个 `index.php` 文件，内容为以下内容

```php
<?php
session_start();

if (!isset($_SESSION['counter'])) {
    $_SESSION['counter'] = 1;
} else {
    $_SESSION['counter']++;
}

// ========================
// 输出信息
// ========================
echo "当前主机名：" . gethostname() . "<br>";
echo "Session ID：" . session_id() . "<br>";
echo "访问次数：{$_SESSION['counter']}<br>";

echo "<pre>Session 数据：\n";
print_r($_SESSION);
echo "</pre>";
```

然后打开上面服务管理中的外部 IP，持续刷新页面，可以看到访问次数正在增加的话，则说明这部分成功了。下面我们会进行多副本的测试。

# 7. 扩容工作负载

到 应用管理 中，选择 apache2 ，然后点击扩容，副本数设置为 3。然后等待部署成功后，多刷新几次，看到主机名变了并且访问次数还在增加，则说明扩容成功且验证通过，然后我们把副本重新设置回 1.
下面我们就可以安装 WordPress 了。

# 8. 下载 WordPress

你可以想说：这个网页文件管理怎么可能上传一份这么大的 WordPress 呢？
啊，当然不可能，但是我们可以使用在线下载的方式，将 WordPress 下载到容器中并解压。或者也可以使用维护助理手动 SFTP 或者 wget 下来。

这里我们使用在线下载的方式，将 WordPress 下载到容器中并解压。到文件管理中，将 index.php 文件内容替换下面的内容，保存后刷新页面。

```php
<?php
$wpZipUrl = 'https://wordpress.org/latest.zip';
$zipFile = __DIR__ . '/wordpress.zip';
echo "正在下载最新版 WordPress...\n";
$zipData = file_get_contents($wpZipUrl);
if ($zipData === false) {
    exit("下载失败，请检查网络连接。\n");
}
file_put_contents($zipFile, $zipData);
echo "下载完成: $zipFile\n";
$self = __FILE__;
echo "删除自身 index.php...\n";
unlink($self);
$zip = new ZipArchive;
if ($zip->open($zipFile) === TRUE) {
    $zip->extractTo(__DIR__);
    $zip->close();
    echo "解压完成\n";
} else {
    unlink($zipFile);
    exit("解压失败。\n");
}
$wpDir = __DIR__ . '/wordpress';
if (is_dir($wpDir)) {
    $dirIterator = new RecursiveIteratorIterator(
        new RecursiveDirectoryIterator($wpDir, RecursiveDirectoryIterator::SKIP_DOTS),
        RecursiveIteratorIterator::SELF_FIRST
    );
    foreach ($dirIterator as $item) {
        $destPath = __DIR__ . '/' . $dirIterator->getSubPathName();
        if ($item->isDir()) {
            if (!is_dir($destPath)) {
                mkdir($destPath, 0755, true);
            }
        } else {
            rename($item, $destPath);
        }
    }
    $it = new RecursiveDirectoryIterator($wpDir, RecursiveDirectoryIterator::SKIP_DOTS);
    $files = new RecursiveIteratorIterator($it, RecursiveIteratorIterator::CHILD_FIRST);
    foreach($files as $file) {
        $file->isDir() ? rmdir($file) : unlink($file);
    }
    rmdir($wpDir);
}
unlink($zipFile);
echo "WordPress 已安装到当前目录，请刷新页面。";
```

然后刷新页面，可以看到 WordPress 已经解压好了。接下来我们就可以安装 WordPress 了。

# 9. 准备一个备案域名并添加一个网站 （可跳过）

这一步是可选的，但是如果你想把这个教程直接部署为生产可用的话，还是建议走这一步。相当于部署成功后，你的 WordPress 可以直接拿出去用了。

到服务管理中，再创建一个服务，新建一个 `apache2-internal` 的服务，服务类型选择 `内部服务`，目标工作负载选择无状态的 apache2，然后创建即可。

到网站管理中，创建一个网站，名称可以写 `wordpress`。域名你可以选择你想要的一个域名，我这里就用 hidemo.ivampiresp.com。然后其余的不用管，服务选择 `apache2-internal`。

下方还有个 `TLS (HTTPS) 配置`，打开后填写上面输入的域名，可以自动生成一个自签名的 SSL 证书，如果你想要配置 SSL，可以打开此项，然后创建网站后自行编辑 SSL 证书为实际的证书即可。

创建成功后，点击查看详情，即可看到要解析到的地址，到 DNS 服务商那边解析即可。

在操作完成后，您可以删除上面创建的名叫“apache2”的外部负载均衡器的服务。

# 10. 安装 WordPress

前往您的 WordPress 安装地址，输入上方设置的数据库名称，用户，密码。数据库主机为 mysql-headless 下一步即可完成安装。

# 11. 设置自动扩缩

嗯，上面的东西已经很复杂了， 相比这个一定也很复杂吧。

不，不复杂，非常简单。

在“自动扩缩”中，点击“新增规则”，不用输入名称，直接选择 `apache2` 这个工作负载，然后点击创建即可。

如果你想的话也可以在上面调整副本数量。

# 12. 你已完成了全部设置！

现在你只是完成了 Leaflow 平台的一系列设置，但是距离优化 WordPress 还有一段路，比如安装 `WP Super Cache` 和设置 `Redis Object Cache`。但是这个不在本任务中赘述。

打开 https://www.itdog.cn/http/ ，测测看你的网站吧，多测试几次，你就会看到效果了。

在应用扩容后，如果长时间内负载不会很高，就自动缩容到 1 副本。
