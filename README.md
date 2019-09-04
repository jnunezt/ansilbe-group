#  Provision de un Docker Swarm Cluster con Vagrant y Ansible

### Aprovisione automáticamente un clúster Docker Swarm compuesto por dos maestros y dos trabajadores

## Precondiciones:

* Linux, Windows or macOS host with at least 8GB RAM
* VirtualBox - https://www.virtualbox.org/
* Vagrant - https://www.vagrantup.com/

Para poner en marcha este proyecto, sólo necesita tener instalado Vagrant en su ordenador o laptop.


## Code Overview

A traves de vagrant (_Vagrantfile_) aprovisionamos cuatro máquinas Linux idénticas (Ubuntu 16.04 LTS) en la misma subred:


```ruby
nodes = [
  { :hostname => 'swarm-worker-1', :ip => '192.168.77.12', :ram => 1024, :cpus => 1 },
  { :hostname => 'swarm-worker-2', :ip => '192.168.77.13', :ram => 1024, :cpus => 1 },
  { :hostname => 'swarm-master-2', :ip => '192.168.77.11', :ram => 1024, :cpus => 1 },
  { :hostname => 'swarm-master-1', :ip => '192.168.77.10', :ram => 1024, :cpus => 1 }
]

Vagrant.configure("2") do |config|
  # Always use Vagrant's insecure key
  config.ssh.insert_key = false
  # Forward ssh agent to easily ssh into the different machines
  config.ssh.forward_agent = true
  # Provision nodes
  nodes.each do |node|
    config.vm.define node[:hostname] do |nodeconfig|
      nodeconfig.vm.box = "bento/ubuntu-16.04";
      nodeconfig.vm.hostname = node[:hostname] + ".box"
      nodeconfig.vm.network :private_network, ip: node[:ip]
      memory = node[:ram] ? node[:ram] : 1024;
      cpus = node[:cpus] ? node[:cpus] : 1;
      nodeconfig.vm.provider :virtualbox do |vb|
        vb.customize [
          "modifyvm", :id,
          "--memory", memory.to_s,
          "--cpus", cpus.to_s
        ]
      end
    end
  end
  # In addition, swarm-worker-2 is the Ansible server
  config.vm.define "swarm-worker-2" do |ansible|
    # Provision Ansible playbook
    ansible.vm.provision "file", source: "../Ansible", destination: "$HOME"
    # Install Ansible and configure nodes
    ansible.vm.provision "shell", path: "ansible.sh"
  end
end
```

Luego se inyecta un script Bash en node _swarm-worker-2_ para  instalar Ansible via _pip_:

```sh
$ sudo apt-get install -y python-pip sshpass
$ sudo -H pip install --upgrade pip
$ sudo -H pip install ansible
```

La configuración del clúster se lleva a cabo mediante cuantro playbooks de Ansible. El primero, _cluster.yaml_, instala Docker CE  y docker-compose en los cuatro hosts, independientemente de su función de destino:

```yaml
---
- hosts: all
  become: true
  vars_files:
  - vars.yml
  strategy: free

  tasks:
    - name: Add the docker signing key
      apt_key: url=https://download.docker.com/linux/ubuntu/gpg state=present

    - name: Add the docker apt repo
      apt_repository: repo='deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable' state=present

    - name: Install packages
      apt:
        name: "{{ PACKAGES }}"
        state: present
        update_cache: true
        force: yes

    - name: Add user vagrant to the docker group
      user:
        name: vagrant
        groups: docker
        append: yes
```

El segundo playbook, _master.yaml_, Inicializa Docker Swarm y configura los hostnamed _swarm-master-1_ como  (leader) Swarm Manager.

```yaml
---
- hosts: swarm-master-1
  become: true

  tasks:
    - name: Initialize the cluster
      shell: docker swarm init --advertise-addr 192.168.77.10 >> cluster_initialized.txt
      args:
        chdir: $HOME
        creates: cluster_initialized.txt
```

El Tercer playbook, _join.yaml_, configura Docker Swarm por cuatro hosts: dos maestros  y dos trabajadores. Para lograr eso, se generan dos _join-tokens_ (uno para unirse al clúster como administrador y otro para unirse al clúster como trabajador) en el host que inicializó el enjambre, y se pasa automáticamente a los tres hosts restantes.
```yaml
---
- hosts: swarm-master-1
  become: true
  gather_facts: false

  tasks:
    - name: Get master join command
      shell: docker swarm join-token manager
      register: master_join_command_raw

    - name: Set master join command
      set_fact:
        master_join_command: "{{ master_join_command_raw.stdout_lines[2] }}"

    - name: Get worker join command
      shell: docker swarm join-token worker
      register: worker_join_command_raw

    - name: Set worker join command
      set_fact:
        worker_join_command: "{{ worker_join_command_raw.stdout_lines[2] }}"

- hosts: swarm-master-2
  become: true

  tasks:
    - name: Master joins cluster
      shell: "{{ hostvars['swarm-master-1'].master_join_command }} >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt

- hosts: workers
  become: true

  tasks:
    - name: Workers join cluster
      shell: "{{ hostvars['swarm-master-1'].worker_join_command }} >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt
```

EL cuarto playbook ejecuta con docker-compose con el que se despliega un servicio de docker en el cluster swarm


```yaml
- name: Deploy on Swarm Cluster
  hosts: swarm-master-1
  
  tasks: 
    - name: create swarm
      shell: docker stack deploy -c docker-compose.yml wordpress
      register: stack_deploy

    - name: Check list of services
      command: docker service ls
      register: service_list

    - name: Check list of stack
      command: docker stack ps wordpress
      register: stack_ps


```

## Step 1: Pull el Vagrant Box

Usamos Bento's Ubuntu 16.04 Vagrant Box - https://app.vagrantup.com/bento/boxes/ubuntu-16.04 - como la plantilla de imagen para todos los hosts. Ingrese un shell y escriba:

```sh
$ vagrant box add bento/ubuntu-16.04 --provider virtualbox
```
para extraer la imagen solicitada del catálogo de Vagrant Cloud.

## Step 2: Ejecute el Vagrantfile

Ingrese el siguiente comando

```sh
$ cd Vagrant
$ vagrant up
```

Este creara las VM, en la maquina swarm-master-1 instalara Ansible, desde esa maquina aprovisionara con Docker y docker-compose , posteriormente montara un Cluster de Swarm, para finalmente ejecutar un docker-compose levantando wordpress como aplicación


Su comando ejecuta el _Vagrantfile_, que a su vez, como se explicó anteriormente, instala Ansible y ejecuta los playbooks. Todo esto llevará unos minutos. Eventualmente, nuestro Docker Swarm Cluster estará configurado y listo para ser utilizado.

## Step 3: Verify the Swarm Cluster

Inicie sesión en el primer nodo utilizando el acceso SSH incorporado de Vagrant como usuario _vagrant_ user:

```sh
$ vagrant ssh swarm-master-1
```

En este nodo, que actúa como Swarm Manager, todos los nodos se muestran ahora en el inventario:

```sh
$ vagrant@swarm-master-1:~$ docker node ls
```

![][1]

El swarm gestiona contenedores individuales en los nodos para nosotros. Ahora trabajamos en un nivel superior, con un nuevo concepto llamado ** _ Servicios _ **. Un servicio es una abstracción que dice: "Quiero ejecutar este tipo de contenedor, pero no voy a iniciar y administrar las instancias individuales, quiero que el enjambre lo haga por mí".



Se puede acceder a la página predeterminada a traves de   http://192.168.77.10/. Tenga en cuenta que el último comando nos dice en qué nodo se ha implementado el contenedor, es decir, _docker-worker-1_ esta vez, pero eso cambiará en cada ejecución. La instancia en ejecución se puede enumerar en el nodo adecuado, al que accedemos utilizando otra consola a través de SSH:


```sh
$ vagrant ssh swarm-worker-1

vagrant@swarm-worker-1:~$ docker ps
```


[1]: ./Images/swarm-001.png
[2]: ./Images/swarm-002.png
[3]: ./Images/swarm-003.png
[4]: ./Images/swarm-004.png
[5]: ./Images/swarm-005.png
[6]: ./Images/swarm-006.png
[7]: ./Images/swarm-007.png
[8]: ./Images/swarm-008.png
[9]: ./Images/swarm-009.png