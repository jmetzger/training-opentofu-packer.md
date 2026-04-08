# Übung: Kubernetes-Cluster mit OpenTofu und Ansible auf Proxmox ausrollen

## Ziel

Du erstellst mit OpenTofu einen Kubernetes-Cluster (1 Control Plane, N Worker Nodes) auf Proxmox aus einem Packer-Template. Die VM-IDs folgen dem Schema `3<TLN_NR>0` (Control Plane), `3<TLN_NR>1`, `3<TLN_NR>2`, ... (Worker). OpenTofu generiert automatisch das Ansible-Inventory.

## Voraussetzungen

- Proxmox-Server mit fertigem VM-Template (VM-ID z.B. 9000, erstellt mit Packer)
  - Im Template bereits vorbereitet: containerd, kubelet, kubeadm, kubectl, Kernel-Module, Sysctl, Swap deaktiviert
- OpenTofu installiert (`tofu version`)
- Ansible installiert (`ansible --version`)
- SSH-Keypair vorhanden (`ssh-keygen -t ed25519` falls nicht)

## Variablen für die Übung

| Variable | Beispielwert | Beschreibung |
|----------|-------------|--------------|
| `TLN_NR` | `1` | Teilnehmernummer (1, 2, 3, 4, 5, ...) |
| `PROXMOX_URL` | `https://176.9.38.183:8006` | Proxmox API-URL |
| `PROXMOX_USER` | `root@pam` | Proxmox Benutzer |
| `PROXMOX_PASS` | `geheim` | Proxmox Passwort |

---

## Schritt 1: Variablen setzen

Passe die Werte an deine Umgebung an:

```bash
export TLN_NR="1"
export PROXMOX_URL="https://176.9.38.183:8006"
export PROXMOX_USER="root@pam"
export PROXMOX_PASS="geheim"
```

---

## Schritt 2: Projektstruktur anlegen

```bash
mkdir -p ~/opentofu-k8s/templates
mkdir -p ~/opentofu-k8s/inventory
mkdir -p ~/opentofu-k8s/ansible
cd ~/opentofu-k8s
```

---

## Schritt 3: Provider und VMs (main.tf)

```bash
cat > main.tf <<'EOF'
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.78"
    }
  }
}

provider "proxmox" {
  endpoint = var.proxmox_url
  username = var.proxmox_user
  password = var.proxmox_password
  insecure = true
}

# --- Control Plane: VM-ID = 3<TLN_NR>0 ---
resource "proxmox_virtual_environment_vm" "controlplane" {
  name      = "k8s-tln${var.tln_nr}-cp"
  node_name = var.proxmox_node
  vm_id     = tonumber("3${var.tln_nr}0")

  clone {
    vm_id = var.template_vm_id
    full  = true
  }

  cpu {
    cores = 2
    type  = "host"
  }

  memory {
    dedicated = 4096
  }

  disk {
    datastore_id = var.datastore
    interface    = "scsi0"
    size         = 40
  }

  network_device {
    bridge = var.bridge
  }

  initialization {
    ip_config {
      ipv4 {
        address = "${var.cp_ip}/24"
        gateway = var.gateway
      }
    }
    dns {
      servers = [var.dns_server]
    }
    user_account {
      keys     = [file(var.ssh_public_key_path)]
      username = var.vm_user
    }
  }
}

# --- Worker Nodes: VM-ID = 3<TLN_NR>1, 3<TLN_NR>2, ... ---
resource "proxmox_virtual_environment_vm" "worker" {
  count     = var.worker_count
  name      = "k8s-tln${var.tln_nr}-worker${count.index + 1}"
  node_name = var.proxmox_node
  vm_id     = tonumber("3${var.tln_nr}${count.index + 1}")

  clone {
    vm_id = var.template_vm_id
    full  = true
  }

  cpu {
    cores = 2
    type  = "host"
  }

  memory {
    dedicated = 4096
  }

  disk {
    datastore_id = var.datastore
    interface    = "scsi0"
    size         = 40
  }

  network_device {
    bridge = var.bridge
  }

  initialization {
    ip_config {
      ipv4 {
        address = "${cidrhost("${var.worker_base_ip}/24", count.index)}/24"
        gateway = var.gateway
      }
    }
    dns {
      servers = [var.dns_server]
    }
    user_account {
      keys     = [file(var.ssh_public_key_path)]
      username = var.vm_user
    }
  }
}
EOF
```

**VM-ID-Schema (Beispiele):**

| TLN_NR | Control Plane | Worker 1 | Worker 2 |
|--------|--------------|----------|----------|
| 1 | **310** | **311** | **312** |
| 2 | **320** | **321** | **322** |
| 5 | **350** | **351** | **352** |

**IP-Schema (Beispiele):**

| TLN_NR | CP | Worker 1 | Worker 2 |
|--------|-----|----------|----------|
| 1 | 10.10.10.**111** | 10.10.10.**112** | 10.10.10.**113** |
| 2 | 10.10.10.**121** | 10.10.10.**122** | 10.10.10.**123** |
| 5 | 10.10.10.**151** | 10.10.10.**152** | 10.10.10.**153** |

---

## Schritt 4: Variablen (variables.tf)

```bash
cat > variables.tf <<'EOF'
variable "tln_nr" {
  description = "Teilnehmernummer (1, 2, 3, ...)"
  type        = string
}

variable "proxmox_url" {
  type = string
}

variable "proxmox_user" {
  type    = string
  default = "root@pam"
}

variable "proxmox_password" {
  type      = string
  sensitive = true
}

variable "proxmox_node" {
  type    = string
  default = "pve"
}

variable "template_vm_id" {
  type    = number
  default = 9000
}

variable "datastore" {
  type    = string
  default = "local"
}

variable "worker_count" {
  description = "Anzahl Worker Nodes"
  type        = number
  default     = 1
}

variable "cp_ip" {
  description = "Statische IP des Control Plane"
  type        = string
}

variable "worker_base_ip" {
  description = "Erste Worker-IP (wird hochgezählt)"
  type        = string
}

variable "gateway" {
  type = string
}

variable "dns_server" {
  type = string
}

variable "vm_user" {
  type    = string
  default = "trainee"
}

variable "bridge" {
  description = "Proxmox Network Bridge"
  type        = string
  default     = "vmbr0"
}

variable "ssh_public_key_path" {
  type    = string
  default = "~/.ssh/id_ed25519.pub"
}
EOF
```

---

## Schritt 5: Inventory-Template (templates/inventory.tpl)

```bash
cat > templates/inventory.tpl <<'EOF'
[controlplane]
${cp_name} ansible_host=${cp_ip}

[workers]
%{ for w in workers ~}
${w.name} ansible_host=${w.ip}
%{ endfor ~}

[k8s_cluster:children]
controlplane
workers

[k8s_cluster:vars]
ansible_user=${vm_user}
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
EOF
```

---

## Schritt 6: Outputs und Inventory-Generierung (outputs.tf)

```bash
cat > outputs.tf <<'EOF'
resource "local_file" "ansible_inventory" {
  content = templatefile("${path.module}/templates/inventory.tpl", {
    cp_name = proxmox_virtual_environment_vm.controlplane.name
    cp_ip   = var.cp_ip
    workers = [
      for i, vm in proxmox_virtual_environment_vm.worker : {
        name = vm.name
        ip   = cidrhost("${var.worker_base_ip}/24", i)
      }
    ]
    vm_user = var.vm_user
  })
  filename        = "${path.module}/inventory/hosts.ini"
  file_permission = "0644"

  depends_on = [
    proxmox_virtual_environment_vm.controlplane,
    proxmox_virtual_environment_vm.worker
  ]
}

output "controlplane_ip" {
  value = var.cp_ip
}

output "worker_ips" {
  value = [
    for i in range(var.worker_count) :
    cidrhost("${var.worker_base_ip}/24", i)
  ]
}

output "vm_ids" {
  value = {
    controlplane = proxmox_virtual_environment_vm.controlplane.vm_id
    workers      = [for vm in proxmox_virtual_environment_vm.worker : vm.vm_id]
  }
}

output "inventory_path" {
  value = local_file.ansible_inventory.filename
}
EOF
```

---

## Schritt 7: tfvars generieren

```bash
cat > terraform.tfvars <<EOF
tln_nr           = "${TLN_NR}"
proxmox_url      = "${PROXMOX_URL}"
proxmox_user     = "${PROXMOX_USER}"
proxmox_password = "${PROXMOX_PASS}"
proxmox_node     = "pve"
template_vm_id   = 9000

worker_count    = 1
cp_ip           = "10.10.10.1${TLN_NR}1"
worker_base_ip  = "10.10.10.1${TLN_NR}2"
gateway         = "10.10.10.1"
dns_server      = "8.8.8.8"

vm_user             = "trainee"
bridge              = "vmbr1"
ssh_public_key_path = "~/.ssh/id_ed25519.pub"
EOF
```

**Hinweis:** `terraform.tfvars` enthält Passwörter — in `.gitignore` aufnehmen!

---

## Schritt 8: Ansible-Playbook (ansible/site.yml)

Da das Template bereits vollständig vorbereitet ist, übernimmt Ansible nur noch das Cluster-Bootstrapping mit Calico als CNI:

```bash
cat > ansible/site.yml <<'EOF'
---
# --- Control Plane initialisieren ---
- name: Control Plane initialisieren
  hosts: controlplane
  become: true
  tasks:

    - name: kubeadm init (mit Calico Pod-CIDR)
      command: >
        kubeadm init
        --pod-network-cidr=192.168.0.0/16
        --apiserver-advertise-address={{ ansible_host }}
      args:
        creates: /etc/kubernetes/admin.conf

    - name: kubeconfig für User einrichten
      shell: |
        mkdir -p /home/{{ ansible_user }}/.kube
        cp /etc/kubernetes/admin.conf /home/{{ ansible_user }}/.kube/config
        chown -R {{ ansible_user }}:{{ ansible_user }} /home/{{ ansible_user }}/.kube

    - name: Calico Operator installieren
      become: false
      command: kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/tigera-operator.yaml
      register: calico_operator
      changed_when: calico_operator.rc == 0
      failed_when: calico_operator.rc != 0 and 'AlreadyExists' not in calico_operator.stderr

    - name: Calico Custom Resources installieren
      become: false
      command: kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/custom-resources.yaml
      register: calico_cr
      changed_when: calico_cr.rc == 0
      failed_when: calico_cr.rc != 0 and 'AlreadyExists' not in calico_cr.stderr

    - name: Warten bis Calico-Pods ready sind
      become: false
      command: kubectl wait --for=condition=Ready pods --all -n calico-system --timeout=120s

    - name: Join-Command erzeugen
      command: kubeadm token create --print-join-command
      register: join_command

    - name: Join-Command als Fact setzen
      set_fact:
        k8s_join_command: "{{ join_command.stdout }}"

# --- Alle Worker joinen ---
- name: Worker dem Cluster hinzufügen
  hosts: workers
  become: true
  tasks:

    - name: Worker joinen
      command: "{{ hostvars[groups['controlplane'][0]].k8s_join_command }}"
      args:
        creates: /etc/kubernetes/kubelet.conf
EOF
```

**Hinweis zu Calico:**
- `--pod-network-cidr=192.168.0.0/16` ist der Calico-Default
- Die `custom-resources.yaml` von Calico verwendet genau dieses CIDR
- Calico-Version (hier v3.29.3) ggf. an aktuelle Version anpassen

---

## Schritt 9: OpenTofu ausführen

```bash
tofu init
tofu plan
tofu apply
```

### Ergebnis prüfen

```bash
cat inventory/hosts.ini
```

Erwartete Ausgabe (bei TLN_NR=1, WORKER_COUNT=1):

```ini
[controlplane]
k8s-tln1-cp ansible_host=10.10.10.111

[workers]
k8s-tln1-worker1 ansible_host=10.10.10.112

[k8s_cluster:children]
controlplane
workers

[k8s_cluster:vars]
ansible_user=trainee
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

```bash
# SSH-Erreichbarkeit testen
ansible -i inventory/hosts.ini k8s_cluster -m ping
```

---

## Schritt 10: Ansible-Playbook ausführen

```bash
ansible-playbook -i inventory/hosts.ini ansible/site.yml
```

---

## Schritt 11: Cluster prüfen

```bash
ssh trainee@${CP_IP} kubectl get nodes
```

Erwartete Ausgabe:

```
NAME              STATUS   ROLES           AGE   VERSION
k8s-tln1-cp         Ready    control-plane   5m    v1.31.x
k8s-tln1-worker1    Ready    <none>          3m    v1.31.x
```

Calico-Status prüfen:

```bash
ssh trainee@${CP_IP} kubectl get pods -n calico-system
```

---

## Schritt 12: Aufräumen

```bash
tofu destroy
```

---

## Optional: Alles in einem Script (deploy.sh)

```bash
cat > deploy.sh <<'SCRIPT'
#!/bin/bash
set -e

echo "=== Step 1: OpenTofu - VMs erstellen ==="
tofu init
tofu apply -auto-approve

echo ""
echo "=== Warte 30s auf VM-Boot ==="
sleep 30

INVENTORY="$(tofu output -raw inventory_path)"
echo "Inventory: $INVENTORY"
cat "$INVENTORY"

echo ""
echo "=== Step 2: Ansible - Cluster mit Calico bootstrappen ==="
ansible-playbook -i "$INVENTORY" ansible/site.yml

echo ""
echo "=== Fertig! Cluster prüfen: ==="
echo "ssh trainee@$(tofu output -raw controlplane_ip) kubectl get nodes"
SCRIPT

chmod +x deploy.sh
```
