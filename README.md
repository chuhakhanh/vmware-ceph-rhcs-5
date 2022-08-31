# vmware-ceph-rhcs-5



## Introduction

# Reference

For service management

    https://docs.ceph.com/en/quincy/cephadm/services/

For reconfigure images for Promethues
    
    https://docs.ceph.com/en/octopus/cephadm/monitoring/
    https://bugzilla.redhat.com/show_bug.cgi?id=2013215

For bootstrap with local private registry
    
    https://bugzilla.redhat.com/show_bug.cgi?id=1935044
    https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/5/html-single/installation_guide/index
# Description

Lab environment is provisioning for 10 people. Each people will use a resource pool with the same name. 

deploy-1 is a deploy server, which contains preconfigured container to run ansible playbook
repo-2 is a webserver contains, QuayIO server:
- config/ : configuration files
- 2022_07/<repo ID>: repository 
- :443/ : images to deploy

## Setup cluster
### Deploy virtual machines cluster

From deploy-1 
Create a Virtual machine cluster 

    docker exec -it deploy-1 -u0 /bin/bash;
    git clone https://github.com/chuhakhanh/vmware-ceph-rhcs-5
    cd /root/vmware-ceph-rhcs-5
    git checkout lab-7-2022
    ansible-playbook -i config/inventory/lab setup_vmware_cluster.yml -e "action=create" -e "lab_name=lab1"
    ansible-playbook -i config/inventory/lab setup_vmware_cluster.yml -e "action=destroy" -e "lab_name=lab1"
    ansible-playbook -i config/inventory/lab setup_vmware_cluster.yml -e "action=destroy" -e "lab_name=lab2"
Push public ssh key into this machines due to predefined password (i=lab#)


    for i in lab1 lab2 lab3 lab4 lab5 lab6 lab7 lab8 lab9 lab10;
    do
        chmod u+x ./script/key_copy.sh; ./script/key_copy.sh config/inventory/$i
    done

    for i in lab1
    do
        chmod u+x ./script/key_copy.sh; ./script/key_copy.sh config/inventory/$i
    done
    
    sshpass -p "alo1234" ssh-copy-id -f -i ~/.ssh/id_rsa.pub -o StrictHostKeyChecking=no root@10.1.17.117
    
Then apply prequisite for virual machines

    for i in lab1 lab2 lab3 lab4 lab5 lab6 lab7 lab8 lab9 lab10;
    do
        ansible-playbook -i config/inventory/$i prepare_vmware_cluster.yml -e "lab_name=$i"
    done

    for i in lab1
    do
        ansible-playbook -i config/inventory/$i prepare_vmware_cluster.yml -e "lab_name=$i"
    done


Fully provisioning all lab

    for i in lab1 lab2 lab3 lab4 lab5 lab6 lab7 lab8 lab9 lab10
    do
        ansible-playbook -i config/inventory/lab setup_vmware_cluster.yml -e "action=create" -e "lab_name=$i"
        chmod u+x ./script/key_copy.sh; ./script/key_copy.sh config/inventory/$i
        ansible-playbook -i config/inventory/$i prepare_vmware_cluster.yml -e "lab_name=$i"
    done
    
## Setup ceph
### Configure quayio as default insecure local registry 

    cat /etc/containers/registries.conf 

    [[registry]]
    location = "repo-2.lab.example.com/quayadmin/lab"
    prefix = "registry.redhat.io"
    
    podman system info

[Following steps in docs/gudie.md to work on ceph cluster](docs/guide.md)
