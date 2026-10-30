## Kubernetes en rocky

#### Introduccción

Este playbook, está adaptado para trabajar con kubernetes en un solo nodo, al disponer de una sola máquina virtual.

También toma como base la configuración  que ta se tenia para los ejercicios de la otra tarea

- mismo nombre de host nodo1

- usuario ansible y la configuración para conexión por ssh usando claves.

Se crea la carpeta de trabajo

**/root/devops2026/ansible-kubernetes**

Y dentro de ella los siguientes archivos

```shellsession
[root@localhost ansible-kubernetes]# tree
.
├── inventario_local.ini
└── k8s_single_node.yml

0 directories, 2 files
```

Con sus respectivos contendidos

inventario_local.ini

```ini
[k8s_single]
nodo1 ansible_connection=local
```

k8s_sibgle_node.yml

```yml
[root@localhost ansible-kubernetes]# cat inventario_local.ini 
[k8s_single]
nodo1 ansible_connection=local
[root@localhost ansible-kubernetes]# car k8s_single_node.yml 
-bash: car: orden no encontrada
[root@localhost ansible-kubernetes]# cat k8s_single_node.yml 
---
- name: "Despliegue de clúster Kubernetes Single-Node en Rocky Linux 9"
  hosts: k8s_single
  become: true
  vars:
    k8s_version: "v1.28"
    pod_network_cidr: "10.244.0.0/16" # CIDR requerido por Flannel CNI
    ansible_user: "ansible"

  tasks:
    # --------------------------------------------------------------------------
    # Paso 1: Hostname y resolución de nombres en /etc/hosts
    # --------------------------------------------------------------------------
    - name: "Paso 1 · Configurar hostname del nodo único"
      ansible.builtin.hostname:
        name: "{{ inventory_hostname }}"

    - name: "Paso 1 · Configurar /etc/hosts con el hostname local"
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "127.0.0.1   {{ inventory_hostname }} localhost"
        state: present

    # --------------------------------------------------------------------------
    # Paso 2: Desactivar memoria Swap
    # --------------------------------------------------------------------------
    - name: "Paso 2 · Desactivar swap en caliente"
      ansible.builtin.command: swapoff -a
      when: ansible_facts['swaptotal_mb'] > 0
      changed_when: true

    - name: "Paso 2 · Comentar swap en /etc/fstab"
      ansible.builtin.lineinfile:
        path: /etc/fstab
        regexp: '^(\s*[^#\s]+\s+\S+\s+swap\s+.*)$'
        line: '# \1'
        backrefs: true

    # --------------------------------------------------------------------------
    # Paso 3: SELinux y Firewall (Unificado para nodo único)
    # --------------------------------------------------------------------------
    - name: "Paso 3 · Establecer SELinux en modo permissive"
      ansible.builtin.command: setenforce 0
      when: ansible_facts['selinux']['status'] == "enabled" and ansible_facts['selinux']['mode'] != "permissive"
      changed_when: true

    - name: "Paso 3 · Persistir SELINUX=permissive en /etc/selinux/config"
      ansible.builtin.lineinfile:
        path: /etc/selinux/config
        regexp: '^SELINUX='
        line: 'SELINUX=permissive'

    - name: "Paso 3 · Abrir puertos en firewall (Master + Worker unificados)"
      ansible.builtin.command: >
        firewall-cmd --permanent --add-port={{ item }}
      loop:
        - "6443/tcp"
        - "2379-2380/tcp"
        - "10250-10252/tcp"
        - "10257-10259/tcp"
        - "30000-32767/tcp"
        - "4789/udp"
      register: fw_ports
      changed_when: "'ALREADY_ENABLED' not in fw_ports.stderr"

    - name: "Paso 3 · Recargar firewalld"
      ansible.builtin.command: firewall-cmd --reload
      changed_when: false

    # --------------------------------------------------------------------------
    # Paso 4: Módulos de Kernel y parámetros de red
    # --------------------------------------------------------------------------
    - name: "Paso 4 · Configurar módulos overlay y br_netfilter"
      ansible.builtin.copy:
        dest: /etc/modules-load.d/containerd.conf
        content: |
          overlay
          br_netfilter
        mode: '0644'

    - name: "Paso 4 · Cargar módulos de kernel en caliente"
      ansible.builtin.command: modprobe {{ item }}
      loop:
        - overlay
        - br_netfilter
      changed_when: false

    - name: "Paso 4 · Configurar parámetros de red en sysctl"
      ansible.builtin.copy:
        dest: /etc/sysctl.d/99-kubernetes-cri.conf
        content: |
          net.bridge.bridge-nf-call-iptables = 1
          net.ipv4.ip_forward = 1
          net.bridge.bridge-nf-call-ip6tables = 1
        mode: '0644'

    - name: "Paso 4 · Aplicar parámetros de sysctl inmediatamente"
      ansible.builtin.command: sysctl --system
      changed_when: false

    # --------------------------------------------------------------------------
    # Paso 5: Container Runtime (containerd)
    # --------------------------------------------------------------------------
    - name: "Paso 5 · Agregar repositorio Docker CE"
      ansible.builtin.yum_repository:
        name: docker-ce-stable
        description: Docker CE Stable - CentOS
        baseurl: https://download.docker.com/linux/centos/9/$basearch/stable
        gpgcheck: true
        gpgkey: https://download.docker.com/linux/centos/gpg
        enabled: true

    - name: "Paso 5 · Instalar containerd.io"
      ansible.builtin.dnf:
        name: containerd.io
        state: present

    - name: "Paso 5 · Generar configuración por defecto de containerd si no existe"
      ansible.builtin.shell: |
        mkdir -p /etc/containerd
        containerd config default > /etc/containerd/config.toml
      args:
        creates: /etc/containerd/config.toml

    - name: "Paso 5 · Habilitar SystemdCgroup en containerd"
      ansible.builtin.lineinfile:
        path: /etc/containerd/config.toml
        regexp: '^\s*SystemdCgroup\s*=.*'
        line: '            SystemdCgroup = true'

    - name: "Paso 5 · Asegurar que containerd esté activo y reiniciado"
      ansible.builtin.service:
        name: containerd
        state: restarted
        enabled: true

    # --------------------------------------------------------------------------
    # Paso 6: Herramientas Kubernetes (kubelet, kubeadm, kubectl)
    # --------------------------------------------------------------------------
    - name: "Paso 6 · Configurar repositorio oficial de Kubernetes"
      ansible.builtin.copy:
        dest: /etc/yum.repos.d/kubernetes.repo
        content: |
          [kubernetes]
          name=Kubernetes
          baseurl=https://pkgs.k8s.io/core:/stable:/{{ k8s_version }}/rpm/
          enabled=1
          gpgcheck=1
          gpgkey=https://pkgs.k8s.io/core:/stable:/{{ k8s_version }}/rpm/repodata/repomd.xml.key
          exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
        mode: '0644'

    - name: "Paso 6 · Instalar kubelet, kubeadm y kubectl"
      ansible.builtin.dnf:
        name:
          - kubelet
          - kubeadm
          - kubectl
        disable_excludes: kubernetes
        state: present

    - name: "Paso 6 · Habilitar e iniciar servicio kubelet"
      ansible.builtin.service:
        name: kubelet
        enabled: true
        state: started

    # --------------------------------------------------------------------------
    # Paso 7: Inicialización del Clúster y ajuste Single-Node
    # --------------------------------------------------------------------------
    - name: "Paso 7 · Inicializar clúster con kubeadm (con pod-network-cidr)"
      ansible.builtin.command: >
        kubeadm init --pod-network-cidr={{ pod_network_cidr }}
      args:
        creates: /etc/kubernetes/admin.conf

    - name: "Paso 7 · Crear directorio .kube para el usuario {{ ansible_user }}"
      ansible.builtin.file:
        path: "/home/{{ ansible_user }}/.kube"
        state: directory
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0750'

    - name: "Paso 7 · Copiar admin.conf a la ruta local del usuario"
      ansible.builtin.copy:
        src: /etc/kubernetes/admin.conf
        dest: "/home/{{ ansible_user }}/.kube/config"
        remote_src: true
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0600'

    - name: "Paso 7 · Instalar plugin CNI (Flannel) para que el nodo pase a Ready"
      ansible.builtin.command: >
        kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
      become: false
      environment:
        KUBECONFIG: "/home/{{ ansible_user }}/.kube/config"
      register: cni_result
      changed_when: "'created' in cni_result.stdout or 'configured' in cni_result.stdout"

    - name: "Paso 7 · Remover Taint del nodo maestro para permitir cargas de trabajo"
      ansible.builtin.command: >
        kubectl taint nodes --all node-role.kubernetes.io/control-plane-
      become: false
      environment:
        KUBECONFIG: "/home/{{ ansible_user }}/.kube/config"
      register: taint_result
      failed_when: false
      changed_when: "'untainted' in taint_result.stdout"

```



#### Ejecución del playbook

```shellsession
[root@localhost ansible-kubernetes]# ansible-playbook -i inventario_local.ini k8s_single_node.yml

PLAY [Despliegue de clúster Kubernetes Single-Node en Rocky Linux 9] ***********************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 1 · Configurar hostname del nodo único] *****************************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 1 · Configurar /etc/hosts con el hostname local] ********************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 2 · Desactivar swap en caliente] ************************************************************************************************************************************************************************
skipping: [nodo1]

TASK [Paso 2 · Comentar swap en /etc/fstab] ************************************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 3 · Establecer SELinux en modo permissive] **************************************************************************************************************************************************************
skipping: [nodo1]

TASK [Paso 3 · Persistir SELINUX=permissive en /etc/selinux/config] ************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 3 · Abrir puertos en firewall (Master + Worker unificados)] *********************************************************************************************************************************************
ok: [nodo1] => (item=6443/tcp)
ok: [nodo1] => (item=2379-2380/tcp)
ok: [nodo1] => (item=10250-10252/tcp)
ok: [nodo1] => (item=10257-10259/tcp)
ok: [nodo1] => (item=30000-32767/tcp)
ok: [nodo1] => (item=4789/udp)

TASK [Paso 3 · Recargar firewalld] *********************************************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 4 · Configurar módulos overlay y br_netfilter] **********************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 4 · Cargar módulos de kernel en caliente] ***************************************************************************************************************************************************************
ok: [nodo1] => (item=overlay)
ok: [nodo1] => (item=br_netfilter)

TASK [Paso 4 · Configurar parámetros de red en sysctl] *************************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 4 · Aplicar parámetros de sysctl inmediatamente] ********************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 5 · Agregar repositorio Docker CE] **********************************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 5 · Instalar containerd.io] *****************************************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 5 · Generar configuración por defecto de containerd si no existe] ***************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 5 · Habilitar SystemdCgroup en containerd] **************************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 5 · Asegurar que containerd esté activo y reiniciado] ***************************************************************************************************************************************************
changed: [nodo1]

TASK [Paso 6 · Configurar repositorio oficial de Kubernetes] *******************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 6 · Instalar kubelet, kubeadm y kubectl] ****************************************************************************************************************************************************************
ok: [nodo1]

TASK [Paso 6 · Habilitar e iniciar servicio kubelet] ***************************************************************************************************************************************************************
changed: [nodo1]

TASK [Paso 7 · Inicializar clúster con kubeadm (con pod-network-cidr)] *********************************************************************************************************************************************
changed: [nodo1]

TASK [Paso 7 · Crear directorio .kube para el usuario ansible] *****************************************************************************************************************************************************
changed: [nodo1]

TASK [Paso 7 · Copiar admin.conf a la ruta local del usuario] ******************************************************************************************************************************************************
changed: [nodo1]

TASK [Paso 7 · Instalar plugin CNI (Flannel) para que el nodo pase a Ready] ****************************************************************************************************************************************
changed: [nodo1]

TASK [Paso 7 · Remover Taint del nodo maestro para permitir cargas de trabajo] *************************************************************************************************************************************
changed: [nodo1]

PLAY RECAP *********************************************************************************************************************************************************************************************************
nodo1                      : ok=24   changed=7    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   


```

### Verificar que esté corriendo kubernetes

```shellsession
[martin@localhost ~]$ su -l ansible
[ansible@nodo1 ~]$ kubectl get nodes
NAME    STATUS   ROLES           AGE     VERSION
nodo1   Ready    control-plane   5h33m   v1.28.15

```

Podemos ver que kuberctl responde.


