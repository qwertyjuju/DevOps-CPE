# TP3 Discover Ansible
## Introduction

- "setup.yml" configuration:
```yml
all:
 vars:
   ansible_user: admin
   ansible_ssh_private_key_file: /home/administrator/.ssh/key_devops_tp3
 children:
   prod:
     hosts: julien.andreoli.takima.cloud
```
This configuration file is used to specify the host to connect to and the connection information. We can then use it with the ansible command:
```bash
ansible all -i inventories/setup.yml -m ping
```
The "-m" option specifies the ansible module to use. In this case it is the ping module. On succes the command returns "pong". It is easier to create playbooks to automate sevral tasks to launch on the target servers. For axample a playbook using the ping module would be like this:
```bash
- hosts: all
  gather_facts: false
  become: true

  tasks:
   - name: Test connection
     ping:
```
The become directive is to launch the command as super user.

## Advanced playbook
- advanced playbook to install and ensure that docker is running on our server:
```yml
- hosts: all
 gather_facts: true
 become: true


 tasks:
   # Install prerequisites for Docker
   - name: Install required packages
     apt:
       name:
         - apt-transport-https
         - ca-certificates
         - curl
         - gnupg
         - lsb-release
         - python3-venv
       state: latest
       update_cache: yes


   # Add Docker’s official GPG key
   - name: Add Docker GPG key
     apt_key:
       url: https://download.docker.com/linux/debian/gpg
       state: present


   # Set up the Docker stable repository
   - name: Add Docker APT repository
     apt_repository:
       repo: "deb [arch=amd64] https://download.docker.com/linux/debian {{ ansible_facts['distribution_release'] }} stable"
       state: present
       update_cache: yes


   # Install Docker
  - name: Install Docker
      apt:
        name: docker-ce
        state: present


   # Install Python3 and pip3
   - name: Install Python3 and pip3
     apt:
       name:
         - python3
         - python3-pip
       state: present


   # Create a virtual environment for Python packages
   - name: Create a virtual environment for Docker SDK
     command: python3 -m venv /opt/docker_venv
     args:
       creates: /opt/docker_venv  # Only runs if this directory doesn’t exist


   # Install Docker SDK for Python in the virtual environment
   - name: Install Docker SDK for Python in virtual environment
     command: /opt/docker_venv/bin/pip install docker


   # Ensure Docker is running
   - name: Make sure Docker is running
     service:
       name: docker
       state: started
     tags: docker
``` 
This playbook has several tasks defined for installing docker and make sure it is running. Each task uses a specific ansible module, for example apt is used to install packages on the server. Tasks can be separated in different roles. It is better to use roles for bigger projects with several servers and a lot of different tasks.

- Command to initialize a role:
```bash
ansible-galaxy init roles/docker
```

Once the role is created we can modify the tasks to be run for this role by modifying the "main.yml" file inside the "tasks" folder of the newly created role. We can then call a role inside our main playbook:
```yml
- hosts: all
  gather_facts: true
  become: true
  roles:
    - docker
    - network
    - database
    - app
    - proxy
```
Here we created roles for different jobs to do on the server:
- inbstall docker and ensure it is running
- create the docker networks
- create the database container
- create the app container
- create the proxy container

The container creation uses the ansible container module. For example the tasks in the proxy role looks like this:
```yml
- name: Run HTTPD
  docker_container:
    name: httpd
    image: qwertyjuju/http-simple-api:latest
    pull: yes
    published_ports: 
      - 80:80
      - 8080:8080
    networks:
      - name: http
```
