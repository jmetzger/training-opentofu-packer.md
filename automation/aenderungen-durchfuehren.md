# Änderungen durchführen

```
# Änderung in tf-files durchführen 
tofu apply -auto-approve 
# -f 1 -> maschinen nacheinander neu starten  / falls Änderung nicht sofort greift 
ansible loadbalancer -m reboot -b -f 1 -i inventory/hosts.ini
```
