+++
title = 'Root Ssh Login Sh'
date = 2024-04-01T15:58:40+08:00
draft = true
+++



配置SSH以允许root用户无密码登录




脚本：

```sh
#!/bin/bash

print_script_header() {
    echo "-----------------------------------------------"
    echo "# 脚本: root_ssh_login.sh"
    echo "# 描述: 配置SSH以允许root用户无密码登录"
    echo "# 作者:"
    echo "# 日期: 2024"
    echo "-----------------------------------------------"
}


# 示例用法
users=("root")

input(){
# 提示用户输入节点，以空格分隔
read -p "请输入节点名称（以空格分隔）: " nodes_input
# 将输入的节点字符串分割成数组
IFS=' ' read -ra nodes <<< "$nodes_input"
}

# 函数：提示用户输入密码并创建password.txt
prompt_for_password() {
    read -s -p "请输入密码: " password
    echo "$password" > /tmp/password.txt
}

# 函数：检查并使用yum安装sshpass
install_sshpass() {
    if ! command -v sshpass &> /dev/null; then
      echo "提示：sshpass未安装，正在安装..."
      yum install -y sshpass > /dev/null 2>&1

      # 检查 yum install 是否执行成功
      if [ $? -eq 0 ]; then
          echo "sshpass安装成功。"
      else
          echo "错误：sshpass安装失败。"
          exit 1
      fi
  else
      echo "提示：sshpass已安装."
  fi
}

# 函数：检查并生成SSH密钥对
generate_ssh_key() {
    local key_path="/root/.ssh/id_rsa"

    if [ ! -f "$key_path" ]; then
        # If the key does not exist, generate it
        ssh-keygen -t rsa -b 2048 -f "$key_path" -N ""   > /dev/null 2>&1
        echo "密钥已生成"
    else
        # If the key already exists, display a message
        echo "密钥已存在，不再生成新的密钥。"
    fi
}


# 函数：将公钥复制到authorized_keys
copy_public_key() {
    local user="$1"
    shift
    local nodes=("$@")
    count=1
    echo -e "\n\t ***** 复制公钥 *****"
    for node in "${nodes[@]}"; do
        sshpass -f /tmp/password.txt ssh-copy-id -o StrictHostKeyChecking=no "${user}@${node}"   > /dev/null 2>&1
        if [ $? -eq 0 ]; then
            echo -e "\e[34m\t[${count}] 公钥成功复制到 ${node}\e[0m"
        else
            echo -e "\e[31m\t[${count}] ERROR 无法复制公钥到 ${node}\e[0m"
        fi
        ((count++))  # 计数器递增
    done
}

# 主脚本
main() {
    print_script_header
    install_sshpass
    input
    prompt_for_password
    generate_ssh_key

    for user in "${users[@]}"; do
        copy_public_key "${user}" "${nodes[@]}"
    done

    echo -e "\e[32m\n root_ssh_login.sh 脚本开结束 \e[0m"
}

# 执行主脚本
main
```

