#!/bin/sh
set -eu
umask 077

BASE_DIR="/etc/sing-box"
CONFIG_FILE="$BASE_DIR/config.json"
STATE_FILE="$BASE_DIR/state.env"
CERT_FILE="$BASE_DIR/cert.pem"
KEY_FILE="$BASE_DIR/key.pem"
LOG_FILE="$BASE_DIR/info.log"

SB_BIN="/usr/local/bin/sing-box"
SYSTEMD_UNIT="/etc/systemd/system/sing-box.service"
OPENRC_INIT="/etc/init.d/sing-box"

HY2_UP_MBPS="${HY2_UP_MBPS:-100}"
HY2_DOWN_MBPS="${HY2_DOWN_MBPS:-100}"

SNI_HOST="${SNI_HOST:-www.bing.com}"

PUBLIC_IPV4=""
PUBLIC_IPV6=""

INIT_STYLE=""
SERVICE_FILE=""
TMPDIR_SBOX=""

USED_UDP_PORTS=""
USED_TCP_PORTS=""

HY2_LISTEN_PORT=""
TUIC_LISTEN_PORT=""
ANYTLS_LISTEN_PORT=""

AUTH_PASS=""
UUID=""

log() { printf '%s\n' "$*" >&2; }
have_cmd() { command -v "$1" >/dev/null 2>&1; }

shq() {
  printf "'%s'" "$(printf '%s' "$1" | sed "s/'/'\\\\''/g")"
}

fmt_hostport() {
  h="$1"
  p="$2"
  case "$h" in
    *:*) printf '[%s]:%s' "$h" "$p" ;;
    *)   printf '%s:%s' "$h" "$p" ;;
  esac
}

cleanup_tmp() {
  [ -n "${TMPDIR_SBOX:-}" ] && [ -d "${TMPDIR_SBOX:-}" ] && rm -rf "$TMPDIR_SBOX" >/dev/null 2>&1 || true
}
trap cleanup_tmp EXIT INT TERM

detect_init() {
  if [ -f /etc/alpine-release ] && have_cmd rc-update; then
    INIT_STYLE="openrc"
    SERVICE_FILE="$OPENRC_INIT"
  elif have_cmd systemctl; then
    INIT_STYLE="systemd"
    SERVICE_FILE="$SYSTEMD_UNIT"
  elif have_cmd rc-update; then
    INIT_STYLE="openrc"
    SERVICE_FILE="$OPENRC_INIT"
  else
    INIT_STYLE=""
    SERVICE_FILE=""
  fi
}

is_installed() {
  [ -x "$SB_BIN" ] && [ -f "$CONFIG_FILE" ] && [ -f "$STATE_FILE" ] && [ -f "$SERVICE_FILE" ]
}

pkg_install() {
  if have_cmd apk; then
    log "[INFO] 安装依赖（Alpine / apk）..."
    apk add --no-cache curl ca-certificates openssl gcompat iproute2 >/dev/null
  elif have_cmd apt-get; then
    log "[INFO] 安装依赖（Debian/Ubuntu / apt）..."
    apt-get update
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      curl tar ca-certificates openssl iproute2
    apt-get clean
    rm -rf /var/lib/apt/lists/*
  elif have_cmd dnf; then
    log "[INFO] 安装依赖（dnf）..."
    dnf install -y curl tar ca-certificates openssl iproute
  elif have_cmd yum; then
    log "[INFO] 安装依赖（yum）..."
    yum install -y curl tar ca-certificates openssl iproute
  else
    log "[ERROR] 未找到可用包管理器（apk/apt/dnf/yum）"
    exit 1
  fi
}

fetch_ip4() {
  curl -fsS4 --connect-timeout 5 --max-time 8 https://ip.sb 2>/dev/null || \
  curl -fsS4 --connect-timeout 5 --max-time 8 https://icanhazip.com 2>/dev/null || \
  true
}
fetch_ip6() {
  curl -fsS6 --connect-timeout 5 --max-time 8 https://ip.sb 2>/dev/null || \
  curl -fsS6 --connect-timeout 5 --max-time 8 https://icanhazip.com 2>/dev/null || \
  true
}

detect_public_ips() {
  PUBLIC_IPV4="$(fetch_ip4 | tr -d '\r\n' || true)"
  PUBLIC_IPV6="$(fetch_ip6 | tr -d '\r\n' || true)"
}

port_in_use() {
  p="$1"
  proto="$2"
  hex="$(printf '%04X' "$p")"

  case "$proto" in
    udp) files="/proc/net/udp /proc/net/udp6" ;;
    tcp) files="/proc/net/tcp /proc/net/tcp6" ;;
    *)
      return 1
      ;;
  esac

  for f in $files; do
    [ -r "$f" ] || continue
    if awk -v hp=":$hex" 'NR>1 && index($2, hp) {found=1; exit} END {if (found) exit 0; else exit 1}' "$f"; then
      return 0
    fi
  done
  return 1
}

proto_port_selected() {
  proto="$1"
  port="$2"
  case "$proto" in
    udp)
      case " $USED_UDP_PORTS " in
        *" $port "*) return 0 ;;
        *) return 1 ;;
      esac
      ;;
    tcp)
      case " $USED_TCP_PORTS " in
        *" $port "*) return 0 ;;
        *) return 1 ;;
      esac
      ;;
    *)
      return 1
      ;;
  esac
}

proto_port_mark() {
  proto="$1"
  port="$2"
  case "$proto" in
    udp) USED_UDP_PORTS="$USED_UDP_PORTS $port" ;;
    tcp) USED_TCP_PORTS="$USED_TCP_PORTS $port" ;;
  esac
}

rand_port() {
  proto="$1"
  while :; do
    h="$(openssl rand -hex 2)"
    n=$((0x$h))
    p=$((20000 + (n % 40000)))

    if ! port_in_use "$p" "$proto" && ! proto_port_selected "$proto" "$p"; then
      printf '%s' "$p"
      return 0
    fi
  done
}

prompt_port() {
  label="$1"
  proto="$2"
  while :; do
    default="$(rand_port "$proto")"
    printf '%s [%s]: ' "$label" "$default" >&2
    IFS= read -r ans || ans=""
    [ -z "$ans" ] && ans="$default"

    case "$ans" in
      ''|*[!0-9]*)
        log "请输入 1-65535 的数字"
        continue
        ;;
    esac

    if [ "$ans" -lt 1 ] || [ "$ans" -gt 65535 ]; then
      log "端口范围必须是 1-65535"
      continue
    fi

    if port_in_use "$ans" "$proto"; then
      log "端口 $ans 已被本机占用，请重新输入"
      continue
    fi

    if proto_port_selected "$proto" "$ans"; then
      log "端口 $ans 与前面同类协议冲突，请重新输入"
      continue
    fi

    proto_port_mark "$proto" "$ans"
    printf '%s' "$ans"
    return 0
  done
}

gen_creds() {
  AUTH_PASS="$(openssl rand -hex 4)"
  UUID="$(cat /proc/sys/kernel/random/uuid)"
}

gen_cert() {
  mkdir -p "$BASE_DIR"
  chmod 700 "$BASE_DIR"

  log "[INFO] 生成自签证书（CN=$SNI_HOST）..."
  openssl req -x509 -nodes -newkey rsa:2048 \
    -keyout "$KEY_FILE" \
    -out "$CERT_FILE" \
    -days 3650 \
    -subj "/CN=$SNI_HOST" >/dev/null 2>&1

  chmod 600 "$KEY_FILE" "$CERT_FILE"
}

save_state() {
  mkdir -p "$BASE_DIR"
  cat > "$STATE_FILE" <<EOF
SNI_HOST=$(shq "$SNI_HOST")
HY2_UP_MBPS=$(shq "$HY2_UP_MBPS")
HY2_DOWN_MBPS=$(shq "$HY2_DOWN_MBPS")
HY2_LISTEN_PORT=$(shq "$HY2_LISTEN_PORT")
TUIC_LISTEN_PORT=$(shq "$TUIC_LISTEN_PORT")
ANYTLS_LISTEN_PORT=$(shq "$ANYTLS_LISTEN_PORT")
AUTH_PASS=$(shq "$AUTH_PASS")
UUID=$(shq "$UUID")
EOF
  chmod 600 "$STATE_FILE"
}

load_state() {
  [ -f "$STATE_FILE" ] || return 1
  . "$STATE_FILE"
}

install_singbox() {
  local_arch="$(uname -m)"
  case "$local_arch" in
    x86_64|amd64) arch="amd64" ;;
    aarch64|arm64) arch="arm64" ;;
    armv7l|armv7*) arch="armv7" ;;
    *)
      log "[ERROR] 不支持的架构: $local_arch"
      exit 1
      ;;
  esac

  log "[INFO] 安装 sing-box..."
  TMPDIR_SBOX="$(mktemp -d)"

  latest_json="$TMPDIR_SBOX/latest.json"
  curl -fsSL -H 'User-Agent: curl' \
    -o "$latest_json" \
    https://api.github.com/repos/SagerNet/sing-box/releases/latest

  latest="$(sed -n 's/.*"tag_name":[[:space:]]*"\([^"]*\)".*/\1/p' "$latest_json" | head -n 1)"
  [ -n "$latest" ] || {
    log "[ERROR] 获取 sing-box 最新版本失败"
    exit 1
  }

  ver="${latest#v}"
  url="https://github.com/SagerNet/sing-box/releases/download/${latest}/sing-box-${ver}-linux-${arch}.tar.gz"

  log "[INFO] 下载 sing-box: $latest ($arch)"
  curl -fL --retry 3 -o "$TMPDIR_SBOX/sing-box.tar.gz" "$url"

  tar -xzf "$TMPDIR_SBOX/sing-box.tar.gz" -C "$TMPDIR_SBOX"

  sb_dir="$(find "$TMPDIR_SBOX" -maxdepth 1 -type d -name "sing-box-${ver}-linux-*" | head -n 1)"
  if [ -z "$sb_dir" ] || [ ! -f "$sb_dir/sing-box" ]; then
    log "[ERROR] 解压 sing-box 失败"
    exit 1
  fi

  install -m 755 "$sb_dir/sing-box" "$SB_BIN"
  log "[INFO] sing-box 已安装到 $SB_BIN"
}

check_singbox_exec() {
  if ! "$SB_BIN" version >/dev/null 2>&1; then
    log "[ERROR] sing-box 已安装但无法执行。"
    log "[ERROR] 若在 Alpine 上运行，请确认已安装 gcompat。"
    exit 1
  fi
}

write_config() {
  mkdir -p "$BASE_DIR"
  cat > "$CONFIG_FILE" <<EOF
{
  "log": {
    "level": "warn",
    "timestamp": true,
    "output": "$LOG_FILE",
  },
  "inbounds": [
    {
      "type": "hysteria2",
      "tag": "hy2",
      "listen": "::",
      "listen_port": $HY2_LISTEN_PORT,
      "up_mbps": $HY2_UP_MBPS,
      "down_mbps": $HY2_DOWN_MBPS,
      "users": [
        {
          "password": "$AUTH_PASS"
        }
      ],
      "tls": {
        "enabled": true,
        "certificate_path": "$CERT_FILE",
        "key_path": "$KEY_FILE",
        "alpn": ["h3"]
      }
    },
    {
      "type": "tuic",
      "tag": "tuic",
      "listen": "::",
      "listen_port": $TUIC_LISTEN_PORT,
      "users": [
        {
          "uuid": "$UUID",
          "password": "$AUTH_PASS"
        }
      ],
      "congestion_control": "bbr",
      "tls": {
        "enabled": true,
        "certificate_path": "$CERT_FILE",
        "key_path": "$KEY_FILE",
        "alpn": ["h3"]
      }
    },
    {
      "type": "anytls",
      "tag": "anytls",
      "listen": "::",
      "listen_port": $ANYTLS_LISTEN_PORT,
      "users": [
        {
          "password": "$AUTH_PASS"
        }
      ],
      "tls": {
        "enabled": true,
        "certificate_path": "$CERT_FILE",
        "key_path": "$KEY_FILE"
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    }
  ],
  "route": {
    "final": "direct"
  }
}
EOF

  chmod 600 "$CONFIG_FILE"
}

write_service() {
  if [ "$INIT_STYLE" = "systemd" ]; then
    cat > "$SYSTEMD_UNIT" <<EOF
[Unit]
Description=sing-box 3in1（hy2/tuic/Anytls）
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=$SB_BIN run -c $CONFIG_FILE
Restart=on-failure
RestartSec=3
LimitNOFILE=1048576
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF
    chmod 644 "$SYSTEMD_UNIT"
  elif [ "$INIT_STYLE" = "openrc" ]; then
    cat > "$OPENRC_INIT" <<EOF
#!/sbin/openrc-run
description="sing-box 3in1（hy2/tuic/Anytls）"
command="$SB_BIN"
command_args="run -c $CONFIG_FILE"
command_background="yes"
pidfile="/run/\${RC_SVCNAME}.pid"

depend() {
  need net
}
EOF
    chmod 755 "$OPENRC_INIT"
  else
    log "[ERROR] 未检测到 systemd 或 OpenRC"
    exit 1
  fi
}

ensure_firewall() {
  # 仅尝试放行本机防火墙；云厂商安全组仍需手动放行
  if have_cmd ufw; then
    if ufw status 2>/dev/null | grep -q 'Status: active'; then
      ufw allow "${HY2_LISTEN_PORT}/udp" >/dev/null 2>&1 || true
      ufw allow "${TUIC_LISTEN_PORT}/udp" >/dev/null 2>&1 || true
      ufw allow "${ANYTLS_LISTEN_PORT}/tcp" >/dev/null 2>&1 || true
    fi
  fi

  if have_cmd firewall-cmd; then
    if firewall-cmd --state >/dev/null 2>&1; then
      firewall-cmd --permanent --add-port="${HY2_LISTEN_PORT}/udp" >/dev/null 2>&1 || true
      firewall-cmd --permanent --add-port="${TUIC_LISTEN_PORT}/udp" >/dev/null 2>&1 || true
      firewall-cmd --permanent --add-port="${ANYTLS_LISTEN_PORT}/tcp" >/dev/null 2>&1 || true
      firewall-cmd --reload >/dev/null 2>&1 || true
    fi
  fi
}

stop_service() {
  if [ "$INIT_STYLE" = "systemd" ]; then
    systemctl stop sing-box >/dev/null 2>&1 || true
  elif [ "$INIT_STYLE" = "openrc" ]; then
    rc-service sing-box stop >/dev/null 2>&1 || true
  fi
  sleep 1
}

disable_service() {
  if [ "$INIT_STYLE" = "systemd" ]; then
    systemctl disable sing-box >/dev/null 2>&1 || true
    systemctl daemon-reload >/dev/null 2>&1 || true
  elif [ "$INIT_STYLE" = "openrc" ]; then
    rc-update del sing-box default >/dev/null 2>&1 || true
  fi
}

start_service() {
  if [ "$INIT_STYLE" = "systemd" ]; then
    systemctl daemon-reload
    systemctl enable sing-box >/dev/null 2>&1 || true
    systemctl restart sing-box >/dev/null 2>&1 || true
    sleep 2
    if ! systemctl is-active --quiet sing-box; then
      log "[ERROR] sing-box 启动失败，查看日志："
      journalctl -u sing-box -e --no-pager || true
      exit 1
    fi
  elif [ "$INIT_STYLE" = "openrc" ]; then
    rc-update add sing-box default >/dev/null 2>&1 || true
    rc-service sing-box restart >/dev/null 2>&1 || rc-service sing-box start >/dev/null 2>&1 || true
    sleep 2
    if ! rc-service sing-box status >/dev/null 2>&1; then
      log "[ERROR] sing-box 启动失败，可执行：rc-service sing-box status"
      exit 1
    fi
  else
    log "[ERROR] 未检测到可用服务管理器"
    exit 1
  fi
}

restart_service() {
  if [ "$INIT_STYLE" = "systemd" ]; then
    systemctl daemon-reload
    if ! systemctl restart sing-box >/dev/null 2>&1; then
      log "[ERROR] sing-box 重启失败，查看日志："
      journalctl -u sing-box -e --no-pager || true
      exit 1
    fi
    sleep 2
    if ! systemctl is-active --quiet sing-box; then
      log "[ERROR] sing-box 重启后未处于 active 状态，查看日志："
      journalctl -u sing-box -e --no-pager || true
      exit 1
    fi
  elif [ "$INIT_STYLE" = "openrc" ]; then
    if ! rc-service sing-box restart >/dev/null 2>&1; then
      log "[ERROR] sing-box 重启失败，可执行：rc-service sing-box status"
      exit 1
    fi
    sleep 2
    if ! rc-service sing-box status >/dev/null 2>&1; then
      log "[ERROR] sing-box 重启后状态异常，可执行：rc-service sing-box status"
      exit 1
    fi
  else
    log "[ERROR] 未检测到可用服务管理器"
    exit 1
  fi
}

emit_hy2() {
  host="$1"
  kind="$2"
  hp="$(fmt_hostport "$host" "$HY2_LISTEN_PORT")"
  printf '%s\n' "hysteria2://$AUTH_PASS@$hp?sni=$SNI_HOST&alpn=h3&insecure=1&allowInsecure=1#HY2_$kind"
}

emit_tuic() {
  host="$1"
  kind="$2"
  hp="$(fmt_hostport "$host" "$TUIC_LISTEN_PORT")"
  userinfo="${UUID}%3A${AUTH_PASS}"
  printf '%s\n' "tuic://$userinfo@$hp?sni=$SNI_HOST&alpn=h3&insecure=1&allowInsecure=1&congestion_control=bbr#TUIC_$kind"
}

emit_anytls() {
  host="$1"
  kind="$2"
  hp="$(fmt_hostport "$host" "$ANYTLS_LISTEN_PORT")"
  kind_lc="$(printf '%s' "$kind" | tr '[:upper:]' '[:lower:]')"
  printf '%s\n' "anytls://$AUTH_PASS@$hp?security=tls&insecure=1&allowInsecure=1&type=tcp#AnyTLS_${kind_lc}"
}

show_node_set() {
  host="$1"
  label="$2"
  printf '\n[%s]\n' "$label"
  emit_hy2 "$host" "$label"
  emit_tuic "$host" "$label"
  emit_anytls "$host" "$label"
}

show_nodes() {
  load_state || true
  detect_public_ips
  printf '\n%s\n' "===== 节点信息 ====="
  if [ -n "$PUBLIC_IPV4" ]; then
    show_node_set "$PUBLIC_IPV4" "V4"
  fi
  if [ -n "$PUBLIC_IPV6" ]; then
    show_node_set "$PUBLIC_IPV6" "V6"
  fi
  if [ -z "$PUBLIC_IPV4" ] && [ -z "$PUBLIC_IPV6" ]; then
    log "[WARN] 未检测到公网 IPv4/IPv6"
  fi
  printf '\n'
}

fresh_install() {
  pkg_install
  install_singbox
  check_singbox_exec

  log "[INFO] 正在检测公网 IPv4/IPv6..."
  detect_public_ips
  log "[INFO] 检测到公网 IPv4: ${PUBLIC_IPV4:-未检测到}"
  log "[INFO] 检测到公网 IPv6: ${PUBLIC_IPV6:-未检测到}"

  HY2_LISTEN_PORT="$(prompt_port '请输入 HY2 端口 (UDP) [回车随机]' udp)"
  TUIC_LISTEN_PORT="$(prompt_port '请输入 TUIC 端口 (UDP) [回车随机]' udp)"
  ANYTLS_LISTEN_PORT="$(prompt_port '请输入 ANYTLS 端口 (TCP) [回车随机]' tcp)"

  gen_creds
  gen_cert
  save_state
  write_config
  write_service
  ensure_firewall

  log "[INFO] 检查 sing-box 配置..."
  "$SB_BIN" check -c "$CONFIG_FILE"

  log "[INFO] 启动服务..."
  start_service

  show_nodes
}

reset_ports() {
  load_state

  stop_service
  USED_UDP_PORTS=""
  USED_TCP_PORTS=""

  HY2_LISTEN_PORT="$(prompt_port '请输入 HY2 端口 (UDP) [回车随机]' udp)"
  TUIC_LISTEN_PORT="$(prompt_port '请输入 TUIC 端口 (UDP) [回车随机]' udp)"
  ANYTLS_LISTEN_PORT="$(prompt_port '请输入 ANYTLS 端口 (TCP) [回车随机]' tcp)"

  save_state
  write_config
  ensure_firewall

  log "[INFO] 检查 sing-box 配置..."
  "$SB_BIN" check -c "$CONFIG_FILE"

  log "[INFO] 重启服务..."
  start_service

  show_nodes
}

uninstall_all() {
  stop_service
  disable_service

  rm -f "$SYSTEMD_UNIT" "$OPENRC_INIT" 2>/dev/null || true
  rm -f "$SB_BIN" 2>/dev/null || true
  rm -rf "$BASE_DIR" 2>/dev/null || true

  if [ "$INIT_STYLE" = "systemd" ]; then
    systemctl daemon-reload >/dev/null 2>&1 || true
  fi
}

reinstall_all() {
  uninstall_all
  fresh_install
}

menu_loop() {
  load_state || true

  while :; do
    printf '\n%s\n' "===== sing-box 3in1（hy2/tuic/Anytls）管理脚本 ====="
    printf '%s\n' "1) 重装"
    printf '%s\n' "2) 卸载"
    printf '%s\n' "3) 重置端口"
    printf '%s\n' "4) 重启服务"
    printf '%s\n' "5) 查看节点"
    printf '%s\n' "0) 退出"
    printf '%s' "请选择 [0-5]: " >&2

    IFS= read -r choice || choice=""

    case "$choice" in
      1)
        log "[INFO] 开始重新安装..."
        reinstall_all
        exit 0
        ;;
      2)
        log "[INFO] 开始卸载..."
        uninstall_all
        printf '%s\n' "[INFO] 已卸载"
        exit 0
        ;;
      3)
        reset_ports
        ;;
      4)
        log "[INFO] 重启服务..."
        restart_service
        printf '%s\n' "[INFO] 服务已重启"
        ;;
      5)
        show_nodes
        ;;
      0|'')
        exit 0
        ;;
      *)
        log "请输入 0-5 之间的选项"
        ;;
    esac
  done
}

main() {
  if [ "$(id -u)" -ne 0 ]; then
    log "[ERROR] 请使用 root 运行"
    exit 1
  fi

  detect_init
  if [ -z "$INIT_STYLE" ]; then
    log "[ERROR] 未检测到 systemd / OpenRC"
    exit 1
  fi

  if is_installed; then
    menu_loop
  else
    fresh_install
  fi
}

main "$@"
