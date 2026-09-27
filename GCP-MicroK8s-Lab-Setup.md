Инструкция: Настройка на GCP VM за MicroK8s & Docker (Стандартен SSH)
1. Параметри на виртуалната машина (Compute Engine)
При създаване на инстанцията изберете:

OS: Ubuntu 22.04 LTS или 24.04 LTS

Machine Type: e2-standard-2 (2 vCPU, 8 GB RAM)

Boot Disk: 30 GB - 50 GB Balanced Persistent Disk

Firewall: Отметнете Allow HTTP traffic и Allow HTTPS traffic



2. GCP Firewall Rule (За NodePort услуги)
Изпълнете през Google Cloud Shell или локален gcloud CLI:

gcloud compute firewall-rules create allow-k8s-nodeports \
    --allow tcp:30000-32767 \
    --source-ranges 0.0.0.0/0 \
    --description "Allow Kubernetes NodePort services for demo"
	
	
. Автоматизиран скрипт за инсталация (gcp-setup.sh)
Свържете се през SSH и създайте файла:

cat << 'EOF' > gcp-setup.sh
#!/bin/bash
set -e

# Ограничаване на интерактивните въпроси от apt
export DEBIAN_FRONTEND=noninteractive

TARGET_USER=${SUDO_USER:-$USER}

echo "=== 0. Корекция на Hostname за MicroK8s (Max 64 символа) ==="
NEW_HOSTNAME="gcp-k8s-node"
sudo hostnamectl set-hostname "$NEW_HOSTNAME"

# Предотвратяване на връщането на стария hostname от GCP Guest Agent
if [ -f /etc/default/instance_configs.cfg ]; then
    sudo sed -i 's/set_hostname = true/set_hostname = false/g' /etc/default/instance_configs.cfg
fi

# Актуализиране на /etc/hosts
if ! grep -q "$NEW_HOSTNAME" /etc/hosts; then
    sudo sed -i "1s/^/127.0.0.1 $NEW_HOSTNAME\n/" /etc/hosts
fi

echo "=== 1. Подготовка на GCP VM и основни пакети ==="
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https iptables

echo "=== 2. Добавяне на Docker & Kubernetes Repositories ==="
sudo install -m 0755 -d /etc/apt/keyrings

# Docker GPG & Repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg --yes
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Kubernetes GPG & Repo (v1.30)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg --yes
sudo chmod a+r /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

echo "=== 3. Инсталиране на Docker Engine & kubectl ==="
sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io kubectl

echo "=== 4. Инсталиране на MicroK8s ==="
sudo snap install microk8s --classic --channel=1.30/stable

echo "=== 5. Конфигуриране на правата за потребителя: $TARGET_USER ==="
sudo usermod -aG docker "$TARGET_USER"
sudo usermod -aG microk8s "$TARGET_USER"

TARGET_HOME=$(eval echo ~"$TARGET_USER")
mkdir -p "$TARGET_HOME/.kube"

echo "=== 6. Настройка на MicroK8s за GCP ==="
# Изчакване на клъстера
sudo microk8s status --wait-ready

# Активиране на нужните добавки (DNS & Storage)
sudo microk8s enable dns storage

# Конфигуриране на kubectl (генериране на kubeconfig за потребителя)
sudo microk8s config > "$TARGET_HOME/.kube/config"
sudo chown -R "$TARGET_USER:$TARGET_USER" "$TARGET_HOME/.kube"
chmod 600 "$TARGET_HOME/.kube/config"

echo "=== 7. Разрешаване на IP Forwarding ==="
sudo sysctl net.ipv4.ip_forward=1
if ! grep -q "net.ipv4.ip_forward=1" /etc/sysctl.conf; then
    echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
fi

echo "-------------------------------------------------------------------"
echo " ✅ Успешна инсталация в GCP!"
echo " ⚠️  ВАЖНО: За да се приложат новите групи (docker & microk8s),"
echo "     излезте и се свържете отново през SSH"
echo "     exit"
echo "     Click the SSH button next to your Virtual Machine instance in the Google Cloud Console"
echo "-------------------------------------------------------------------"
EOF

chmod +x gcp-setup.sh
./gcp-setup.sh


4. Финална проверка
След като скриптът приключи, най-добрият и чист начин да активирате новите групи е просто да влезете наново през SSH:


# Излизане от настоящата сесия
exit

# Ново влизане през SSH
Click the SSH button next to your Virtual Machine instance in the Google Cloud Console


Проверете статуса на средата:

# Проверка на групите
groups

# Проверка на Kubernetes
kubectl get nodes
kubectl get pods -A

