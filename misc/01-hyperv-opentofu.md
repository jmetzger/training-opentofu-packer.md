# OpenTofu und Hyper-V – Einschätzung & Alternativen

## Status: OpenTofu/Terraform für Hyper-V

Es existiert **kein brauchbarer Provider** für Hyper-V. Die vorhandenen Community-Provider sind veraltet, unvollständig und nicht produktionsreif. **OpenTofu lohnt sich für Hyper-V aktuell nicht.**

## Empfohlene Alternativen

### 1. Packer + Ansible + PowerShell (empfohlen)

Die solideste Kombination für Hyper-V-Automatisierung:

- **Packer** – Automatisierter Image-Bau (Windows/Linux VHD/VHDX)
- **Ansible** – Konfigurationsmanagement nach dem VM-Deployment
- **PowerShell (Hyper-V-Module)** – VM-Lifecycle (Erstellen, Starten, Stoppen, Netzwerk, Snapshots)

### 2. PowerShell DSC (Desired State Configuration)

- Deklarative Konfiguration direkt von Microsoft
- Gut dokumentiert, nativ in Windows integriert
- Lässt sich in CI/CD-Pipelines einbinden

### 3. Vagrant mit Hyper-V-Provider

- Tauglich für Dev-/Lab-Umgebungen
- Einfache Vagrantfile-Syntax
- **Nicht empfohlen für Produktion**

### 4. SCVMM (System Center Virtual Machine Manager)

- Management-Schicht mit REST-API und PowerShell-Cmdlets
- Sinnvoll, wenn System Center bereits im Einsatz ist
- Hoher Lizenz- und Betriebsaufwand

### 5. Plattformwechsel: Proxmox oder vSphere

- Beide haben **funktionierende OpenTofu/Terraform-Provider**
- Proxmox: kostenlos, gut für Homelabs und KMU
- vSphere: Enterprise-Standard, umfangreiches Tooling

## Fazit

Für Hyper-V-Umgebungen ist **Packer + Ansible + PowerShell** der pragmatische Weg. Wer Infrastructure-as-Code mit OpenTofu nutzen will, sollte auf **Proxmox oder vSphere** als Hypervisor setzen.
