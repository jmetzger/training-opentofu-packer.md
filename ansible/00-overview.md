# Was ist ansible ? 

- Open-Source-Automatisierungstool von Red Hat
- Configuration Management, Application Deployment, Orchestrierung
- Agentless – arbeitet über SSH
- Deklarativ via YAML-Playbooks
- Idempotent
- Push-basiert
- Inventory definiert die Zielhosts
- Module führen Aktionen aus (z.B. `apt`, `copy`, `template`, `kubernetes.core.*`)
- Typischer Einsatz in deinem Stack: Provisionierung nach Packer/OpenTofu – OS-Konfiguration, Package-Installation, kubeadm-Bootstrap
