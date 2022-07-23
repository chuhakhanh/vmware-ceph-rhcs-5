# vmware-ceph-rhcs-5

## Setup cluster
### Deploy virtual machines cluster
From deploy-1 
Create a Virtual machine cluster and push public ssh key into this machines due to predefined password
Then apply prequisite for virual machines
    
    docker exec -it deploy-1 /bin/bash;
    git clone https://github.com/chuhakhanh/vmware-ceph-rhcs-5
    ansible-playbook -i config/inventory setup_vmware_cluster.yml -e "action=create"
    chmod u+x ./script/key_copy.sh; ./script/key_copy.sh config/inventory
    ansible-playbook -i config/inventory prepare_all_node.yml

## Setup ceph version rhcs5
### Configure quayio as default insecure local registry 
Due to Bug 2013215 (https://bugzilla.redhat.com/show_bug.cgi?id=2013215) we have to specify the local private registry into the bootstrap command. 
Below is tested configuration have worked with cephadm bootstrap command. 

    cat /etc/containers/registries.conf | grep . | grep -v "#"
    unqualified-search-registries = ["registry.fedoraproject.org", "registry.access.redhat.com", "registry.centos.org", "docker.io"]
    [[registry]]
    location = "repo-2:8080"
    insecure = true
    # prefix = "registry.redhat.io"
    short-name-mode = "permissive"


    podman system info
    podman login --tls-verify=false repo-2:8080 --username quayadmin --password password
### cephadm-ansible on c8-server-c
Login node c8-server-c, install require software to bootstrap ceph cluster
    
    yum install cephadm-ansible
    cd /usr/share/cephadm-ansible
    ssh-keygen
    chmod u+x ./script/key_copy.sh; ./script/key_copy.sh config/inventory

### bootstrap ceph cluster on c8-server-c

#### bootstrap with a default parameters (using redhat registry)
 
    cephadm bootstrap --mon-ip=10.1.17.69 \
    --initial-dashboard-password=redhat \
    --dashboard-password-noupdate \
    --allow-fqdn-hostname --yes-i-know

#### bootstrap with a local private registry (work)

    cephadm --image repo-2:8080/quayadmin/lab/rhceph/rhceph-5-rhel8 bootstrap --mon-ip=10.1.17.69 \
    --initial-dashboard-password=redhat \
    --dashboard-password-noupdate \
    --allow-fqdn-hostname --yes-i-know \
    --registry-url repo-2:8080 \
    --registry-username quayadmin \
    --registry-password password

#### or bootstrap with a local private registry using auth.json  (work)
    vi /root/auth.json
    {
    "url": "repo-2:8080",
    "username": "quayadmin",
    "password": "password"
    }


   cephadm --image repo-2:8080/quayadmin/lab/rhceph/rhceph-5-rhel8 bootstrap --mon-ip=10.1.17.69 \
    --initial-dashboard-password=redhat \
    --dashboard-password-noupdate \
    --allow-fqdn-hostname --yes-i-know \
    --registry-json /root/auth.json

#### enable telemetry after bootstrap
    ceph telemetry on --license sharing-1-0