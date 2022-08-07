
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

Configure rgw service myrealm.myzone to start 2 RGW instances in serverd and servere, port 8080    
    
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

    ceph orch apply -i rgw_service.yaml    
    ceph orch ps --daemon-type rgw
    NAME HOST STATUS
    REFRESHED AGE PORTS ...
    rgw.myrealm.myzone.serverd.tknapl serverd.lab.example.com running (14s) 0s ago
    14s *:8080 ...
    rgw.myrealm.myzone.serverd.xpabfe serverd.lab.example.com running (6s) 0s ago
    6s *:8081 ...
    rgw.myrealm.myzone.servere.lwusbq servere.lab.example.com running (18s) 0s ago
    17s *:8080 ...
    rgw.myrealm.myzone.servere.uyginy servere.lab.example.com running (10s) 0s ago
    10s *:8081 ...

On node serverd, servere

    podman ps -a --format "{{.ID}} {{.Names}}" | grep rgw
    curl http://serverd:8080
    curl http://serverd:8081

## Section 2 - deploy rgw - multisite

Setup realm, zonegroup, zone 
- Each realm has an associated period, and each period has an associated epoch. A period is used to track the configuration state of the realm, zone groups, and zones at a particular time. 
- Epochs are version numbers to track configuration changes for a particular realm period.
- Each period has a unique ID, contains realm configuration, and knows the previous period ID.
##  realm: cl260
### zonegroup: classroom, zone: us-east-1

Realm: cl260
    
    radosgw-admin realm create --rgw-realm=cl260 --default

Zonegroup: classroom

    radosgw-admin zonegroup create --rgw-zonegroup=classroom --endpoints=http://serverc:80 --master --default

Zone: us-east-1

    radosgw-admin zone create --rgw-zonegroup=classroom --rgw-zone=us-east-1 --endpoints=http://serverc:80 --master --default --access-key=replication --secret=secret

User: repl.user

    radosgw-admin user create --uid="repl.user" --system --display-name="Replication User" --secret=secret --access-key=replication
    
Commit:

    radosgw-admin period update --commit

Service: cl260-1

    ceph orch apply rgw cl260-1 --realm=cl260 --zone=us-east-1 --placement="1 serverc"
    ceph orch ps --daemon-type rgw
    NAME HOST STATUS REFRESHED AGE PORTS ...
    rgw.cl260-1.serverc.sxsntj serverc.lab.example.com running (6m) 6m ago 6m
    *:80 ...

Update zone name:

    ceph config set client.rgw rgw_zone us-east-1

 View:

    radosgw-admin realm pull --url=http://serverc:80 --access-key=replication --secret-key=secret
    radosgw-admin period get-current
    {
    "current_period": "7cdc83cf-69d8-478e-b625-d5250ac4435b"
    }
### zonegroup: classroom, zone: us-east-2

Realm: cl260
    
    radosgw-admin realm default --rgw-realm=cl260

Zonegroup: classroom

    radosgw-admin zonegroup default --rgw-zonegroup=classroom 

Zone: us-east-2

    radosgw-admin zone create --rgw-zonegroup=classroom --rgw-zone=us-east-2 --endpoints=http://serverf:80 --default --default --access-key=replication --secret=secret

Commit:

    radosgw-admin period update --commit --rgw-zone=us-east-2
    {
    "id": "7cdc83cf-69d8-478e-b625-d5250ac4435b",
    }

Update zone name:

    ceph config set client.rgw rgw_zone us-east-2

Service: cl260-2

    ceph orch apply rgw cl260-2 --realm=cl260 --zone=us-east-2 --placement="1 serverf"
    ceph orch ps --daemon-type rgw
    NAME HOST STATUS REFRESHED AGE PORTS ...
    rgw.east.serverf.zgkgem serverf.lab.example.com running (37m) 6m ago 37m
    *:80 ...


 View:

    radosgw-admin realm pull --url=http://serverc:80 --access-key=replication --secret-key=secret
    radosgw-admin period get-current
    {
    "current_period": "7cdc83cf-69d8-478e-b625-d5250ac4435b"
    }

    radosgw-admin sync status

##  realm: prod

### zonegroup: us-west, zone: us-west-1

    radosgw-admin realm create --rgw-realm=prod --default
    radosgw-admin zonegroup create --rgw-zonegroup=us-west --endpoints=http://serverc:8080 --master --default
    radosgw-admin zone create --rgw-zonegroup=us-west --rgw-zone=us-west-1 --endpoints=http://serverc:8080 --master --access-key=admin --secret=secure --default
    radosgw-admin user create --uid="admin.user" --system --display-name="Admin User" --access-key=admin --secret=secure
    radosgw-admin period update --commit
    radosgw-admin period get-current
    ceph orch apply rgw prod-object --realm=prod --zone=us-west-1 --port 8080 --placement="2 serverc.lab.example.com servere.lab.example.com"


# Part 9 - rgw - rest API service

## Section 1 - Amazon S3

On node clienta

    sudo cephadm shell -- radosgw-admin user create --uid="operator" --display-name="S3 Operator" --email="operator@example.com" --access_key="12345" --secret="67890"

On clienta as S3 client

    aws configure --profile=ceph
    aws --profile=ceph --endpoint=http://serverc:80 s3 mb s3://testbucket
    aws --profile=ceph --endpoint=http://serverc:80 s3 ls
    dd if=/dev/zero of=/tmp/10MB.bin bs=1024K count=10
    aws --profile=ceph --endpoint=http://serverc:80 --acl=public-read-write s3 cp /tmp/10MB.bin s3://testbucket/10MB.bin
    
On clienta as ceph admin    

    wget -O /dev/null http://serverc:80/testbucket/10MB.bin
    cephadm shell -- radosgw-admin bucket list
    cephadm shell -- radosgw-admin metadata get bucket:testbucket

## Section 2 - Amazon Swift

On node clienta

    cephadm shell -- radosgw-admin subuser create --uid="operator" --subuser="operator:swift" --access="full" --secret="opswift"

On clienta as Swift client   

    sudo pip3 install --upgrade python-swiftclient
    swift -A http://serverc:80/auth/1.0 -U operator:swift -K opswift stat
    swift -A http://serverc:80/auth/1.0 -U operator:swift -K opswift list
    testbucket
    swift -A http://serverc:80/auth/1.0 -U operator:swift -K opswift post testcontainer
    swift -A http://serverc:80/auth/1.0 -U operator:swift -K opswift list
    testbucket
    testcontainer
    dd if=/dev/zero of=/tmp/swift.dat bs=1024K count=10
    swift -A http://serverc:80/auth/1.0 -U operator:swift -K opswift upload testcontainer /tmp/swift.dat
    tmp/swift.dat
    
    swift -A http://serverc:80/auth/1.0 -U operator:swift -K opswift stat
    swift -A http://serverc:80/auth/1.0 -U operator:swift -K opswift stat testcontainer

## Section 3 - Access in multisite 

On node serverf

    swift -V 1.0 -A http://serverf:80/auth/v1 -U operator:swift -K opswift stat testcontainer    
    swift -V 1.0 -A http://serverf:80/auth/v1 -U operator:swift -K opswift download testcontainer swift.dat

# Part 10 - cephfs (ceph file share)

## Section 1 - cephfs - Deploy

### client to ceph : admin

On clienta as ceph admin

    ceph osd pool create mycephfs_data
    ceph osd pool create mycephfs_metadata
    ceph fs new mycephfs mycephfs_metadata mycephfs_data
    ceph orch apply mds mycephfs --placement="1 serverc"
    ceph mds stat
    ceph status
    ceph df
    ls -l /etc/ceph
    -rw-r--r--. 1 root root 63 Sep 17 21:42 ceph.client.admin.keyring

On clienta as cephfs client - admin user


    yum install ceph-common
    mkdir /mnt/mycephfs
    [root@clienta ~]# ls -l /etc/ceph
    -rw-r--r--. 1 root root 63 Sep 17 21:42 ceph.client.admin.keyring
    mount.ceph serverc.lab.example.com:/ /mnt/mycephfs -o name=admin
    
    df -h
    mkdir /mnt/mycephfs/dir1
    mkdir /mnt/mycephfs/dir2
    ls -al /mnt/mycephfs/
    touch /mnt/mycephfs/dir1/atestfile
    dd if=/dev/zero of=/mnt/mycephfs/dir1/ddtest bs=1024 count=10000
    umount /mnt/mycephfs

On clienta as ceph admin

    cephadm shell -- ceph fs status

### client to ceph : restricted user

On clienta as cephfs admin

    cephadm shell --mount /etc/ceph
    ceph fs authorize mycephfs client.restricteduser / r /dir2 rw
    ceph auth get client.restricteduser -o /mnt/ceph.client.restricteduser.keyring

On clienta as cephfs client - restricted user
    
    mount.ceph serverc.lab.example.com:/ /mnt/mycephfs -o name=restricteduser,fs=mycephfs
    tree /mnt
    /mnt
    └── mycephfs
        ├── dir1
        │ ├── a3rdfile
        │ └── ddtest
        └── dir2
    touch /mnt/mycephfs/dir1/restricteduser_file1
    Permission denied
    touch /mnt/mycephfs/dir2/restricteduser_file2
    umount /mnt/mycephfs



### cephfuse

On clienta as cephfs client - restricted user
    
    yum install ceph-fuse
    mkdir /mnt/mycephfuse
    ceph-fuse -n client.restricteduser --client_fs mycephfs /mnt/mycephfuse
    tree /mnt
    /mnt
    ├── mycephfs
    └── mycephfuse
        ├── dir1
        │ ├── atestfile
        │ └── ddtest
        └── dir2
    umount /mnt/mycephfuse

Mount at boot

    cat /etc/fstab
    serverc.lab.example.com:/ /mnt/mycephfuse fuse.ceph ceph.id=restricteduser,_netdev
    mount -a
    df -h
    umount /mnt/mycephfuse

## Section 2 - cephfs - manafing file

### setfattr

On clienta as cephfs admin - setfattr

    mkdir /mnt/mycephfs/dir1
    touch /mnt/mycephfs/dir1/ddtest
    getfattr -n ceph.dir.layout /mnt/mycephfs/dir1
    /mnt/mycephfs/dir1: ceph.dir.layout: No such attribute
    
    setfattr -n ceph.dir.layout.stripe_count -v 2 /mnt/mycephfs/dir1
    getfattr -n ceph.dir.layout /mnt/mycephfs/dir1
    stripe_count=2
    
    getfattr -n ceph.file.layout /mnt/mycephfs/dir1/ddtest
    stripe_count=1

    touch /mnt/mycephfs/dir1/anewfile
    getfattr -n ceph.file.layout /mnt/mycephfs/dir1/anewfile
    stripe_count=2

    setfattr -n ceph.file.layout.stripe_count -v 3 /mnt/mycephfs/dir1/anewfile
    getfattr -n ceph.file.layout /mnt/mycephfs/dir1/anewfile
    stripe_count=3

    echo "Not empty" > /mnt/mycephfs/dir1/anewfile
    setfattr -n ceph.file.layout.stripe_count -v 4 /mnt/mycephfs/dir1/anewfile
    setfattr: /mnt/mycephfs/dir1/anewfile: Directory not empty

    setfattr -x ceph.dir.layout /mnt/mycephfs/dir1
    touch /mnt/mycephfs/dir1/a3rdfile
    getfattr -n ceph.file.layout /mnt/mycephfs/dir1/a3rdfile
    stripe_count=1

    umount /mnt/mycephfs

### snapshot

On clienta as cephfs admin

    mount.ceph serverc.lab.example.com:/ /mnt/mycephfs -o name=restricteduser
    cd /mnt/mycephfs/.snap
    mkdir mysnapshot
    tree /mnt/mycephfs
    /mnt/mycephfs
    └── dir1
        ├── a3rdfile
        ├── anewfile
        └── ddtest

    tree /mnt/mycephfs/.snap/mysnapshot
    /mnt/mycephfs/.snap/mysnapshot
    └── dir1
        ├── a3rdfile
        ├── anewfile
        └── ddtest

On clienta as cephfs admin

    ceph mgr module enable snap_schedule
    ceph fs snap-schedule add / 1h
    ceph fs snap-schedule status /

Wait for several time

    ls /mnt/mycephfs/.snap

# Part 11 - Cluster

## Section 1 - Monitoring

Ceph module

    ceph mgr module ls | more
    ceph mgr services
    ceph osd stat
    ceph osd find 2

OSD service

    sudo systemctl list-units "ceph*"
    sudo systemctl stop ceph-ff97a876-1fd2-11ec-8258-52540000fa0c@osd.2.service
    ceph osd stat
    journalctl -u ceph-ff97a876-1fd2-11ec-8258-52540000fa 0c@osd.2.service | grep systemd

OSD in out

    ceph osd out 4
    ceph osd stat
    ceph osd tree
    ceph osd df tree
    hdd 0.00980 osd.4 up 0 1.00000
    ceph osd in 4
    ceph osd tree
    ceph osd df tree

PG stat

    ceph pg stat
    ceph osd pool create testpool 32 32
    rados -p testpool put testobject /etc/ceph/ceph.conf
    ceph osd map testpool testobject
    osdmap e332 pool 'testpool' (9) object 'testobject' -> pg 9.98824931 (9.11) -> up
    ([8,2,5], p8) acting ([8,2,5], p8)
    ceph pg 9.11 query
    ceph versions
    ceph tell osd.* version
    ceph balancer status

## Section 2 - Maintenance

### Cluster settings
Set the noscrub and nodeep-scrub flags to prevent the cluster from starting scrubbing
operations temporarily.

On node clienta

    ceph osd set noscrub
    ceph osd set nodeep-scrub
    ceph health detail

    ceph osd tree | grep -i down
    osd.3 down
    ceph osd find osd.3
    "osd": 3, "host": "serverd.lab.example.com",

On node serverd

    cephadm shell
    ceph-volume lvm list

    systemctl list-units --all "ceph*"
    journalctl -ru ceph-2ae6d05a-229a-11ec-925e-52540000fa0c@osd.3.service
    systemctl start ceph-2ae6d05a-229a-11ec-925e-52540000fa0c@osd.3.service

    ceph orch daemon start osd.3

 On node clienta
 
    ceph osd unset noscrub
    ceph osd unset nodeep-scrub
    ceph -w

### Cluster Scale

Add MON daemon on node serverg

    ceph orch ls --service_type=mon
    ceph cephadm get-pub-key > ~/ceph.pub
    ssh-copy-id -f -i ~/ceph.pub root@serverg

Add HOST serverg

    ceph orch host add serverg.lab.example.com

Add MON serverg    

    ceph orch apply mon --placement="clienta.lab.example.com serverc.lab.example.com serverd.lab.example.com servere.lab.example.com serverg.lab.example.com"
    ceph orch ls --service_type=mon
    ceph mon stat
    
Remove MON serverg    
    
    ceph orch apply mon --placement="clienta.lab.example.com serverc.lab.example.com serverd.lab.example.com servere.lab.example.com"
    ceph orch ls --service_type=mon
    ceph mon stat

Remove OSD serverg

    ceph orch ps serverg.lab.example.com
    ceph osd stop 9 10 11
    ceph osd out 9 10 11
    ceph osd crush remove osd.9
    ceph osd crush remove osd.10
    ceph osd crush remove osd.11
    ceph osd rm 9 10 11

Remove HOST serverg
    ceph orch host rm serverg.lab.example.com
    ceph orch host ls

Maintenance HOST servere

    ceph orch host maintenance enter servere.lab.example.com
    ceph orch host ls
    ssh admin@servere sudo reboot # reboot HOST servere
    ceph orch host maintenance exit servere.lab.example.com

Maintenance OSD

Stop and start OSD

    systemctl start ceph-2ae6d05a-229a-11ec-925e-52540000fa0c@osd.6.service
    ceph -w
    ceph osd tree | grep -i down
    ceph osd find osd.6 | grep host
    ceph orch daemon start osd.6
    ceph osd tree | grep osd.6

Out OSD deamon osd.5

    ceph osd out 5
    ceph -w

Verify that all PGs have been migrated off of the OSD 5 daemon. It will take some time for the data migration to finish
    
    ceph osd df
    ceph osd in 5
    ceph osd df
    ceph balancer status
    
Object and PG

    ceph osd map pool1 data1
    (6.1c)` -> up ([8,2,3], p8) acting ([8,2,3], p8)
    ceph pg 6.1c query

# Part 12 - Tuning

## Section 1 - Performance

### Settings pg_autoscale_mode

    ceph osd pool create testpool
    ceph health detail
    ceph osd pool autoscale-status
    
    ceph osd pool set testpool pg_autoscale_mode off
    ceph osd pool set testpool pg_num 8
    ceph osd pool autoscale-status
    ceph health detail

Set the PG autoscale option to warn for the pool testpool. Verify that cluster health status is now WARN, because the recommended number of PGs is higher than the current number of PGs.

    ceph osd pool set testpool pg_autoscale_mode warn
    ceph health detail
    Pool testpool has 8 placement groups, should have 32
    
    ceph osd pool set testpool pg_autoscale_mode on
    ceph osd pool autoscale-status

### OSD Affinity  

 Modify the primary affinity settings on an OSD so that it is more likely to be selected as primary for placement groups. Set the primary affinity for OSD 7 to 0

    ceph osd primary-affinity 7 0
    ceph osd tree
    ceph osd dump | grep affinity

    ceph osd pool create benchpool 100 100
    rbd pool init benchpool
    rados -p benchpool bench 30 write
    ceph osd perf
    osd commit_latency(ms) apply_latency(ms)
    7 94 94
    8 117 117
    6 195 195
    ceph tell osd.6 perf dump > perfdump.txt
    