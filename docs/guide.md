
# Part 2    
## Section 1 - install ceph cluster
### install ceph cluster serverc, serverd, servere

Login node serverc, install require software to bootstrap ceph cluster
    
run preflight  

    ssh-keygen
    for i in clienta clientb serverc serverd servere serverf serverg; do 
        ssh-copy-id@$i
    done

run preflight    

    yum install -y cephadm-ansible
    cd /usr/share/cephadm-ansible
    ansible-playbook -i preflight_host_list.txt cephadm-preflight.yml --extra-vars "ceph_origin="
    
bootstrap with a yaml 

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

### install ceph cluster serverf

run preflight    

    yum install -y cephadm-ansible
    cd /usr/share/cephadm-ansible
    ansible-playbook -i preflight_host_list.txt cephadm-preflight.yml --extra-vars "ceph_origin="

    podman login repo-2.lab.example.com --username quayadmin --password password

bootstrap with a yaml    

    cephadm --image repo-2.lab.example.com/quayadmin/lab/rhceph/rhceph-5-rhel8 bootstrap --mon-ip=serverf \
    --initial-dashboard-password=redhat \
    --dashboard-password-noupdate \
    --allow-fqdn-hostname \
    --registry-url=repo-2.lab.example.com \
    --registry-username=quayadmin \
    --registry-password=password  \
    --yes-i-know

## Section 2 - expand ceph cluster

### add OSD daemon by CLI

On node serverf, add OSD deamon to cluster by CLI 

    cephadm shell -- ceph orch daemon add osd serverc:/dev/sdb
    cephadm shell -- ceph orch daemon add osd serverc:/dev/sdc
    cephadm shell -- ceph orch daemon add osd serverc:/dev/sdd

On node serverf, add OSD deamon to cluster by service specification file /tmp/osd-spec.yaml

    service_type: osd
    service_id: default_drive_group
    placement:
    hosts:
    - serverf
    data_devices:
    paths:
    - /dev/vde
    - /dev/vdf

    ceph orch apply -i /tmp/osd-spec.yaml
    ceph status

# Part 3 - configure cluster    
## Section 1 - configuration settings

On the clienta perform

    ceph config dump

### Config settings debug_ms for OSD daemon

    ceph config show osd.1
    ceph config show osd.1 debug_ms
    ceph config get osd.1 debug_ms
    
    ceph config set osd.1 debug_ms 10
    ceph config show osd.1 debug_ms
    
    ceph orch daemon restart osd.1
    ceph config show osd.1 debug_ms
    ceph config get osd.1 debug_ms

### GUI 
## Section 2 - mon
## Section 3 - network

On node clienta

    ceph config get osd public_network
    ceph config get mon public_network

    ceph config set mon public_network 172.25.250.0/24
    ceph config get mon public_network

# Part 4 - storage components

## Section 1 - bluestore with lvm
On clienta

    ceph device ls
    
    ceph orch device ls | awk /server/ | grep Yes
    ceph orch device ls --wide --refresh
### Using CLi to add OSD daemon

### Using spec to add OSD daemon

The effect of ceph orch apply is persistent which means that the Orchestrator automatically finds the device, adds it to the cluster, and creates new OSDs. This occurs under the following conditions:
New disks or drives are added to the system.
Existing disks or drives are zapped.
An OSD is removed and the devices are zapped.

Add by service and test remove an OSD

    ceph orch apply osd --all-available-devices

Disable by service and test remove an OSD

    ceph orch apply osd --all-available-devices --unmanaged=true

## Section 2 - pool
## Section 3 - authentication   

# Part 5 - storage map

## Section 1 - crush map
On clienta

Created class ssd with id 1 to crush map 

    ceph osd crush class create hdd 
    ceph osd crush class create ssd 
    ceph osd crush class ls

Set new device class to "ssd" on serverc, serverd, servere

    id=7
    ceph osd crush rm-device-class osd.$id
    ceph osd crush set-device-class ssd osd.$id


### Create crush rule with pool belong ssd 

Create a crush rule by command line

    ceph osd crush rule create-replicated onssd default host ssd
    ceph osd crush rule ls

    ceph osd crush tree
    ID CLASS WEIGHT TYPE NAME
    -1 0.08817 root default
    -3 0.02939 host serverc
    0 hdd 0.00980 osd.0
    2 hdd 0.00980 osd.2
    1 ssd 0.00980 osd.1
    -5 0.02939 host serverd
    3 hdd 0.00980 osd.3
    7 hdd 0.00980 osd.7
    5 ssd 0.00980 osd.5
    -7 0.02939 host servere
    4 hdd 0.00980 osd.4
    8 hdd 0.00980 osd.8
    6 ssd 0.00980 osd.6

Create a pool 

    ceph osd pool create myfast 32 32 onssd
    ceph osd lspools
    6 myfast

    ceph pg dump pgs_brief
    PG_STAT STATE UP UP_PRIMARY ACTING ACTING_PRIMARY
    6.1b active+clean [6,5,1] 6 [6,5,1] 6

### Create crush rule with pool with 1 ssd and 2 osd

Create a tree structure

    ceph osd crush add-bucket default-cl260 root
    ceph osd crush add-bucket rack1 rack
    ceph osd crush add-bucket rack2 rack
    ceph osd crush add-bucket rack3 rack
    ceph osd crush add-bucket hostc host
    ceph osd crush add-bucket hostd host
    ceph osd crush add-bucket hoste host

    ceph osd crush move rack1 root=default-cl260
    ceph osd crush move rack2 root=default-cl260
    ceph osd crush move rack3 root=default-cl260
    ceph osd crush move hostc rack=rack1
    ceph osd crush move hostd rack=rack2
    ceph osd crush move hoste rack=rack3
    ceph osd crush move hostd rack=rack2

    [ceph: root@clienta /]# ceph osd crush tree
    ID CLASS WEIGHT TYPE NAME
    ID CLASS WEIGHT TYPE NAME
    -13 0 root default-cl260
    -14 0 rack rack1
    -15 0 host hostc
    -16 0 rack rack2
    -17 0 host hostd
    -18 0 rack rack3
    -19 0 host hoste
    -1 0.08817 root default
    -3 0.02939 host serverc
    0 hdd 0.00980 osd.0
    2 hdd 0.00980 osd.2
    1 ssd 0.00980 osd.1
    -5 0.02939 host serverd
    3 hdd 0.00980 osd.3
    7 hdd 0.00980 osd.7
    5 ssd 0.00980 osd.5
    -7 0.02939 host servere
    4 hdd 0.00980 osd.4
    8 hdd 0.00980 osd.8
    6 ssd 0.00980 osd.6

    ceph osd crush set osd.1 1.0 root=default-cl260 rack=rack1 host=hostc
    ceph osd crush set osd.5 1.0 root=default-cl260 rack=rack1 host=hostc
    ceph osd crush set osd.6 1.0 root=default-cl260 rack=rack1 host=hostc
    ceph osd crush set osd.3 1.0 root=default-cl260 rack=rack2 host=hostd
    ceph osd crush set osd.0 1.0 root=default-cl260 rack=rack2 host=hostd
    ceph osd crush set osd.4 1.0 root=default-cl260 rack=rack2 host=hostd
    ceph osd crush set osd.2 1.0 root=default-cl260 rack=rack3 host=hoste
    ceph osd crush set osd.7 1.0 root=default-cl260 rack=rack3 host=hoste
    ceph osd crush set osd.8 1.0 root=default-cl260 rack=rack3 host=hoste
    
    ceph osd crush tree
    [ceph: root@clienta /]# ceph osd crush tree
    ID CLASS WEIGHT TYPE NAME
    -13 9.00000 root default-cl260
        -14 3.00000 rack rack1
        -15 3.00000 host hostc
                1 ssd 1.00000 osd.1
                5 ssd 1.00000 osd.5
                6 ssd 1.00000 osd.6
        -16 3.00000 rack rack2
        -17 3.00000 host hostd
                0 hdd 1.00000 osd.0
                3 hdd 1.00000 osd.3
                4 hdd 1.00000 osd.4
        -18 3.00000 rack rack3
        -19 3.00000 host hoste
                2 hdd 1.00000 osd.2
                7 hdd 1.00000 osd.7
                8 hdd 1.00000 osd.8
    -1 0 root default
    -3 0 host serverc
    -5 0 host serverd
    -7 0 host servere


Create a crush rule by using file

    ceph osd getcrushmap -o ~/cm-org.bin
    crushtool -d ~/cm-org.bin -o ~/cm-org.txt
    cp ~/cm-org.txt ~/cm-new.txt
    cat ~/cm-new.txt
    ...output omitted...
    rule onssd {
    id 3
    type replicated
    min_size 1
    max_size 10
    step take default class ssd
    step chooseleaf firstn 0 type host
    step emit
    }

    rule ssd-first {
    id 5
    type replicated
    min_size 1
    max_size 10
    step take rack1
    step chooseleaf firstn 1 type host
    step emit
    step take default-cl260 class hdd
    step chooseleaf firstn -1 type rack
    step emit
    }

    crushtool -c ~/cm-new.txt -o ~/cm-new.bin
    crushtool -i ~/cm-new.bin --test --show-mappings --rule=5 --num-rep 3
    ceph osd setcrushmap -i ~/cm-new.bin
    ceph osd crush rule ls
    ceph osd pool create testcrush 32 32 ssd-first
    ceph osd lspools
    ...output omitted...
    6 myfast
    7 testcrush
    ceph pg dump pgs_brief | grep ^6
    dumped pgs_brief
    7.b active+clean [1,8,3] 1 [1,8,3] 1
    7.8 active+clean [5,3,7] 5 [5,3,7] 5
    7.9 active+clean [5,0,7] 5 [5,0,7] 5
    7.e active+clean [1,2,4] 1 [1,2,4] 1

Remap 1 HDD OSD (id=3) of pg 7.8 to other HDD OSD (id=0)
7.8 active+clean [5,3,7] 5 [5,3,7] ->  7.8 (7.8) -> up [5,0,7] acting [5,0,7]
    
    ceph osd pg-upmap-items 7.8 3 0
    ceph pg map 7.8

### Create a replicated rule (maybe conflict with previous test)

Edit the crush structure

    ceph osd crush add-bucket dc1 datacenter
    ceph osd crush add-bucket dc2 datacenter
    ceph osd crush move dc1 root=review-cl260
    ceph osd crush move dc2 root=review-cl260
    ceph osd crush move rack2 datacenter=dc1
    ceph osd crush move rack3 datacenter=dc2

Create a replicated pool and verify object replicas in different Datacenter

    ceph osd crush rule create-replicated replicated1 default-cl260 datacenter
    ceph osd crush rule dump | grep -B2 -A 20 replicated1
    ceph osd pool create reviewpool 64 64 replicated replicated1
    ceph osd pool ls detail | grep reviewpool
    pool 8 'reviewpool' replicated size 3 min_size 2 crush_rule 1 object_hash rjenkins
    ceph osd crush tunables optimal
    ceph pg dump pgs_brief | grep ^8

## Section 2 - manage osd map

    ceph osd dump
    ceph osd set-full-ratio 0.9
    ceph osd set-nearfull-ratio 0.9
    ceph osd getmap -o map.bin
    osdmaptool --print map.bin
 
    osdmaptool --export-crush crush.bin map.bin
    crushtool -d crush.bin -o crush.txt 
    crushtool -c crush.txt -o crushnew.bin
    cp map.bin mapnew.bin
    osdmaptool --import-crush crushnew.bin mapnew.bin
    osdmaptool --test-map-pgs-dump mapnew.bin

# Part 6 - block storage - basic

## Section 1 - rbd
## Section 2 - snapshot
## Section 3 - import/export 

# Part 7 - block storage - advanced

## Section 1 - rbd mirrors


Create prepare mirror 
On node clienta 
    
    ceph osd pool create rbd 32 32
    ceph osd pool application enable rbd rbd
    rbd pool init -p rbd
    ceph orch apply rbd-mirror --placement=serverc

On node serverf

    ceph osd pool create rbd 32 32
    ceph osd pool application enable rbd rbd
    rbd pool init -p rbd
    

Create pool and mirror pool

On node clienta 

    rbd create image1 --size 1024 --pool rbd --image-feature=exclusive-lock,journaling
    rbd -p rbd ls
    rbd --image image1 info

    rbd mirror pool enable rbd pool
    rbd --image image1 info
    
    mkdir /root/mirror
    cephadm shell --mount /root/mirror/
    rbd mirror pool peer bootstrap create --site-name primary rbd > /mnt/bootstrap_token_prod
    rsync -avP /root/mirror/bootstrap_token_prod serverf:/root/bootstrap_token_prod

On node serverf

    cephadm shell --mount /root/bootstrap_token_prod
    ceph orch apply rbd-mirror --placement=serverf
    ceph orch ls
    rbd mirror pool peer bootstrap import --site-name secondary --direction rx-only rbd /mnt/bootstrap_token_prod
    rbd -p rbd ls

Verify mirror status 

On node clienta, serverf
    
    rbd mirror image status rbd/myimage
    rbd mirror pool info rbd
    rbd mirror pool status

Test 1: switch the image between cluster

On node clienta
    rbd mirror image demote rbd/myimage
    rbd mirror image status rbd/myimage

On node serverf
    rbd mirror image promote rbd/myimage
    rbd mirror image status rbd/myimage

Fallback

On node serverf
    rbd mirror image demote rbd/myimage
    rbd mirror image status rbd/myimage

On node clienta
    rbd mirror image promote rbd/myimage
    rbd mirror image status rbd/myimage

Test 2: remove the image on primary cluster

On node clienta

    rbd rm image1 -p rbd
    rbd -p rbd ls

On node serverf

    rbd -p rbd ls


## Section 2 - iscsi

On node clienta 

    ceph config set osd osd_heartbeat_interval 5
    ceph config set osd osd_heartbeat_grace 20
    ceph config set osd osd_client_watch_timeout 15

    vi /etc/ceph/iscsi-gateway.yaml
    service_type: iscsi
    service_id: iscsi
    placement:
        hosts:
        - serverc
        - servere
    spec:
        pool: iscsipool1
        trusted_ip_list: "<ip serverc>,<ip servere>"
        api_port: 5000
        api_secure: false
        api_user: admin
        api_password: redhat
    
    ceph orch apply -i /etc/ceph/iscsi-gateway.yaml
    ceph dashboard iscsi-gateway-list

Create a target on GUI
Configure an iSCSI Initiator

    yum install iscsi-initiator-utils device-mapper-multipath
    mpathconf --enable --with_multipathd y
    vi /etc/multipath.conf
    devices {
        device {
        vendor "LIO-ORG"
        hardware _handler "1 alua"
        path_grouping_policy "failover"
        path_selector "queue-length 0"
        failback 60
        path_checker tur
        prio alua
        prio_args exclusive_pref_bit
        fast_io_fail_tmo 25
        no_path_retry queue
        }
    }
    systemctl reload multipathd
    vi /etc/iscsi/iscsid.conf
    node.session.auth.authmethod = CHAP
    node.session.auth.username = user
    node.session.auth.password = password
    iscsiadm -m discovery -t st -p 10.30.0.210
    iscsiadm -m node -T iqn.2001-07.com.ceph:1634089632951 -l
    lsblk
    multipath -ll

# Part 8 - rgw

## Section 1 - deploy rgw

Create prepare rgw 
    
    ceph orch ls
    ceph orch ls --service-type rgw

Configure rgw service to start 2 RGW instances in serverd and servere, port 8080    
    
    vi /tmp/rgw_service.yaml
    service_type: rgw
    service_id: myrealm.myzone
    service_name: rgw.myrealm.myzone
    placement:
    count: 4
        hosts:
        - serverd.lab.example.com
        - servere.lab.example.com
    spec:
        rgw_frontend_port: 8080
        
## Section 2 - deploy rgw - multisite