# Installation ansible 


Für Ubuntu 24.04: **pip** ist der beste Weg.

Das Ubuntu-Paket (`apt`) hinkt oft mehrere Versionen hinterher. Mit pip bekommst du immer die aktuelle Version und kannst gezielt upgraden.

```bash
sudo su -
```

```bash 
apt install -y python3-pip python3-venv
pip install ansible --break-system-packages
```

Upgrade später einfach mit:

```bash
pip install --upgrade ansible --break-system-packages
```

Die offizielle Ansible-Doku empfiehlt ebenfalls pip als bevorzugten Installationsweg.
