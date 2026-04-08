# 🚀 ANSIBLE COMPLETE SETUP GUIDE (SINGLE BOX)

```text
1. INSTALLATION (Ubuntu/Debian)
--------------------------------------------------
$ sudo apt update
$sudo apt install software-properties-common -y$ sudo add-apt-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible -y

2. DIRECTORY STRUCTURE
--------------------------------------------------
$mkdir ansible-workspace && cd ansible-workspace$ touch ansible.cfg hosts.ini playbook.yml

3. CONFIGURATION (ansible.cfg)
--------------------------------------------------
# Is file ke hone se aapko "-i hosts.ini" likhne ki zaroorat nahi padegi.
[defaults]
inventory      = ./hosts.ini
remote_user    = ubuntu
host_key_checking = False
become         = True

[privilege_escalation]
become_method  = sudo
become_user    = root
become_ask_pass = False

4. INVENTORY (hosts.ini)
--------------------------------------------------
[web]
server1 ansible_host=192.168.1.10
server2 ansible_host=192.168.1.11

[db]
db_server ansible_host=192.168.1.20

[all:vars]
ansible_python_interpreter=/usr/bin/python3

5. SSH SETUP (Passwordless)
--------------------------------------------------
$ssh-keygen -t rsa  # Press enter for all prompts$ ssh-copy-id ubuntu@192.168.1.10
$ssh-copy-id ubuntu@192.168.1.11$ ssh-copy-id ubuntu@192.168.1.20

6. VERIFICATION (Testing Connection)
--------------------------------------------------
# Ab sirf "ansible all" likhein, config file apne aap inventory utha legi.
$ ansible all -m ping

7. SAMPLE PLAYBOOK (playbook.yml)
--------------------------------------------------
---
- name: Hello World Setup
  hosts: all
  tasks:
    - name: Ensure Apache is installed
      apt:
        name: apache2
        state: present
      when: ansible_os_family == "Debian"

8. RUNNING THE PLAYBOOK
--------------------------------------------------
$ ansible-playbook playbook.yml
