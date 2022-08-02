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
#### Add Mon on serverd, servere   
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@serverd
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@servere
    cephadm shell -- ceph orch host add serverd
    cephadm shell -- ceph orch host add servere
    ceph telemetry on --license sharing-1-0
    
    for i in serverc serverd servere; do sudo ceph orch host label add $i osd; done
    for i in serverc serverd servere; do sudo ceph orch host label add $i mon; done

    cephadm shell -- ceph orch apply mon 3
    cephadm shell -- ceph orch apply mon serverc,serverd,servere
    cephadm shell -- ceph log last cephadm

    [root@serverc cephadm-ansible]# ceph orch host ls
    HOST         ADDR        LABELS          STATUS  
    serverc  10.1.17.73  _admin osd mon          
    serverd  10.1.17.74  osd mon                 
    servere  10.1.17.75  osd mon                 
    3 hosts in cluster


#### Add MON on clienta

    cephadm shell -- ceph config dump
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@clienta
    cephadm shell -- ceph orch host add clienta
    cephadm shell -- ceph orch apply mon 4
    cephadm shell -- ceph orch apply mon serverc,serverd,servere,clienta
    cephadm shell -- ceph orch host label add clienta mon
    cephadm shell -- ceph orch host label add clienta _admin

### OSD 

#### Create OSD class

    ceph osd crush class ls
    ceph osd crush class create ssd 
#### Add OSD by cli on admin clienta

Enable the orchestrator service to create OSD daemons automatically from the available cluster devices.

    ceph orch apply osd --all-available-devices

    cephadm shell -- ceph orch device ls
    cephadm shell -- ceph orch daemon add osd serverc:/dev/sdb
    cephadm shell -- ceph orch daemon add osd serverc:/dev/sdc
    cephadm shell -- ceph orch daemon add osd serverd:/dev/sdb
    cephadm shell -- ceph orch daemon add osd serverd:/dev/sdc
    cephadm shell -- ceph orch daemon add osd servere:/dev/sdb
    cephadm shell -- ceph orch daemon add osd servere:/dev/sdc

    for i in serverc serverd servere; do 
        cephadm shell -- ceph orch daemon add osd $i:/dev/sdb
        cephadm shell -- ceph orch daemon add osd $i:/dev/sdc 
    done

#### Remove OSD by cli

To completely remove an OSD from Ceph cluster 
    ceph orch daemon stop osd.8
    ceph orch daemon rm osd.8 --force
    ceph osd rm 8
    ceph osd crush rm osd.8
    
    ceph orch device zap serverc /dev/sde --force
    ceph orch osd rm status


#### Create Bluestore OSD

Add OSD by using CLI

    ceph orch device zap serverc /dev/sdd --force
    ceph orch daemon add osd serverc:/dev/sdd

Add OSD by apply yaml

    ceph orch ls --service-type osd --format yaml
    ceph orch apply -i /tmp/osd_spec.yml


    ceph orch apply -i all-available-devices.yaml
    
    [root@clienta ~]# cat all-available-devices.yaml 
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

#### Set a OSD class HDD or SSD

Specify the class when deploying OSD, for example, specify the OSD where the deployment disk is located to the specified class:

    ceph-disk prepare --crush-device-class <class> /dev/XXX
    
created class ssd with id 1 to crush map 

    ceph osd crush class create hdd 
    ceph osd crush class create ssd 
    ceph osd crush class ls

Set new device class to "ssd"

    id=7
    ceph osd crush rm-device-class osd.$id
    ceph osd crush set-device-class ssd osd.$id

### CONFIG 
#### Compact MON Database

    [root@clienta ~]# ceph mon dump
    epoch 8
    fsid 4d785516-f211-11ec-b064-005056bafcf9
    last_changed 2022-06-23T09:36:51.882439+0000
    created 2022-06-22T09:55:37.190264+0000
    min_mon_release 16 (pacific)
    election_strategy: 1
    0: [v2:10.1.17.75:3300/0,v1:10.1.17.75:6789/0] mon.servere
    1: [v2:10.1.17.71:3300/0,v1:10.1.17.71:6789/0] mon.clienta
    2: [v2:10.1.17.73:3300/0,v1:10.1.17.73:6789/0] mon.serverc
    3: [v2:10.1.17.74:3300/0,v1:10.1.17.74:6789/0] mon.serverd
    dumped monmap epoch 8

    ceph config show mon.serverc
    ceph config show mon.serverc mon_host
    ceph config set mon mon_compact_on_start true
    ceph orch restart mon
    ssh serverc sudo du -sch /var/lib/ceph/4d7...cf9/mon.serverc/store.db/

#### Network    

TO change cluster_network to 192.167.126.0/24

    ceph config get mon public_network 
    ceph config get mon cluster_network 
    ceph config set mon public_network 192.168.126.0/24

    [osd]
        cluster network = 192.168.126.0/24
    
    cephadm shell --mount osd-cluster-network.conf 
    [ceph: root@clienta /]# ceph config assimilate-conf -i /mnt/osd-cluster-network.conf
    [ceph: root@clienta /]# ceph config get osd cluster_network                         
    192.168.126.0/24
    [ceph: root@clienta /]# ceph config get mon public_network
    10.1.0.0/16
    [ceph: root@clienta /]# ceph config set mon public_network 10.1.0.0/16

    podman restart $(podman ps -a -q)

#### Warning

    ceph config get mon.serverc mon_data_avail_warn
    ceph config get mon.serverc mon_max_pg_per_osd
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

    root@clienta ceph]# cat ceph.client.admin.keyring 
    [client.admin]
            key = AQAY57JiybKYARAAziO8ezIrsO4VksdNGJVSxQ==
            caps mds = "allow *"
            caps mgr = "allow *"
            caps mon = "allow *"
            caps osd = "allow *"