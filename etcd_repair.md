## How to restore 3.11 OC etcd-cluster
#### 1. We should define any of etcd nodes to latest version (we can check timestamp on database files)
#### 2. Make this etcd node as one-member cluster:
```
cp -a /var/lib/etcd /var/lib/etcd.back

mkdir /etc/origin/node/pods-stopped

mv /etc/origin/node/pods/etcd.yaml /etc/origin/node/pods-stopped/

systemctl restart origin-node

chown -R etcd.etcd /etc/etcd
chown -R etcd.etcd /var/lib/etcd

sed -i '/ExecStart/s/"$/ --force-new-cluster"/' /usr/lib/systemd/system/etcd.service;

systemctl daemon-reload

systemctl restart etcd

systemctl show -p SubState -p ActiveState etcd

etcdctl --ca-file=/etc/origin/master/master.etcd-ca.crt \
--cert-file=/etc/origin/master/master.etcd-client.crt \
--key-file=/etc/origin/master/master.etcd-client.key \
-C https://$(hostname -i):2379 member list

sed -i '/ExecStart/s/ --force-new-cluster//g' /usr/lib/systemd/system/etcd.service;

systemctl daemon-reload

systemctl stop etcd

chown -R root.root /etc/etcd

chown -R root.root /var/lib/etcd

mv /etc/origin/node/pods-stopped/etcd.yaml /etc/origin/node/pods/

systemctl restart origin-node

```

#### 3. Wait for etcd containers are running and check etcd cluster state:
```
etcdctl --ca-file=/etc/origin/master/master.etcd-ca.crt \
> --cert-file=/etc/origin/master/master.etcd-client.crt \
> --key-file=/etc/origin/master/master.etcd-client.key \
> -C https://$(hostname -i):2379 member list

{ID}: name=master-1 peerURLs=https://{master-1-ip}:2380 clientURLs=https://{master-1-ip}:2379 isLeader=true
```
#### 4. Add neighbor members one by one to cluster:
On alive etcd node:
```
etcdctl2 member add master-2 https://{master-2-ip}:2380
```
Added member named master-2 with ID {ID} to cluster
```
ETCD_NAME="master-2"
ETCD_INITIAL_CLUSTER="master-2=https://192.168.0.9:2380,master-1=https://192.168.0.18:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```
on master-2 node:
```
systemctl stop origin-node
docker ps | grep etcd
docker stop {id-of-container-with-etcd}
rm -rf /var/lib/etcd/member/*
vi /etc/etcd/etcd.conf - set parameters from step on first etcd node
systemctl restart origin-node
```
and wait for two docker containers etcd will be running

#### 5. Repeat procedure from 4 point to other needed nodes