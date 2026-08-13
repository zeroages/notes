# Linux 常用命令

* PostgreSQL 相关

```bash
sudo -i -u postgres        # 切换到 postgres 用户
psql                        # 启动 psql 客户端
\l                          # 查看所有数据库
\c <数据库名>                # 切换数据库
\dt                         # 查看所有表
\d <表名>                   # 查看表结构
\q                          # 退出 psql（或 Ctrl+D）
```

* Nginx 命令

```bash
sudo systemctl start nginx                    # 启动 Nginx
sudo systemctl restart nginx                  # 重启 Nginx
sudo systemctl reload nginx                   # 重新加载配置
sudo nginx -t                                 # 测试配置文件
cd /etc/nginx/sites-available                 # 查看配置目录
sudo kill -9 $(sudo lsof -t -i:9520)          # 关闭指定端口进程
```

* PHP 命令

| 命令 | 说明 |
|------|------|
| sudo systemctl start php7.4-fpm | 启动 PHP-FPM |
| sudo systemctl stop php7.4-fpm | 停止 PHP-FPM |
| sudo systemctl restart php7.4-fpm | 重启 PHP-FPM |
| sudo systemctl status php7.4-fpm | 查看状态 |
| sudo systemctl enable php7.4-fpm | 开机自启 |

> **Tips：** PHP 项目如果是宝塔部署会存在 `.user.ini` 配置，内容为 `open_basedir=/xx/xx/:/tmp/:/proc/`，限制目录读取，访问页面时显示 `No input file specified`，删除该配置文件即可。

* 防火墙命令（UFW）

```bash
sudo ufw allow 22/tcp              # 开放端口
sudo ufw deny 22/tcp               # 关闭端口
sudo ufw delete 1                  # 删除规则（按编号）
sudo ufw delete allow 3306/tcp     # 删除指定允许规则
sudo ufw reload                    # 重启防火墙
sudo ufw status                    # 查看状态
```

* MySQL 常用命令

```sql
SHOW DATABASES;                                              -- 查看数据库
mysql -uroot -p                                              -- 连接数据库
USE <数据库名>;                                               -- 切换数据库
SHOW TABLES;                                                 -- 查看表
SHOW COLUMNS FROM <表名>;                                     -- 查看字段
ALTER TABLE <表名> ADD COLUMN <新字段名> <数据类型>;           -- 添加字段
mysqldump -u root -p <数据库名> > backup.sql                  -- 导出数据库
UPDATE <表名> SET <列名> = <值> WHERE <条件>;                 -- 修改数据
```

* 定时任务

```bash
crontab -e    # 设置定时任务
```
