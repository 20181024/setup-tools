#!/bin/bash
set -e

echo "🔧 SSH 登录模式设置"
echo "1) 启用密码和密钥登录（默认）"
echo "2) 禁用密码，仅密钥登录（会自动写入公钥）"
echo "3) 恢复密码登录（保留密钥）"
read -p "请输入选项 [默认: 1]: " choice
choice=${choice:-1}

# 设置公钥（请自行替换为你的）
PUBKEY='ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKVOziPIfXRRQKCWSIGEuO/Dz68uLfZsnpnTY20u94Za my-ed25519'

# 公钥部署逻辑（1 和 2 都会执行）
deploy_pubkey() {
  mkdir -p /root/.ssh
  chmod 700 /root/.ssh
  if ! grep -q "$PUBKEY" /root/.ssh/authorized_keys 2>/dev/null; then
    echo "$PUBKEY" >> /root/.ssh/authorized_keys
    chmod 600 /root/.ssh/authorized_keys
    chown -R root:root /root/.ssh
    echo "✅ 公钥已写入 /root/.ssh/authorized_keys"
  else
    echo "🔍 公钥已存在，跳过写入"
  fi
}

# 配置 sshd_config
case "$choice" in
  1)
    echo "✅ 设置为允许密码 + 公钥登录"
    sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
    sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
    sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
    deploy_pubkey
    ;;
  2)
    echo "⚠️ 禁用密码登录，仅使用密钥"
    sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
    sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
    sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
    deploy_pubkey
    ;;
  3)
    echo "🔁 恢复密码登录（公钥保留）"
    sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
    sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
    sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
    ;;
  *)
    echo "⚠️ 输入无效，默认启用密码 + 公钥登录"
    sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
    sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
    sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
    deploy_pubkey
    ;;
esac

# 重启 ssh 服务
systemctl restart ssh
echo "✅ SSH 配置完成并已重启服务"
