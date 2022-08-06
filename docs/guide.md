    
## Part 1
### cephadm-ansible on serverc

Login node serverc, install require software to bootstrap ceph cluster
    
    yum install -y cephadm-ansible
    cd /usr/share/cephadm-ansible
    ssh-keygen

### bootstrap ceph cluster on serverc

    
#### bootstrap with a yaml 

Due to Bug 2013215 (https://bugzilla.redhat.com/show_bug.cgi?id=2013215) we have to specify the local private registry into the bootstrap command. 

    podman login repo-2.lab.example.com --username quayadmin --password password
    cd /root/ceph
    cephadm --image repo-2.lab.example.com/quayadmin/lab/rhceph/rhceph-5-rhel8 bootstrap --mon-ip=serverc \
    --apply-spec=initial-config-primary-cluster.yaml \
    --initial-dashboard-password=redhat \
    --dashboard-password-noupdate \
    --allow-fqdn-hostname \
    --registry-url=repo-2.lab.example.com \
    --registry-username=quayadmin \
    --registry-password=password  \
    --yes-i-know

### Ceph Operation
