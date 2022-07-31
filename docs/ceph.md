## Ceph GLOBAL

### Access

Ceph Administration

    https://10.1.17.101:8443/

Ceph Dashboard

    https://10.1.17.103:3000/

### STATUS

    ceph status
    
    ceph orch ps
    ceph orch device ls
    ceph orch host ls

    ceph osd tree

    ceph osd crush class ls
    ceph osd crush tree

    ceph device ls


### telemetry

Enable telemetry after bootstrap
In case, node exporter is set to redhat registry. We need to reconfigure image base and re-deploy node-expoter and alertmanager

    ceph config set mgr mgr/cephadm/container_image_base repo-2.lab.example.com/quayadmin/lab/rhceph/rhceph-5-rhel8
    ceph config set mgr mgr/cephadm/container_image_alertmanager repo-2.lab.example.com/quayadmin/lab/openshift4/ose-prometheus-alertmanager
    ceph config set mgr mgr/cephadm/container_image_prometheus repo-2.lab.example.com/quayadmin/lab/openshift4/ose-prometheus
    ceph config set mgr mgr/cephadm/container_image_grafana repo-2.lab.example.com/quayadmin/lab/rhceph/rhceph-5-dashboard-rhel8
    ceph config set mgr mgr/cephadm/container_image_node_exporter repo-2.lab.example.com/quayadmin/lab/openshift4/ose-prometheus-node-exporter
    for i in {container_image_prometheus,container_image_grafana,container_image_alertmanager,container_image_node_exporter};do ceph config get mgr mgr/cephadm/$i ;done  
    ceph orch redeploy node-exporter

    ceph log last cephadm
    ceph telemetry on --license sharing-1-0

Disable Monitoring
To disable monitoring and remove the software that supports it, run the following commands:
    
    ceph orch rm grafana
    ceph orch rm prometheus --force
    ceph orch rm node-exporter
    ceph orch rm alertmanager
    ceph mgr module disable prometheus

To redeploy the monitoring run:

    ceph mgr module enable prometheus
    ceph orch apply node-exporter '*'
    ceph orch apply alertmanager 1
    ceph orch apply prometheus 1
    ceph orch apply grafana 1

### MON 
#### Add Mon on c3-server-d, c3-server-e   
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@c3-server-d
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@c3-server-e
    cephadm shell -- ceph orch host add c3-server-d
    cephadm shell -- ceph orch host add c3-server-e
    ceph telemetry on --license sharing-1-0
    
    for i in c3-server-c c3-server-d c3-server-e; do sudo ceph orch host label add $i osd; done
    for i in c3-server-c c3-server-d c3-server-e; do sudo ceph orch host label add $i mon; done

    cephadm shell -- ceph orch apply mon 3
    cephadm shell -- ceph orch apply mon c3-server-c,c3-server-d,c3-server-e
    cephadm shell -- ceph log last cephadm

    [root@c3-server-c cephadm-ansible]# ceph orch host ls
    HOST         ADDR        LABELS          STATUS  
    c3-server-c  10.1.17.73  _admin osd mon          
    c3-server-d  10.1.17.74  osd mon                 
    c3-server-e  10.1.17.75  osd mon                 
    3 hosts in cluster


#### Add MON on c3-server-a

    cephadm shell -- ceph config dump
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@c3-server-a
    cephadm shell -- ceph orch host add c3-server-a
    cephadm shell -- ceph orch apply mon 4
    cephadm shell -- ceph orch apply mon c3-server-c,c3-server-d,c3-server-e,c3-server-a
    cephadm shell -- ceph orch host label add c3-server-a mon
    cephadm shell -- ceph orch host label add c3-server-a _admin

### OSD 

#### Create OSD class

    ceph osd crush class ls
    ceph osd crush class create ssd 
#### Add OSD by cli on admin c3-server-a

Enable the orchestrator service to create OSD daemons automatically from the available cluster devices.

    ceph orch apply osd --all-available-devices

    cephadm shell -- ceph orch device ls
    cephadm shell -- ceph orch daemon add osd c3-server-c:/dev/sdb
    cephadm shell -- ceph orch daemon add osd c3-server-c:/dev/sdc
    cephadm shell -- ceph orch daemon add osd c3-server-d:/dev/sdb
    cephadm shell -- ceph orch daemon add osd c3-server-d:/dev/sdc
    cephadm shell -- ceph orch daemon add osd c3-server-e:/dev/sdb
    cephadm shell -- ceph orch daemon add osd c3-server-e:/dev/sdc

    for i in c3-server-c c3-server-d c3-server-e; do 
        cephadm shell -- ceph orch daemon add osd $i:/dev/sdb
        cephadm shell -- ceph orch daemon add osd $i:/dev/sdc 
    done

#### Remove OSD by cli

To completely remove an OSD from Ceph cluster 
    ceph orch daemon stop osd.8
    ceph orch daemon rm osd.8 --force
    ceph osd rm 8
    ceph osd crush rm osd.8
    
    ceph orch device zap c3-server-c /dev/sde --force
    ceph orch osd rm status


#### Create Bluestore OSD

Add OSD by using CLI

    ceph orch device zap c3-server-c /dev/sdd --force
    ceph orch daemon add osd c3-server-c:/dev/sdd

Add OSD by apply yaml

    ceph orch ls --service-type osd --format yaml
    ceph orch apply -i /tmp/osd_spec.yml


    ceph orch apply -i all-available-devices.yaml
    
    [root@c3-server-a ~]# cat all-available-devices.yaml 
    service_type: osd
    service_id: all-available-devices
    service_name: osd.all-available-devices
    placement:
      host_pattern: '*'
    spec:
      data_devices:
        all: true
      filter_logic: AND
      objectstore: bluestore
    unmanaged: true

Show all service 

    ceph orch ls

### CONFIG 
#### Compact MON Database

    [root@c3-server-a ~]# ceph mon dump
    epoch 8
    fsid 4d785516-f211-11ec-b064-005056bafcf9
    last_changed 2022-06-23T09:36:51.882439+0000
    created 2022-06-22T09:55:37.190264+0000
    min_mon_release 16 (pacific)
    election_strategy: 1
    0: [v2:10.1.17.75:3300/0,v1:10.1.17.75:6789/0] mon.c3-server-e
    1: [v2:10.1.17.71:3300/0,v1:10.1.17.71:6789/0] mon.c3-server-a
    2: [v2:10.1.17.73:3300/0,v1:10.1.17.73:6789/0] mon.c3-server-c
    3: [v2:10.1.17.74:3300/0,v1:10.1.17.74:6789/0] mon.c3-server-d
    dumped monmap epoch 8

    ceph config show mon.c3-server-c
    ceph config show mon.c3-server-c mon_host
    ceph config set mon mon_compact_on_start true
    ceph orch restart mon
    ssh c3-server-c sudo du -sch /var/lib/ceph/4d7...cf9/mon.c3-server-c/store.db/

#### Network    

TO change cluster_network to 192.167.126.0/24

    ceph config get mon public_network 
    ceph config get mon cluster_network 
    ceph config set mon public_network 192.168.126.0/24

    [osd]
        cluster network = 192.168.126.0/24
    
    cephadm shell --mount osd-cluster-network.conf 
    [ceph: root@c3-server-a /]# ceph config assimilate-conf -i /mnt/osd-cluster-network.conf
    [ceph: root@c3-server-a /]# ceph config get osd cluster_network                         
    192.168.126.0/24
    [ceph: root@c3-server-a /]# ceph config get mon public_network
    10.1.0.0/16
    [ceph: root@c3-server-a /]# ceph config set mon public_network 10.1.0.0/16

    podman restart $(podman ps -a -q)

#### Warning

    ceph config get mon.c3-server-c mon_data_avail_warn
    ceph config get mon.c3-server-c mon_max_pg_per_osd
    ceph config set mon mon_data_avail_warn 15
    ceph config set mon mon_max_pg_per_osd 400

## POOL

### Create pool

    ceph osd pool create replpool1 64 64

#### get set config 
    
    ceph osd pool ls detail -f json-pretty
    ceph osd pool set replpool1 size 4
    ceph osd pool get replpool1 size

    ceph osd pool application enable replpool1 rbd
    ceph osd pool rename replpool1 newpool
    ceph osd pool delete newpool

#### Erasure code

    ceph osd erasure-code-profile ls
    ceph osd erasure-code-profile get default
    ceph osd erasure-code-profile set ecprofile-k4-m2 k=4 m=2
    ceph osd pool create ecpool1 64 64 erasure ecprofile-k4-m2

## AUTHENTICATION

### Client Name

Naming convention

    librados:       client.openstack
    rgw:            client.rgw.webgw01
    Administration: client.admin

Keyring    
    /etc/ceph/$cluster.$name.keyring
    /etc/ceph/ceph.client.openstack.keyring
    /etc/ceph/ceph.client.rgw.webgw01.keyring

    root@c3-server-a ceph]# cat ceph.client.admin.keyring 
    [client.admin]
            key = AQAY57JiybKYARAAziO8ezIrsO4VksdNGJVSxQ==
            caps mds = "allow *"
            caps mgr = "allow *"
            caps mon = "allow *"
            caps osd = "allow *"