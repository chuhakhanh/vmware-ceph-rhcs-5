# vmware-ceph-rhcs-5

## Setup cluster

### Methodology
We need to setup clusters for 10 people. Each people use a resource pool with the same name. 

    ansible-playbook -i config/inventory setup_vmware_cluster.yml -e "action=create" -e "lab=lab-1"

### Deploy virtual machines cluster
From deploy-1 
Create a Virtual machine cluster and push public ssh key into this machines due to predefined password
Then apply prequisite for virual machines
    
    docker exec -it deploy-1 -u0
     /bin/bash;
    git clone https://github.com/chuhakhanh/vmware-ceph-rhcs-5
    cd /root/vmware-ceph-rhcs-5
    ansible-playbook -i config/inventory setup_vmware_cluster.yml -e "action=create"
    
    cp -u config/hosts /etc/hosts
    chmod u+x ./script/key_copy.sh; ./script/key_copy.sh config/inventory
    
    ansible-playbook -i config/inventory prepare_all_node.yml

## Setup ceph version rhcs5
### Configure quayio as default insecure local registry 
Due to Bug 2013215 (https://bugzilla.redhat.com/show_bug.cgi?id=2013215) we have to specify the local private registry into the bootstrap command. 
Below is tested configuration have worked with cephadm bootstrap command. 

    cat /etc/containers/registries.conf 
    unqualified-search-registries = ["registry.fedoraproject.org", "registry.access.redhat.com", "registry.centos.org", "docker.io"]
    [[registry]]
    location = "repo-2.lab.example.com"
    prefix = "registry.redhat.io"
    
    short-name-mode = "permissive"


    podman system info
    
### cephadm-ansible on c8-server-c
Login node c8-server-c, install require software to bootstrap ceph cluster
    
    yum install -y cephadm-ansible
    cd /usr/share/cephadm-ansible
    ssh-keygen
    chmod u+x ./script/key_copy.sh; ./script/key_copy.sh config/inventory

### bootstrap ceph cluster on c8-server-c

### copy initial-config-primary-cluster.yaml to repo-2
From deploy-1

    scp config/initial-config-primary-cluster.yaml repo-2:/data/repos/html/config
    
#### bootstrap with a yaml 

    podman login repo-2.lab.example.com --username quayadmin --password password
    cd /root/ceph
    wget repo-2/config/initial-config-primary-cluster.yaml /root/ceph

    cephadm --image repo-2.lab.example.com/quayadmin/lab/rhceph/rhceph-5-rhel8 bootstrap --mon-ip=10.1.17.103 \
    --apply-spec=initial-config-primary-cluster.yaml \
    --initial-dashboard-password=redhat \
    --dashboard-password-noupdate \
    --allow-fqdn-hostname \
    --registry-url=repo-2.lab.example.com \
    --registry-username=quayadmin \
    --registry-password=password  \
    --yes-i-know
