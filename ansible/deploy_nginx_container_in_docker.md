---
- name: Docker Setup and Nginx Deployment
  hosts: all
  become: yes

  tasks:
    - name: 1. Docker Install Karo
      apt:
        name: docker.io
        state: present
        update_cache: yes

    - name: 2. Ubuntu user ko docker group mein shamil karo
      user:
        name: ubuntu
        groups: docker
        append: yes

    - name: 3. Connection reset karo (Taaki group changes apply ho jayein)
      meta: reset_connection
      # Yeh 'newgrp docker' ka automation wala badal hai

    - name: 4. Docker service start aur enable karo
      service:
        name: docker
        state: started
        enabled: yes

    - name: 5. Nginx Container chalao (3000:80)
      community.docker.docker_container:
        name: my_nginx_container
        image: nginx:latest
        state: started
        published_ports:
          - "3000:80"
