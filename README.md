
Simple test that a multipass can be used to test cloud-init strings and be able to run on RedHat,CentOS,AlmaLinux,RockyLinux and Debian varients.  

For testing the following was used.  
Installed ubuntu 20.04 on a x86_64 with 32gigs memory.
```
ssh-keygen 
sudo snap install multipass
```
Download cloud images:
 RedHat 8	- https://access.redhat.com/
 AlmaLinux 8	- https://github.com/AlmaLinux/cloud-images/releases
 CentOS 8	- https://cloud.centos.org/altarch/8/x86_64/images/
 Rocky Linux 8  - https://download.rockylinux.org/pub/rocky/8.5/images/
 Ubuntu 2004    - https://cloud-images.ubuntu.com/focal/current/

Cloud-init yaml file adding your ssh public key is added to make the ansible connection.  The SSL checker is some extra code i use to verify internal ca-cert is loaded. 
```
#cloud-config
users:
  - default
  - name: ubuntu
    gecos: Ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    shell: /bin/bash
    ssh_import_id: None
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-rsa AAAAB3N <SSH key from .ssh/id_rsa.pub>
package_update: true
package_upgrade: true
write_files:
  # Validate SSL. Used when we use an MITMproxy.  
  - path: /tmp/check_ca-cert.sh
    permissions: "0755"
    content: |
            #!/bin/bash
            . /etc/os-release
            if [[ "$ID_LIKE" == *"debian"* ]]; then
              awk -v cmd='openssl x509 -noout -subject' '
                /BEGIN/{close(cmd)};{print | cmd}' < /etc/ssl/certs/ca-certificates.crt
            elif [[ "$ID_LIKE" == *"rhel"* ]]; then
              awk -v cmd='openssl x509 -noout -subject' '
                /BEGIN/{close(cmd)};{print | cmd}' < /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
            else
              echo "Linux type not detected."
            fi
runcmd:
  - "if [ `bash ./view_ca-certificates.sh | grep -c Sandia` -ge 1 ]; then touch /root/cert-installed.txt; fi"
``` 
sudo multipass launch -n redalma file:///home/user/images/AlmaLinux8.img
sudo multipass launch -n redcent file:///home/user/images/CentOSLinux8.img
sudo multipass launch -n redrock file:///home/user/images/RockyLinux8.img
sudo multipass launch -n redubun file:///home/user/images/Ubuntu2004.img


Example Output:
user@red:~/projects/demo1$ multipass list
Name                    State             IPv4             Image
redalma                 Running           10.108.56.181    Not Available
redcent                 Running           10.108.56.102    Not Available
redrock                 Running           10.108.56.111    Not Available
redubun                 Running           10.108.56.243    Not Available
reddebs                 Running           10.108.56.104    Not Available
redrpms                 Running           10.108.56.26     Not Available

ansible setup
Create inventory
```
redalma ansible_host=10.108.56.181 ansible_user=ubuntu
redcent ansible_host=10.108.56.102 ansible_user=ubuntu
redrock ansible_host=10.108.56.111 ansible_user=ubuntu
redubun ansible_host=10.108.56.243 ansible_user=ubuntu
```
ssh to each ipaddress or fun the ssh-scankey tool to accept the key handshakes.

playbook.yml
```
---
- hosts: all
  become: true
  tasks:
    - name: Install Packages if Debian variety
      apt: name={{ item }} update_cache=yes state=latest
      loop: [ 'nginx', 'vim' ]
      tags: [ 'setup' ]
      when: ansible_distribution_file_variety == 'Debian'
    - name: Install Packages if RedHat variety
      yum: name={{ item }} update_cache=yes state=latest
      loop: [ 'nginx', 'vim' ]
      tags: [ 'setup' ]
      when: ansible_distribution_file_variety == 'RedHat'
    - name: Copy index page for Debian
      copy:
        src: index.html
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
      tags: [ 'update', 'sync' ]
      when: ansible_distribution_file_variety == 'Debian'
    - name: Copy index page for RedHat
      copy:
        src: index.html
        dest: /usr/share/nginx/html/index.html
        owner: nginx
        group: nginx
        mode: '0644'
      tags: [ 'update', 'sync' ]
      when: ansible_distribution_file_variety == 'RedHat'
``` 

```
echo "This was made by ANSIBLE." > index.html
```

Commands for using ansible in this simple way.
# View what ansible sees in the inventory file.
ansible-inventory -i inventory --list

# Use to verify each vm is reachable and can connect. By default multipass creates the ubuntu user.  So that will be the user we use. 
ansible all -u ubuntu -i inventory -m ping
# Run the playbook and it will install vim nginx on all four machines.
ansible-playbook -u ubuntu -i inventory playbook.yml 
# List out the tags that are setup in the playbook.yml file.
ansible-playbook -i inventory playbook.yml --list-tags
# Use this to see what facts can be used as variables.
ansible localhost -m ansible.builtin.setup


To do's:
  Add start,stop,enable function/tags to the playbook. 
  Add TLS/SSL certs generation and ca-cert to configuration. 
  Setup Proxy usage. 
