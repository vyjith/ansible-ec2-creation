# ansible-ec2-creation
```
---
- name: "Creating Aws Infra Using Ansible"
  hosts: localhost
  vars_files:
    - awscredentials.yml
    - projectvar.yml
  tasks:
# -------------------------------------------------- 
# Creation aws key pair 
# -------------------------------------------------- 

    - name: "Creating-sshkey-pair for {{ project }}"
      amazon.aws.ec2_key:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{project}}"
        state: present
        tags:
          Name: "{{ project }}"
          project: "{{ project }}"
      register: keypair
        
    - name: "Copying the {{ project }} ssh key into a file {{ project }}.pem"
      when: keypair.changed
      copy: 
        content: "{{ keypair.key.private_key }}"
        dest: "{{ project }}.pem"
        mode: 0400
# --------------------------------------------------         
# Creation of security group for allowing http and https connections
# -------------------------------------------------- 

    - name: "Creating security group {{ project }}"
      amazon.aws.ec2_group:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "security-{{project}}"
        description: "security-{{ project }}"
        state: present
        tags: 
          Name: "security-{{ project }}"
          project: "{{ project }}"
        
        rules: 
          - proto: tcp
            cidr_ip: 0.0.0.0/0
            ports: "{{ webports }}"
            rule_desc: " Allow all on port 80 and port 443" 
      register: sg_webserver
# -------------------------------------------------- 
# Creation of security group for allowing remote connections
# -------------------------------------------------- 

    - name: "Creating security group {{ project }} for remote"
      amazon.aws.ec2_group:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "remote-{{project}}"
        description: "remote-{{ project }}"
        state: present
        tags: 
          Name: "remote-{{ project }}"
          project: "{{ project }}"
        
        rules: 
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0            
            rule_desc: " Allow all on port 80 and port 443" 
      register: sg_remote

# -------------------------------------------------- 
# Creating ec2 instance
# -------------------------------------------------- 
    - name: "Creatign EC2 Instance"
      amazon.aws.ec2_instance:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "web-{{project}}"
        key_name: "{{ keypair.key.name }}"
        instance_type: "{{ instance_type }}"
        image_id: "{{ image_id }}"
        state: present
        security_groups:
          - "{{ sg_webserver.group_id }}"
          - "{{ sg_remote. group_id }}"
        user_data: "{{ lookup('file', '/home/ec2-user/awsansiblerole/userdata.sh', errors='strict') }}"
        exact_count: 3
        tags:
          Name: "web-{{ project }}"
          project: "{{ project }}"
      register: ec2

# -------------------------------------------------- 
# Waiting for ec2 instance to be online
# -------------------------------------------------- 

    - name: "Amazon - Waiting For Ec2 Instacne To Be Online"
      when: ec2.changed 
      wait_for:
        timeout: 60

# -------------------------------------------------- 
# Fetching ec2 instance module
# -------------------------------------------------- 

    - name: "Fecthing ec2 instance info"
      amazon.aws.ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}" 
        filters:
            "tag:Name=web-{{project}}"
      register: ec2


# -------------------------------------------------- 
# Creating dynamic inventory
# -------------------------------------------------- 

    - name: "Amazon - Creating Dynamic Inventory"
      add_host:
        hostname: '{{ item.public_ip_address  }}'
        ansible_ssh_host: '{{ item.public_ip_address }}'
        ansible_ssh_port: 22
        ansible_ssh_private_key_file: "{{ project }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
        groups:
          - web
      with_items: "{{ ec2.instances }}"

- name: "Deploying website on the machine"
  hosts: localhost
  become: true
  vars:
    packages:
      - httpd
      - php
       
  tasks:
    - name: "Installing apache website"
      yum:
        name: "{{ packages }}"
        state: present
      tags:
        - apache

    - name: "Copying website content"         
      copy:
        src: ./website
        dest: /var/www/html/
        group: apache
        owner: apache

      notify:
         - restart-httpd
      tags:
        - apache 

  handlers:    
    - name: 'restart-httpd'
      service:
        name: httpd
        state: restarted
        enabled: true
    
```
