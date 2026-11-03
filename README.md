
cat > docker-setup.sh <<'SCRIPT'
#!/usr/bin/env bash

set -Eeuo pipefail
umask 077

ARVAN_MIRROR="https://docker.arvancloud.ir"
DOCKER_CONFIG="/etc/docker/daemon.json"
DOCKER_SOURCE="/etc/apt/sources.list.d/docker.sources"

log()  { echo -e "\n\033[1;34m[INFO]\033[0m $*"; }
ok()   { echo -e "\033[1;32m[ OK ]\033[0m $*"; }
warn() { echo -e "\033[1;33m[WARN]\033[0m $*" >&2; }
die()  { echo -e "\n\033[1;31m[ERROR]\033[0m $*" >&2; exit 1; }

on_error() {
    local code=$?
    echo -e "\n\033[1;31m[ERROR]\033[0m خطا در خط ${BASH_LINENO[0]:-unknown} (کد ${code})" >&2
    exit "$code"
}
trap on_error ERR

# بررسی دسترسی
[[ "$EUID" -eq 0 ]] || die "این اسکریپت را با root اجرا کن."

# بررسی سیستم‌عامل
source /etc/os-release

[[ "${ID:-}" == "ubuntu" ]] || die "فقط Ubuntu پشتیبانی می‌شود."

[[ "${VERSION_ID:-}" == "24.04" ]] ||
    die "این نسخه اسکریپت مخصوص Ubuntu 24.04 است."

ARCH="$(dpkg --print-architecture)"

[[ "$ARCH" == "amd64" || "$ARCH" == "arm64" ]] ||
    die "معماری پشتیبانی نمی‌شود: $ARCH"

log "سیستم: ${PRETTY_NAME}"
log "معماری: ${ARCH}"

# جلوگیری از تداخل با Docker CE
for pkg in docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-buildx-plugin; do
    if dpkg-query -W -f='${Status}' "$pkg" 2>/dev/null | grep -q "install ok installed"; then
        die "بسته ${pkg} از مخزن Docker نصب شده است. برای جلوگیری از تداخل، نصب متوقف شد."
    fi
done

# حذف مخزن Docker CE که در اجرای قبلی باعث خطای 403 شد
if [[ -f "$DOCKER_SOURCE" ]]; then
    log "حذف فایل مخزن رسمی Docker که قبلاً اضافه شده بود..."
    rm -f "$DOCKER_SOURCE"
fi

# به‌روزرسانی مخازن Ubuntu
log "به‌روزرسانی مخازن Ubuntu..."

apt-get update

# نصب Docker و Compose Plugin
log "نصب Docker Engine و Docker Compose Plugin..."

DEBIAN_FRONTEND=noninteractive apt-get install -y \
    ca-certificates \
    curl \
    python3 \
    docker.io \
    docker-compose-v2 \
    docker-buildx

# فعال‌سازی Docker
log "فعال‌سازی سرویس Docker..."

systemctl enable --now docker

systemctl is-active --quiet docker ||
    die "سرویس Docker فعال نشد."

ok "Docker Engine نصب و فعال شد."

# تنظیم میرور آروان با حفظ تنظیمات موجود
log "تنظیم میرور آروان..."

mkdir -p /etc/docker

if [[ -f "$DOCKER_CONFIG" ]]; then
    python3 -m json.tool "$DOCKER_CONFIG" >/dev/null ||
        die "فایل daemon.json معتبر نیست. آن را دستی بررسی کن."
fi

python3 - "$DOCKER_CONFIG" "$ARVAN_MIRROR" <<'PY'
import json
import os
import sys
import tempfile

path, mirror = sys.argv[1], sys.argv[2]

if os.path.exists(path):
    with open(path, encoding="utf-8") as f:
        config = json.load(f)
else:
    config = {}

if not isinstance(config, dict):
    raise SystemExit("Docker config must be a JSON object.")

mirrors = config.get("registry-mirrors", [])

if not isinstance(mirrors, list):
    raise SystemExit("registry-mirrors must be an array.")

if mirror not in mirrors:
    mirrors.append(mirror)

config["registry-mirrors"] = mirrors

fd, temp_path = tempfile.mkstemp(
    prefix=".daemon.json.",
    dir=os.path.dirname(path)
)

try:
    with os.fdopen(fd, "w", encoding="utf-8") as f:
        json.dump(config, f, indent=2)
        f.write("\n")

    os.chmod(temp_path, 0o644)
    os.replace(temp_path, path)

finally:
    if os.path.exists(temp_path):
        os.unlink(temp_path)
PY

# راه‌اندازی مجدد Docker
systemctl restart docker

systemctl is-active --quiet docker ||
    die "Docker پس از تنظیم میرور فعال نشد."

ok "میرور آروان با موفقیت تنظیم شد."

# بررسی نسخه‌ها
log "بررسی نسخه Docker و Compose..."

docker --version
docker compose version
docker buildx version

# ورود به GHCR
log "ورود به GitHub Container Registry"

echo
echo "GitHub Username و Personal Access Token (classic) را وارد کن."
echo "توکن باید مجوز read:packages داشته باشد."
echo "از رمز عبور معمولی GitHub استفاده نکن."
echo

read -r -p "GitHub Username: " GHCR_USER

[[ -n "$GHCR_USER" ]] ||
    die "نام کاربری GitHub خالی است."

read -r -s -p "GitHub Personal Access Token: " GHCR_TOKEN
echo

[[ -n "$GHCR_TOKEN" ]] ||
    die "توکن خالی است."

if ! printf '%s' "$GHCR_TOKEN" | docker login ghcr.io \
    --username "$GHCR_USER" \
    --password-stdin; then

    unset GHCR_TOKEN
    die "ورود به GHCR ناموفق بود. نام کاربری، توکن و مجوز read:packages را بررسی کن."
fi

unset GHCR_TOKEN

ok "ورود به GHCR موفق بود."

# بررسی نهایی
log "بررسی نهایی Docker..."

docker info --format 'Docker Root Dir: {{.DockerRootDir}}'

echo
docker info | sed -n '/Registry Mirrors/,+3p'

echo
docker compose version

echo
ok "نصب Docker با موفقیت به پایان رسید."

echo
echo "Docker:"
docker --version

echo
echo "Docker Compose:"
docker compose version

echo
echo "Arvan Mirror: ${ARVAN_MIRROR}"
echo "Registry: ghcr.io"

echo
echo "حالا وارد پوشه پروژه شو و اجرا کن:"
echo
echo "docker compose pull && docker compose up -d"

echo
echo "نکته: این اسکریپت هیچ کانتینر، Volume یا شبکه‌ای را حذف نمی‌کند."
echo "اعتبارنامه GHCR در /root/.docker/config.json ذخیره می‌شود."
echo "این فایل را در اختیار دیگران قرار نده."
SCRIPT

chmod +x docker-setup.sh
./docker-setup.sh

