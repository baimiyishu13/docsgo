+++
title = 'Mysqldump'
date = 2024-03-28T16:11:37+08:00
draft = true
+++

## 🍭 mysqldump



✅ 执行 mysqldump.sh 完成全量备份 变量 (必须):

1. BACKUP_DIR：备份目录
2. MYSQL_UNAME：用户
3. MYSQL_PWORD：密码
4. PATH：mysqldump命令路径
5. KEEP_BACKUPS_FOR：保留天数(默认7天)

🌐 恢复(示例)

```sh
for file in ./2024-03-07/*.sql.gz; do gunzip > "$file" | mysql -u your_username -p your_database_name; done
```



⛳️ cron(示例)

例：每天定时凌晨1点10分备份数据库

```
10 1 * * * /bin/bash /data/mysql/.sh/mysql_backup.sh
```



脚本：

```shell
#!/bin/bash
#==============================================================================
#TITLE:            mysql_backup.sh
#DESCRIPTION:      自动执行日常 mysql 备份的脚本
#DATE:             2024.03.07
#VERSION:          0.1
#USAGE:            ./mysql_backup.sh
#CRON:
  # 每天凌晨 00:00 执行数据库备份的 cron 示例
  # 0 0 * * *   /Users/[your user name]/scripts/mysql_backup.sh

# 从备份恢复
  #$  gunzip < [backupfile.sql.gz] | mysql -u [uname] -p[pass] [dbname]

#==============================================================================
# 自定义设置
#==============================================================================
# YYYY-MM-DD
TIMESTAMP=$(date +%F)

# 备份目录
BACKUP_DIR=/data/backup/$TIMESTAMP
mkdir $BACKUP_DIR
# MYSQL 用户密码
MYSQL_UNAME=root
MYSQL_PWORD=""

# mysqldump 命令
PATH=$PATH:/opt/mysql/bin/

# 保留天数
KEEP_BACKUPS_FOR=7 #days

#==============================================================================
# 方法
#==============================================================================

# 删除7天前备份文件
function delete_old_backups()
{
  echo "正在删除 $BACKUP_DIR/*.sql.gz $KEEP_BACKUPS_FOR 天前的备份文件"
  find $BACKUP_DIR -type d -ctime +7 -exec rm -rf {} \;
}

# 登陆 mysql
function mysql_login() {
  local mysql_login="-u $MYSQL_UNAME" 
  if [ -n "$MYSQL_PWORD" ]; then
    local mysql_login+=" -p$MYSQL_PWORD" 
  fi
  echo $mysql_login
}

# 列出 MySQL 数据库中的数据库列表
function database_list() {
  local show_databases_sql="SHOW DATABASES"
  echo $(mysql $(mysql_login) -e "$show_databases_sql"|awk -F " " '{if (NR!=1) print $1}')
}

function echo_status(){
  printf '\r'; 
  printf ' %0.s' {0..100} 
  printf '\r'; 
  printf "$1"'\r'
}

# 全量备份
function backup_database(){
    backup_file="$BACKUP_DIR/$TIMESTAMP.$database.sql.gz" 
    output+="$database => $backup_file\n"
    echo_status "...backing up $count of $total databases: $database"
    $(mysqldump $(mysql_login) $database | gzip -9 > $backup_file)
}

function backup_databases(){
  local databases=$(database_list)
  local total=$(echo $databases | wc -w | xargs)
  local output=""
  local count=1
  for database in $databases; do
    backup_database
    local count=$((count+1))
  done
  echo -ne $output | column -t
}

function hr(){
  printf '=%.0s' {1..100}
  printf "\n"
}

#==============================================================================
# RUN SCRIPT
#==============================================================================
delete_old_backups
hr
backup_databases
hr
printf "All backed up!\n\n"

```

