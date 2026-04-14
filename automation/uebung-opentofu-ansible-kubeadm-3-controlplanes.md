cat > ansible/site.yml <<'EOF'
---
# =============================================================
# 1) HAProxy + Keepalived auf den Load Balancer Nodes
# =============================================================
- name: HAProxy und Keepalived einrichten
  hosts: loadbalancer
  become: true
  vars:
    cp_hosts: "{{ groups['controlplane'] | map('extract', hostvars, 'ansible_host') | list }}"
  tasks:

    - name: HAProxy und Keepalived installieren
      apt:
        name:
          - haproxy
          - keepalived
        state: present
        update_cache: true

    - name: HAProxy konfigurieren
      copy:
        dest: /etc/haproxy/haproxy.cfg
        content: |
          global
              log /dev/log local0 warning
              chroot /var/lib/haproxy
              pidfile /var/run/haproxy.pid
              maxconn 4000
              user haproxy
              group haproxy
              daemon
              stats socket /var/lib/haproxy/stats

          defaults
              log     global
              option  tcplog
              option  dontlognull
              timeout connect 5000
              timeout client  50000
              timeout server  50000

          frontend k8s-api
              bind *:6443
              mode tcp
              default_backend k8s-api

          backend k8s-api
              mode tcp
              option tcp-check
              balance roundrobin
              default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
          {% for host in cp_hosts %}
              server cp{{ loop.index }} {{ host }}:6443 check
          {% endfor %}
      notify: haproxy restart

    - name: Keepalived Health-Check Script erstellen
      copy:
        dest: /etc/keepalived/check_apiserver.sh
        mode: '0755'
        content: |
          #!/bin/sh
          /usr/bin/killall -0 haproxy

    - name: Keepalived konfigurieren
      copy:
        dest: /etc/keepalived/keepalived.conf
        content: |
          vrrp_script chk_haproxy {
              script "/etc/keepalived/check_apiserver.sh"
              interval 3
              weight -2
              fall 10
              rise 2
          }

          vrrp_instance haproxy-vip {
              state {{ 'MASTER' if inventory_hostname == groups['loadbalancer'][0] else 'BACKUP' }}
              priority {{ '200' if inventory_hostname == groups['loadbalancer'][0] else '100' }}
              interface eth0
              virtual_router_id 51
              advert_int 1
              authentication {
                  auth_type PASS
                  auth_pass kubernetes
              }
              virtual_ipaddress {
                  {{ vip }}
              }
              track_script {
                  chk_haproxy
              }
          }
      notify: keepalived restart

    - name: Services aktivieren
      systemd:
        name: "{{ item }}"
        enabled: true
        state: started
      loop:
        - haproxy
        - keepalived

  handlers:
    - name: haproxy restart
      systemd:
        name: haproxy
        state: restarted

    - name: keepalived restart
      systemd:
        name: keepalived
        state: restarted

# =============================================================
# 2) Ersten Control Plane Node initialisieren
# =============================================================
- name: Ersten Control Plane initialisieren
  hosts: controlplane[0]
  become: true
  tasks:

    - name: kubeadm init mit HA-Endpoint
      command: >
        kubeadm init
        --control-plane-endpoint "{{ vip }}:6443"
        --upload-certs
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
      command: kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml
      register: calico_operator
      changed_when: calico_operator.rc == 0
      failed_when: calico_operator.rc != 0 and 'AlreadyExists' not in calico_operator.stderr

    - name: Warten bis Calico CRDs registriert sind
      become: false
      command: kubectl wait --for=condition=Established crd/installations.operator.tigera.io --timeout=30s
      retries: 10
      delay: 10
      register: crd_wait
      until: crd_wait.rc == 0

    - name: Calico Custom Resources installieren
      become: false
      command: kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml
      register: calico_cr
      changed_when: calico_cr.rc == 0
      failed_when: calico_cr.rc != 0 and 'AlreadyExists' not in calico_cr.stderr

    - name: Warten bis Calico-Node Pods ready sind
      become: false
      shell: kubectl wait --for=condition=Ready pods -l k8s-app=calico-node -n calico-system --timeout=180s
      retries: 5
      delay: 15
      register: calico_wait
      until: calico_wait.rc == 0

    - name: Certificate-Key erzeugen
      command: kubeadm init phase upload-certs --upload-certs
      register: cert_key_output

    - name: Certificate-Key extrahieren
      set_fact:
        cert_key: "{{ cert_key_output.stdout_lines | last }}"

    - name: Worker-Join-Command erzeugen
      command: kubeadm token create --print-join-command
      register: join_command

    - name: Join-Commands als Facts setzen
      set_fact:
        k8s_join_command: "{{ join_command.stdout }}"
        k8s_cp_join_command: "{{ join_command.stdout }} --control-plane --certificate-key {{ cert_key }}"

# =============================================================
# 3) Weitere Control Plane Nodes joinen
# =============================================================
- name: Weitere Control Planes joinen
  hosts: controlplane[1:]
  become: true
  serial: 1
  tasks:

    - name: Als Control Plane joinen
      command: >
        {{ hostvars[groups['controlplane'][0]].k8s_cp_join_command }}
        --apiserver-advertise-address={{ ansible_host }}
      args:
        creates: /etc/kubernetes/admin.conf

    - name: kubeconfig für User einrichten
      shell: |
        mkdir -p /home/{{ ansible_user }}/.kube
        cp /etc/kubernetes/admin.conf /home/{{ ansible_user }}/.kube/config
        chown -R {{ ansible_user }}:{{ ansible_user }} /home/{{ ansible_user }}/.kube

# =============================================================
# 4) kubeconfig lokal kopieren (zeigt auf VIP)
# =============================================================
- name: kubeconfig lokal bereitstellen
  hosts: controlplane[0]
  become: true
  tasks:

    - name: kubeconfig vom Control Plane holen
      fetch:
        src: /etc/kubernetes/admin.conf
        dest: /tmp/kubeconfig
        flat: true

    - name: Lokales .kube Verzeichnis erstellen
      delegate_to: localhost
      become: false
      file:
        path: "{{ lookup('env', 'HOME') }}/.kube"
        state: directory
        mode: '0700'

    - name: kubeconfig nach ~/.kube/config kopieren
      delegate_to: localhost
      become: false
      copy:
        src: /tmp/kubeconfig
        dest: "{{ lookup('env', 'HOME') }}/.kube/config"
        mode: '0600'

    - name: Temp-Datei aufräumen
      delegate_to: localhost
      become: false
      file:
        path: /tmp/kubeconfig
        state: absent

# =============================================================
# 5) Worker Nodes joinen
# =============================================================
- name: Worker dem Cluster hinzufügen
  hosts: workers
  become: true
  tasks:

    - name: Worker joinen
      command: "{{ hostvars[groups['controlplane'][0]].k8s_join_command }}"
      args:
        creates: /etc/kubernetes/kubelet.conf
EOF
