# Rook On Minikube

1. Start minikube with kvm2 
    ```bash
    > minikube start \
        --force \
        --memory="4096" \
        --cpus="2" \
        -b kubeadm \
        --kubernetes-version="v1.19.2" \
        --driver="kvm2" \
        --feature-gates="BlockVolume=true,CSIBlockVolume=true,VolumeSnapshotDataSource=true,ExpandCSIVolumes=true"
    ```

2. Create and attach devices for ceph
    ```bash
    > sudo -S qemu-img create -f raw /var/lib/libvirt/images/minikube-box-vm-disk-20G-0 20G
    > sudo -S qemu-img create -f raw /var/lib/libvirt/images/minikube-box-vm-disk-20G-1 20G
    > sudo -S qemu-img create -f raw /var/lib/libvirt/images/minikube-box-vm-disk-20G-2 20G

    > virsh -c qemu:///system attach-disk minikube --source /var/lib/libvirt/images/minikube-box-vm-disk-20G-0 --target vdb --cache none
    > virsh -c qemu:///system attach-disk minikube --source /var/lib/libvirt/images/minikube-box-vm-disk-20G-1 --target vdc --cache none
    > virsh -c qemu:///system attach-disk minikube --source /var/lib/libvirt/images/minikube-box-vm-disk-20G-2 --target vdd --cache none
    > virsh -c qemu:///system reboot --domain minikube
    ```

3. Check minikube node device state
    ```bash
    # start minikube again
    > minikube start \
        --force \
        --memory="4096" \
        --cpus="2" \
        -b kubeadm \
        --kubernetes-version="v1.19.2" \
        --driver="kvm2" \
        --feature-gates="BlockVolume=true,CSIBlockVolume=true,VolumeSnapshotDataSource=true,ExpandCSIVolumes=true"

    # vdb, vdc, vdd should be created
    > minikube ssh
                             _             _
                _         _ ( )           ( )
      ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
    /' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
    | ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
    (_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)

    $ ls /dev | grep vd
    vda
    vda1
    vdb
    vdc
    vdd
    ```

4. Create local storageclass & persistentvolume
    ```bash
    > kubectl create -f local-sc.yaml
    > kubectl create -f local-pv.yaml

    # check
    > kubectl get sc
    NAME                 PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    local                kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  2d17h
    standard (default)   k8s.io/minikube-hostpath       Delete          Immediate              false                  2d18h

    > kubectl get pv
    NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
    local-0   20Gi       RWO            Retain           Available           local                   60s
    local-1   20Gi       RWO            Retain           Available           local                   60s
    local-2   20Gi       RWO            Retain           Available           local                   60s
    ```

4. Create cephcluster custom resource
    ```bash
    > kubectl create -f cephcluster.yaml
    cephcluster.ceph.rook.io/rook-ceph created
    
    > kubectl get cephclusters.ceph.rook.io -n rook-ceph
    NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE         MESSAGE                 HEALTH   EXTERNAL
    rook-ceph   /var/lib/rook     1          27s   Progressing   Configuring Ceph Mons

    # wait until all the components are running
    > kubectl get po -n rook-ceph
    NAME                                            READY   STATUS      RESTARTS   AGE
    cluster-cleanup-job-minikube-867b7              0/1     Completed   0          9m9s
    csi-cephfsplugin-provisioner-6d4bd9b669-blx9h   6/6     Running     0          2d18h
    csi-cephfsplugin-q44bq                          3/3     Running     0          2d18h
    csi-rbdplugin-5p6f7                             3/3     Running     0          2d18h
    csi-rbdplugin-provisioner-6bcd78bb89-fvqdc      6/6     Running     0          2d18h
    rook-ceph-mgr-a-654795847c-f4w4q                1/1     Running     0          5m32s
    rook-ceph-mon-a-75d8997474-bc8tf                1/1     Running     0          7m59s
    rook-ceph-operator-757546f8c7-2n4gz             1/1     Running     0          2d19h
    rook-ceph-osd-0-c6fd478bc-sswss                 1/1     Running     0          3m2s
    rook-ceph-osd-1-7cbdf49d5c-5n675                1/1     Running     0          2m50s
    rook-ceph-osd-2-849fdccf5d-lnlpt                1/1     Running     0          2m48s
    rook-ceph-osd-prepare-set-data-0jbqmg-fpr6w     0/1     Completed   0          4m13s
    rook-ceph-osd-prepare-set-data-1xk8k9-7vk48     0/1     Completed   0          4m12s
    rook-ceph-osd-prepare-set-data-2xzhhp-lts45     0/1     Completed   0          4m12s
    ```

5. Install rook-ceph krew plugin
    ```bash
    # https://github.com/rook/kubectl-rook-ceph#install
    > kubectl krew install rook-ceph
    > kubectl rook-ceph ceph -s
      cluster:
        id:     d9aad0fb-51e8-41a9-8b2c-51a61b366674
        health: HEALTH_WARN
                Reduced data availability: 1 pg inactive
                Degraded data redundancy: 1 pg undersized

      services:
        mon: 1 daemons, quorum a (age 7m)
        mgr: a(active, since 3m)
        osd: 3 osds: 3 up (since 2m), 3 in (since 3m)

      data:
        pools:   1 pools, 1 pgs
        objects: 0 objects, 0 B
        usage:   15 MiB used, 60 GiB / 60 GiB avail
        pgs:     100.000% pgs not active
                1 undersized+peered
    ```

6. Create replicapool
    ```bash 
    > kubectl create -f cephblockpool.yaml
    cephblockpool.ceph.rook.io/replicapool created

    > kubectl rook-ceph ceph osd pool ls
    device_health_metrics
    replicapool
    ```

7. Create rbd storageclass
    ```bash
    > kubectl create -f rbd-sc.yaml
    storageclass.storage.k8s.io/rbd created
    ```

8. Test pvc dynamic provisioning, create test pod
    ```bash 
    > kubectl create -f rbd-pvc.yaml
    persistentvolumeclaim/rbd-pvc created
    > kubectl get pvc 
    NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    rbd-pvc   Bound    pvc-70b4532a-9c17-47c3-a45d-dedf58c2df13   1Gi        RWO            rbd            0s

    > kubectl create -f nginx-with-rbd-pvc.yaml
    pod/nginx-with-rbd-pvc created
    # wait until running
    > kubectl get po 
    NAME                 READY   STATUS    RESTARTS   AGE
    nginx-with-rbd-pvc   1/1     Running   0          3m7s
    
    > kubectl exec -it nginx-with-rbd-pvc -- df -h | grep rbd
    /dev/rbd0               975.9M      2.5M    957.4M   0% /data
    ```