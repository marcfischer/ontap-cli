

Author: Marc Fischer (mf@marcfischer.de)
## Cluster und SVM-Peering
### Interfaces erstellen
jeweils eines pro Node

 Cluster1:
```bash
Cluster1::> net int create -vserver Cluster1 -lif ic_node01 -service-policy default-intercluster -subnet-name subnet -home-node Cluster1-01 -home-port e0e -broadcast-domain Default
  (network interface create)

Cluster1::> net int create -vserver Cluster1 -lif ic_node02 -service-policy default-intercluster -subnet-name subnet -home-node Cluster1-02 -home-port e0e -broadcast-domain Default
  (network interface create)
  
Cluster1::> net int show
  (network interface show)
            Logical    Status     Network            Current       Current Is
Vserver     Interface  Admin/Oper Address/Mask       Node          Port    Home
----------- ---------- ---------- ------------------ ------------- ------- ----
Cluster
            Cluster1-01_clus1 up/up 169.254.1.1/16   Cluster1-01   e0a     true
            Cluster1-01_clus2 up/up 169.254.1.2/16   Cluster1-01   e0b     true
            Cluster1-02_clus1 up/up 169.254.2.1/16   Cluster1-02   e0a     true
            Cluster1-02_clus2 up/up 169.254.2.2/16   Cluster1-02   e0b     true
Cluster1
            Cluster1-01_mgmt1 up/up 192.168.0.111/24 Cluster1-01   e0c     true
            Cluster1-02_mgmt1 up/up 192.168.0.112/24 Cluster1-02   e0c     true
            cluster_mgmt up/up    192.168.0.101/24   Cluster1-01   e0d     true
            ic_node01    up/up    192.168.0.63/24    Cluster1-01   e0e     true
            ic_node02    up/up    192.168.0.64/24    Cluster1-02   e0e     true
c1_svm1
            c1_svm1_nas_lif1 up/up 192.168.0.60/24   Cluster1-01   e0d     true
c1_svm2
            c1_svm2_nas_lif1 up/up 192.168.0.61/24   Cluster1-02   e0d     true
c1_svm3
            c1_svm3_nas_lif1 up/up 192.168.0.62/24   Cluster1-01   e0f     true
```

Cluster2:
```bash
Cluster2::> net int create -vserver Cluster2 -lif ic_node01 -service-policy default-intercluster -address 192.168.0.163 -netmask-length 24 -home-node Cluster2-01 -home-port e0g -broadcast-domain Default
  (network interface create)

Cluster2::> net int create -vserver Cluster2 -lif ic_node02 -service-policy default-intercluster -address 192.168.0.164 -netmask-length 24 -home-node Cluster2-02 -home-port e0g -broadcast-domain Default
  (network interface create)

Cluster2::> net int show
  (network interface show)
            Logical    Status     Network            Current       Current Is
Vserver     Interface  Admin/Oper Address/Mask       Node          Port    Home
----------- ---------- ---------- ------------------ ------------- ------- ----
Cluster
            Cluster2-01_clus1
                         up/up    169.254.3.1/16     Cluster2-01   e0a     true
            Cluster2-01_clus2
                         up/up    169.254.3.2/16     Cluster2-01   e0b     true
            Cluster2-02_clus1
                         up/up    169.254.4.1/16     Cluster2-02   e0a     true
            Cluster2-02_clus2
                         up/up    169.254.4.2/16     Cluster2-02   e0b     true
Cluster2
            Cluster2-01_mgmt1
                         up/up    192.168.0.113/24   Cluster2-01   e0c     true
            Cluster2-02_mgmt1
                         up/up    192.168.0.114/24   Cluster2-02   e0c     true
            cluster_mgmt up/up    192.168.0.102/24   Cluster2-01   e0d     true
            ic_node01    up/up    192.168.0.163/24   Cluster2-01   e0g     true
            ic_node02    up/up    192.168.0.164/24   Cluster2-02   e0g     true
c2_svm1
            c2_svm1_nas_lif1
                         up/up    192.168.0.56/24    Cluster2-01   e0g     true
```

### Cluster Peeren
Cluster 1:
```bash
Cluster1::> cluster peer create -address-family ipv4 -peer-addrs 192.168.0.163,192.168.0.164 -generate-passphrase

Notice:
        Passphrase: eVVqCncVWCyLiFxkENwBS7yy
        Expiration Time: 3/6/2025 10:15:30 +00:00
        Initial Allowed Vserver Peers: -
        Intercluster LIF IP: 192.168.0.63
        Peer Cluster Name: Cluster2

        Warning: make a note of the passphrase - it cannot be displayed again.
```

Cluster 2:
```bash
Cluster2::> cluster peer create -address-family ipv4 -peer-addrs 192.168.0.63,192.168.0.64
Notice: Use a generated passphrase or choose a passphrase of 8 or more characters. To ensure the
        authenticity of the peering relationship, use a phrase or sequence of characters that would
        be hard to guess.

Enter the passphrase:
Confirm the passphrase:

Cluster2::> cluster peer show
Peer Cluster Name         Cluster Serial Number Availability   Authentication
------------------------- --------------------- -------------- --------------
Cluster1                  1-80-000011           Available      ok
```

### SVM Peering
Cluster 1 (erstellen):
```
Cluster1::> vserver peer create -vserver c1_svm1 -peer-cluster Cluster2 -peer-vserver c2_svm1 -applications snapmirror

Info: [Job 124] 'vserver peer create' job queued


Cluster2::> vserver peer show
            Peer        Peer                           Peering        Remote
Vserver     Vserver     State        Peer Cluster      Applications   Vserver
----------- ----------- ------------ ----------------- -------------- ---------
c2_svm1     c1_svm1     pending      Cluster1          snapmirror     c1_svm1
```

Cluster 2 (annehmen):
```bash
Cluster2::> vserver peer accept -vserver c2_svm1 -peer-vserver c1_svm1

Info: [Job 120] 'vserver peer accept' job queued

Cluster2::> vserver peer show
            Peer        Peer                           Peering        Remote
Vserver     Vserver     State        Peer Cluster      Applications   Vserver
----------- ----------- ------------ ----------------- -------------- ---------
c2_svm1     c1_svm1     peered       Cluster1          snapmirror     c1_svm1
```

## Asynchrones Snapmirror
### Volumes erstellen
Security-Style und Language muss zu dem Source Volume passen 
Auf Cluster 2:
```bash
Cluster2::> vol create -volume c1_svm1_vol1_dest -type DP -language C.utF-8 -security-style ntfs -size 1g -aggregate cluster2_01_FC1
[Job 124] Job succeeded: Successful
```

### Snapmirror-Policies anlegen
Auf Cluster 2 (Destination).
#### Snapmirror nur mit dem AFS (keine Snapshots)
```bash
Cluster2::> snapmirror policy create -vserver Cluster2 -policy afs-only -tries 8 -transfer-priority normal -type async-mirror

Cluster2::> snapmirror policy show afs-only
Vserver Policy             Policy Number         Transfer
Name    Name               Type   Of Rules Tries Priority Comment
------- ------------------ ------ -------- ----- -------- ----------
Cluster2 afs-only          async-mirror  1     8  normal  -
  SnapMirror Label: sm_created                         Keep:       1
                                                 Total Keep:       1
```

#### Snapmirror des gesamten Volumes inkl. Snapshots
```bash
Cluster2::> snapmirror policy create -vserver Cluster2 -policy all-snapshots -tries 8 -transfer-priority normal -type async-mirror

Cluster2::> snapmirror policy show all-snapshots
Vserver Policy             Policy Number         Transfer
Name    Name               Type   Of Rules Tries Priority Comment
------- ------------------ ------ -------- ----- -------- ----------
Cluster2 all-snapshots     async-mirror  1     8  normal  -
  SnapMirror Label: sm_created                         Keep:       1
                                                 Total Keep:       1

Cluster2::> snapmirror policy add-rule -vserver Cluster2 -policy all-snapshots -snapmirror-label all_source_snapshots -keep 1

Cluster2::> snapmirror policy show all-snapshots
Vserver Policy             Policy Number         Transfer
Name    Name               Type   Of Rules Tries Priority Comment
------- ------------------ ------ -------- ----- -------- ----------
Cluster1 all-snapshots     async-mirror  2     8  normal  -
  SnapMirror Label: sm_created                         Keep:       1
                    all_source_snapshots                           1
                                                 Total Keep:       2
```

### Snapmirror-Beziehung anlegen
auf der Destination

#### Anlegen der Initialen Beziehung
```bash
Cluster2::> snapmirror create -source-cluster Cluster1 -source-vserver c1_svm1 -source-volume c1_svm1_vol1 -destination-vserver c2_svm1 -destination-volume c1_svm1_vol1_dest -vserver c2_svm1 -policy afs-only -schedule 5min -type XDP
Operation succeeded: snapmirror create for the relationship with destination "c2_svm1:c1_svm1_vol1_dest".

Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm1:c1_svm1_vol1
            XDP  c2_svm1:c1_svm1_vol1_dest
                              Uninitialized
                                      Idle           -         true    -
```

#### Beziehung Initialisieren (Baseline Copy)
```bash
Cluster2::> snapmirror initialize -destination-path c2_svm1:c1_svm1_vol1_dest
Operation is queued: snapmirror initialize of destination "c2_svm1:c1_svm1_vol1_dest".
```

Die Verbindung ist nun "Snapmirrored" und "Idle" und damit erfolgreich gelaufen:
```bash
Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm1:c1_svm1_vol1
            XDP  c2_svm1:c1_svm1_vol1_dest
                              Snapmirrored
                                      Idle           -         true    -

```

#### Erstellund einer Beziehung mit allen Snapshots
Volume erstellen
```bash
Cluster2::> vol create -volume c1_svm1_vol1_allsnap_dest -type DP -language C.utF-8 -security-style ntfs -size 1g -aggregate cluster2_01_FC1                                                                    [Job 126] Job succeeded: Successful
```

Snapmirror-Verbindung erstellen:
```bash
Cluster2::> snapmirror create -source-cluster Cluster1 -source-vserver c1_svm1 -source-volume c1_svm1_vol1 -destination-vserver c2_svm1 -destination-volume c1_svm1_vol1_allsnap_dest  -vserver c2_svm1 -policy all-snapshots -schedule 5min -type XDP
Operation succeeded: snapmirror create for the relationship with destination "c2_svm1:c1_svm1_vol1_allsnap_dest".

Cluster2::> snapmirror show
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm1:c1_svm1_vol1
            XDP  c2_svm1:c1_svm1_vol1_dest
                              Snapmirrored
                                      Idle           -         true    -
                 c2_svm1:c1_svm1_vol1_allsnap_dest
                              Uninitialized
                                      Idle           -         true    -
2 entries were displayed.
```

Snapmirror Initialisieren:
```bash
Cluster2::> snapmirror initialize -destination-path c2_svm1:c1_svm1_vol1_allsnap_dest
Operation is queued: snapmirror initialize of destination "c2_svm1:c1_svm1_vol1_allsnap_dest".
```

## Snapmirror Vault

### Source Konfigurieren

#### Snapshot Policy anlegen
Auf Source:
```bash
Cluster1::> snap policy create -policy ontap-backup -enabled true -schedule1 5min -count1 2 -prefix1 5min -snapmirror-label1 5min -schedule2 10min -prefix2 10min -count2 2 -snapmirror-label2 10min

Cluster1::> snap policy show ontap-backup
Vserver: Cluster1
                         Number of Is
Policy Name              Schedules Enabled Comment
------------------------ --------- ------- ----------------------------------
ontap-backup                     2 true    -
    Schedule       Count  Prefix        SnapMirror Label    Retention Period
    -------------- -----  ------------- ------------------  ------------------
    5min               2  5min          5min                0 seconds
    10min              2  10min         10min               0 seconds
```

#### Volumes anlegen und Snapshot-Policy zuweise
Auf der Source:
```bash
Cluster1::> vol create -vserver c1_svm1 -volume vol_src -state online -size 1g -security-style ntfs -aggregate n1_hdd_1
[Job 128] Job succeeded: Successful

Cluster1::> vol mount -vserver c1_svm1 -volume vol_src -junction-path /vol_src

Warning: The export-policy "default" has no rules in it. The volume will therefore be inaccessible over
         NFS and CIFS protocol.
Do you want to continue? {y|n}: y

Cluster1::> vol modify -vserver c1_svm1 -volume vol_src -snapshot-policy ontap-backup

Warning: You are changing the Snapshot policy on volume "vol_src" to "ontap-backup". Snapshot copies on
         this volume that do not match any of the prefixes of the new Snapshot policy will not be
         deleted. However, when the new Snapshot policy takes effect, depending on the new retention
         count, any existing Snapshot copies that continue to use the same prefixes might be deleted.
         See the 'volume modify' man page for more information.
Do you want to continue? {y|n}: y
Volume modify successful on volume vol_src of Vserver c1_svm1.
```

#### Volume Freigeben
```bash
Cluster1::> cifs share create -vserver c1_svm1 -share-name vol_src -path /vol_src
```

### Destination Konfigurieren

#### Snapmirror Policy anlegen:
```bash
Cluster2::> snapmirror policy create -vserver Cluster2 -policy sm-ontap-backup -tries 8 -transfer-priority normal -type vault

Cluster2::> snapmirror policy add-rule -vserver Cluster2 -policy sm-ontap-backup -snapmirror-label 5min -keep 5

Cluster2::> snapmirror policy add-rule -vserver Cluster2 -policy sm-ontap-backup -snapmirror-label 10min -keep 3

Cluster2::> snapmirror policy show sm-ontap-backup
Vserver Policy             Policy Number         Transfer
Name    Name               Type   Of Rules Tries Priority Comment
------- ------------------ ------ -------- ----- -------- ----------
Cluster2
        sm-ontap-backup    vault         2     8  normal  -
  SnapMirror Label: 5min                               Keep:       5
                    10min                                          3
                                                 Total Keep:       8
```

#### Destination Volume erstellen
```bash
Cluster2::> vol create -volume vol_src_dest -size 5G -aggregate cluster2_02_FC1 -security-style ntfs -type dp
[Job 130] Job succeeded: Successful
```

#### Snapmirror-Beziehung anlegen
```bash
Cluster2::> snapmirror create -source-cluster Cluster1 -source-vserver c1_svm1 -source-volume vol_src -destination-path c2_svm1:vol_src_dest -policy sm-ontap-backup -schedule 5min -vserver c2_svm1 -throttle unlimited -type xdp
Operation succeeded: snapmirror create for the relationship with destination "c2_svm1:vol_src_dest".

Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm1:vol_src
            XDP  c2_svm1:vol_src_dest
                              Uninitialized
                                      Idle           -         true    -
```

#### Snapmirror-Beziehung initialisieren (Baseline Transfer)
```bash
Cluster2::> snapmirror initialize -destination-path c2_svm1:vol_src_dest
Operation is queued: snapmirror initialize of destination "c2_svm1:vol_src_dest".

Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
            XDP  c2_svm1:vol_src_dest
                              Uninitialized
                                      Finalizing     22.01KB   true    03/06 12:45:42
2 entries were displayed.
```

## Snapmirror Failover

### Beziehung Brechen und Destination Volume schreibbar machen
Auf der Destination ausführen. Auf dem Source Volume sollten keine Schreibzugriffe mehr stattfinden

**Quiesce**: Automatischen Sync aussetzen:
```bash
Cluster2::> snapmirror quiesce -destination-path c2_svm1:c1_svm1_vol1_dest
Operation succeeded: snapmirror quiesce for destination "c2_svm1:c1_svm1_vol1_dest".
```

**Break**: Beziehung brechen und Destination Volume schreibbar machen:
```bash
Cluster2::> snapmirror break -destination-path c2_svm1:c1_svm1_vol1_dest
Operation succeeded: snapmirror break for destination "c2_svm1:c1_svm1_vol1_dest".
```

Das Destination Volume ist schreibbar ("RW" anstatt "DP"):
```bash
Cluster2::> vol show c1_svm1_vol1_dest
Vserver   Volume       Aggregate    State      Type       Size  Available Used%
--------- ------------ ------------ ---------- ---- ---------- ---------- -----
c2_svm1   c1_svm1_vol1_dest
                       cluster2_01_FC1
                                    online     RW          1GB    779.6MB   23%
```

### Daten von ehemaliger Source auf Destination Synchronisieren
Damit wird das ehemalige Source Volume zum neuen DP-Volume.

**Resync**:
```bash
Cluster1::> snapmirror resync -destination-path c1_svm1:c1_svm1_vol1 -source-path c2_svm1:c1_svm1_vol1_dest

Warning: All data newer than Snapshot copy
         snapmirror.c79011ea-6f69-11ef-879f-005056990301_2163254227.2025-03-06_134500 on volume
         c1_svm1:c1_svm1_vol1 will be deleted.
Do you want to continue? {y|n}: y
Operation is queued: initiate snapmirror resync to destination "c1_svm1:c1_svm1_vol1".
```

```bash
Cluster1::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c2_svm1:c1_svm1_vol1_dest XDP c1_svm1:c1_svm1_vol1 Snapmirrored Transferring 0B true 03/06 13:59:27
```

```bash
Cluster1::> vol show c1_svm1_vol1
Vserver   Volume       Aggregate    State      Type       Size  Available Used%
--------- ------------ ------------ ---------- ---- ---------- ---------- -----
c1_svm1   c1_svm1_vol1 n2_hdd_1     online     DP          2GB     1.66GB   12%

```

### Ursprung wiederherstellen
Nun können die Daten wieder von der Source an die Destination synchronisiert werden:

#### Auf der Source

**Break**: Beziehung brechen und Destination Volume schreibbar machen:
```bash
Cluster1::> snapmirror break -destination-path c1_svm1:c1_svm1_vol1
Operation succeeded: snapmirror break for destination "c1_svm1:c1_svm1_vol1".
```

#### Auf der Destination
Mit list-destination lassen sich die Snapmirror-Verbindungen anzeigen, die von dem Cluster ausgehen:
```bash
Cluster2::> snapmirror list-destinations
                                                  Progress
Source             Destination         Transfer   Last         Relationship
Path         Type  Path         Status Progress   Updated      Id
----------- ----- ------------ ------- --------- ------------ ---------------
c2_svm1:c1_svm1_vol1_dest
            XDP   c1_svm1:c1_svm1_vol1
                               Idle    -         -            af6440f6-fa84-11ef-b32b-005056990101
```

**Resync**: Verbindung anders herum fortsetzen:
```bash
Cluster2::> snapmirror resync -destination-path c2_svm1:c1_svm1_vol1_dest

Warning: All data newer than Snapshot copy
         snapmirror.9d6f8eea-6f58-11ef-8aed-005056990101_2162178963.2025-03-06_135927 on volume
         c2_svm1:c1_svm1_vol1_dest will be deleted.
Do you want to continue? {y|n}: y
Operation is queued: initiate snapmirror resync to destination "c2_svm1:c1_svm1_vol1_dest".

Cluster2::> snapmirror show
    show         show-history
Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm1:c1_svm1_vol1
            XDP  c2_svm1:c1_svm1_vol1_dest
                              Snapmirrored
                                      Idle           -         true    -
```   

## Snapmirror Synchronous

Volume auf der Source erstellen:
```bash
Cluster1::> vol create -vserver c1_svm1 -volume sync_src -state online -size 1g -security-style ntfs -aggregate n1_hdd_1 -junction-path /sync_src -snapshot-policy none

Warning: The export-policy "default" has no rules in it. The volume will therefore be inaccessible over
         NFS and CIFS protocol.
Do you want to continue? {y|n}: y
[Job 129] Job succeeded: Successful
```

Volume auf der Destination erstellen
```bash
Cluster2::> vol create -volume sync_dest -size 1G -aggregate cluster2_02_FC1 -security-style ntfs -type dp                                                                                                      [Job 132] Job succeeded: Successful
```

Snapmirror-Beziehung anlegen:
```bash
Cluster2::>  snapmirror create -source-cluster Cluster1 -source-vserver c1_svm1 -source-volume sync_src -destination-path c2_svm1:sync_dest  -policy Sync
Operation succeeded: snapmirror create for the relationship with destination "c2_svm1:sync_dest".

Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm1:sync_src
            XDP  c2_svm1:sync_dest
                              Uninitialized
                                      Idle           -         true    -
```

Snapmirror initialisieren (Baseline-Copy)
```bash
Cluster2::> snapmirror initialize

Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm1:sync_src
            XDP  c2_svm1:sync_dest
                              Uninitialized
                                      Transferring   0B        true    03/06 14:54:21
3 entries were displayed.
```

Nach dem Baseline-Transfer ist die Beziehung 'InSync'
```bash
Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm1:sync_src
            XDP  c2_svm1:sync_dest
                              Snapmirrored
                                      InSync         -         true    -
```

## SVM DR
In diesem Beispiel eine Snapmirror-Verbindung mit dem Type `-Identity-preserve true` und keinem `discard-config network`. Damit wird die gesamte Identität der Source SVM inkl. der Netzwerk- und des CIFS-Servers übernommen.

**Parameter `-Identity-preserve true|false`:**
- Wird in der Snapmirror-Beziehung gesezt

**Parameter `discard-config network`:**
- Befindet sich in der Snapmirror-policy

### SVM DR mit Übernahme der Identität

dp-destination vserver auf Destination-Cluster anlegen (-> offline ohne root volume):
```bash
Cluster2::> vserver create -vserver svmdr-dest -subtype dp-destination
[Job 135] Job succeeded:
Vserver creation completed.
```

vserver peer erstellen
```bash
Cluster1::> vserver peer create -vserver c1_svm2 -peer-vserver svmdr-dest -applications snapmirror -peer-cluster Cluster2

Info: [Job 131] 'vserver peer create' job queued
```

vserver peer auf der Destination accepten:
```bash
Cluster2::> vserver peer show
            Peer        Peer                           Peering        Remote
Vserver     Vserver     State        Peer Cluster      Applications   Vserver
----------- ----------- ------------ ----------------- -------------- ---------
c2_svm1     c1_svm1     peered       Cluster1          snapmirror     c1_svm1
svmdr-dest  c1_svm2     pending      Cluster1          snapmirror     c1_svm2
2 entries were displayed.

Cluster2::> vserver peer accept -vserver svmdr-dest -peer-vserver c1_svm2

Info: [Job 136] 'vserver peer accept' job queued
```

Snapmirror-Beziehung anlegen :
```bash
Cluster2::> snapmirror create -source-cluster Cluster1 -source-vserver c1_svm2 -destination-path svmdr-dest: -identity-preserve true -policy MirrorAllSnapshots -schedule 10min

Cluster2::> snapmirror show
    show         show-history
Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm2:    XDP  svmdr-dest:  Uninitialized
                                      Idle           -         true    -
```

Snapmirror-Beziehung initialisieren
```bash
Cluster2::> snapmirror initialize -destination-path svmdr-dest:

Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm2:    XDP  svmdr-dest:  Uninitialized
                                      Transferring   -         true    -
4 entries were displayed.
```

Nach einiger Zeit ist der Baseline-Transfer durchgelaufen und die Verbindung ist "idle" und healthy:
```bash
Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm2:    XDP  svmdr-dest:  Snapmirrored
                                      Idle           -         true    -
4 entries were displayed.
```

Volumes, Lifs und CIFS-Shares wurden angelegt:
```bash
Cluster2::> vol show -vserver svmdr-dest
Vserver   Volume       Aggregate    State      Type       Size  Available Used%
--------- ------------ ------------ ---------- ---- ---------- ---------- -----
svmdr-dest
          c1_svm2_root cluster2_02_FC1
                                    online     RW         20MB    18.71MB    1%
svmdr-dest
          c1_svm2_vol1 cluster2_02_FC1
                                    online     DP          1GB    972.1MB    0%
2 entries were displayed.

Cluster2::> net int show -vserver svmdr-dest
  (network interface show)
            Logical    Status     Network            Current       Current Is
Vserver     Interface  Admin/Oper Address/Mask       Node          Port    Home
----------- ---------- ---------- ------------------ ------------- ------- ----
svmdr-dest
            c1_svm2_nas_lif1
                         up/down  192.168.0.61/24    Cluster2-02   e0f     true

Cluster2::> cifs share show -vserver svmdr-dest
Vserver        Share         Path              Properties Comment  ACL
-------------- ------------- ----------------- ---------- -------- -----------
svmdr-dest     c$            /                 oplocks    -        BUILTIN\Administrators / Full Control
                                               browsable
                                               changenotify
                                               show-previous-versions
svmdr-dest     c1_svm2_share /vol1             oplocks    -        Everyone / Full Control
                                               browsable
                                               changenotify
                                               show-previous-versions
svmdr-dest     ipc$          /                 browsable  -        -
3 entries were displayed.
```

Die SVM bleibt offline, da identity-preserve=true:
```bash
Cluster2::> vserver show
                               Admin      Operational Root
Vserver     Type    Subtype    State      State       Volume     Aggregate
----------- ------- ---------- ---------- ----------- ---------- ----------
Cluster2    admin   -          -          -           -          -
Cluster2-01 node    -          -          -           -          -
Cluster2-02 node    -          -          -           -          -
c2_svm1     data    default    running    running     c2_svm1_   cluster2_
                                                      root       01_FC1
svmdr-dest  data    dp-destination        stopped     c1_svm2_   cluster2_
                               running                root       02_FC1
5 entries were displayed.
```

### SVM DR ohne Übernahme der Identität

Erstellen der SVM auf der Destination:
```bash
Cluster2::> vserver create -vserver svmdr-noid -subtype dp-destination
[Job 164] Job succeeded:
Vserver creation completed.
```

SVM Peer auf dem Source Cluster erstellen:
```bash
Cluster1::> vserver peer create -vserver c1_svm3 -peer-vserver svmdr-noid  -applications snapmirror -peer-cluster Cluster2

Info: [Job 149] 'vserver peer create' job queued
```

Peer auf dem Destiation-Cluster annehmen:
```bash
Cluster2::> vserver peer accept -vserver svmdr-noid -peer-vserver c1_svm3

Info: [Job 165] 'vserver peer accept' job queued
```

Snapmirror-Relationship auf Destination-Cluster erstellen, mit `-identity-preserve false`:
```bash
Cluster2::> snapmirror create -source-cluster Cluster1 -source-vserver c1_svm3 -destination-path svmdr-noid: -vserver svmdr-noid -policy MirrorAllSnapshots -identity-preserve false -schedule 10min

Warning: Accessing data on destination of an identity discard Vserver DR relationship requires LIFs to be created manually. If the
         CIFS protocol is enabled on the source Vserver, enable the CIFS Vserver on the destination Vserver before running
         "snapmirror initialize" or "snapmirror resync".
```

Da mit `-identity-preserve false` keine Lifs und Cifs-Identität angelegt werden, müssen diese nun erstellt werden.

Erstellen des LIFs für die Destination SVM:
```bash
Cluster2::> net int create -vserver svmdr-noid -lif lif_svmdr_noid01 -service-policy default-data-files -address 192.168.0.210 -netmask-length 24 -home-node Cluster2-01 -home-port e0c -status-admin up -broadcast-domain Default
  (network interface create)
```

SVM starten (damit CIFS konfiguriert werden kann):
```bash
Cluster2::> vserver start -vserver svmdr-noid
[Job 166] Job succeeded: DONE
```

DNS-Server für die SVM konfigurieren (ansonsten kann beim CIFS-Join die Domäne nicht aufgelöst werden):
```bash
Cluster2::> dns create -domains demo.netapp.com -name-servers 192.168.0.253 -timeout 2 -vserver svmdr-noid                           
Warning: Only one DNS server is configured. Configure more than one DNS server to avoid a single-point-of-failure.
```

CIFS-Server aktivieren und ins AD joinen:
```bash
Cluster2::> cifs server create -cifs-server noid-svm -domain demo.netapp.com -ou CN=Computers -default-site "" -status-admin up -comment "" -vserver svmdr-noid

In order to create an Active Directory machine account for the CIFS server, you must supply the name and password of a Windows
account with sufficient privileges to add computers to the "CN=Computers" container within the "DEMO.NETAPP.COM" domain.

Enter the user name: administrator

Enter the password:

Notice: SMB1 protocol version is obsolete and considered insecure. Therefore it is deprecated and disabled on this CIFS server.
Support for SMB1 might be removed in a future release. If required, use the (privilege: advanced) "vserver cifs options modify
-vserver svmdr-noid -smb1-enabled true" to enable it.
```

Snapmirror Initialisieren (Baseline übertragen):
```bash
Cluster2::> snapmirror initialize -destination-path svmdr-noid:
```

Nach einiger Zeit ist die Übertragung der Baseline abgeschlossen. Der vserver kann gestartet werden und CIFS-Shares können für einen Readonly-Zugriff bereits eingerichtet werden.

### SVM DR Destination schreibbar machen

Snapmirror-Beziehung quiescen und amit den Schedule aussetzen:
```bash
Cluster2::> snapmirror quiesce -destination-path
    c2_svm1:            c2_svm1:<volume>    svmdr-dest:
    svmdr-dest:<volume>

Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm1:c1_svm1_vol1
            XDP  c2_svm1:c1_svm1_vol1_allsnap_dest
                              Broken-off
                                      Idle           -         false   -
c1_svm2:    XDP  svmdr-dest:  Snapmirrored
                                      Quiesced       -         true    -
2 entries were displayed.
```

Mit Snapmnirror Break dieReplikation brechen und die Destinations Volumes und SVMs schreibbar machen:
```bash
Cluster2::> snapmirror break -destination-path svmdr-dest:

Notice: Volume quota and efficiency operations will be queued after "SnapMirror break"
        operation is complete. To check the status, run "job show -description "Vserverdr
        Break Callback job for Vserver : svmdr-dest"".

Cluster2::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c1_svm2:    XDP  svmdr-dest:  Broken-off
                                      Idle           -         true    -
```

Die SVM ist vom Subtype nun "default",  bleibt aber "stopped". Die Interfaces bleiben daher auch "down":
```bash
Cluster2::> vserver show -vserver svmdr*
                               Admin      Operational Root
Vserver     Type    Subtype    State      State       Volume     Aggregate
----------- ------- ---------- ---------- ----------- ---------- ----------
Cluster2    admin   -          -          -           -          -
Cluster2-01 node    -          -          -           -          -
Cluster2-02 node    -          -          -           -          -
svmdr-dest  data    default    running    stopped     c1_svm2_   cluster2_
                                                      root       02_FC1
4 entries were displayed.

Cluster2::> net int show -vserver svmdr-dest
  (network interface show)
            Logical    Status     Network            Current       Current Is
Vserver     Interface  Admin/Oper Address/Mask       Node          Port    Home
----------- ---------- ---------- ------------------ ------------- ------- ----
svmdr-dest
            c1_svm2_nas_lif1
                         up/down  192.168.0.61/24    Cluster2-02   e0e     true
```

Auf der ehemaligen Source sollte nun sicher gestellt werden, dass die SVM offline ist, und die Interfaces undVolumes nicht mehr erreichbar sind:
```bash
Cluster1::> vserver stop -vserver c1_svm2
[Job 103] Job succeeded: DONE

Cluster1::> cifs share show -vserver c1_svm2                                                 Cluster1::> vserver show
    show            show-aggregates show-protocols
Cluster1::> vserver show
                               Admin      Operational Root
Vserver     Type    Subtype    State      State       Volume     Aggregate
----------- ------- ---------- ---------- ----------- ---------- ----------
Cluster1    admin   -          -          -           -          -
Cluster1-01 node    -          -          -           -          -
Cluster1-02 node    -          -          -           -          -
c1_svm2     data    default    stopped    stopped     c1_svm2_   n2_hdd_1
                                                      root
4 entries were displayed.
```


Jetzt kann die SVM auf der Destination gestartet werden:
```bash
Cluster2::> vserver start -vserver svmdr-dest
[Job 109] Job succeeded: DONE

Cluster2::> net int show -vserver svmdr-dest
  (network interface show)
            Logical    Status     Network            Current       Current Is
Vserver     Interface  Admin/Oper Address/Mask       Node          Port    Home
----------- ---------- ---------- ------------------ ------------- ------- ----
svmdr-dest
            c1_svm2_nas_lif1
                         up/up    192.168.0.61/24    Cluster2-02   e0e     true
```

### SVM DR Resync (Reverse Resync)
In den folgenden Schritten wird die ehemalige Source nun zum Ziel der Snapmirror Synchronisation.

Mit `snapmirror list-destinations`kann überprüft werden, von welches Destinationen Snapmirror-Beziehungen ausgehen:
```bash
Cluster1::> snapmirror list-destinations
                                                  Progress
Source             Destination         Transfer   Last         Relationship
Path         Type  Path         Status Progress   Updated      Id
----------- ----- ------------ ------- --------- ------------ ---------------
c1_svm2:    XDP   svmdr-dest:  Idle    -         -            73a109ad-529a-11f0-b4ee-005056990301
```

Snapmirror-Beziehung anlegen. Die ehemalige Quelle ist nun das Ziel. Sie erscheint als "Broken-off":
```bash
Cluster1::> snapmirror create -source-path svmdr-dest: -destination-path c1_svm2: -vserver c1_svm2 -throttle unlimited -type XDP -schedule 5min -policy MirrorAllSnapshots -identity-preserve true


Cluster1::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
svmdr-dest: XDP  c1_svm2:     Broken-off Idle        -         true    -
2 entries were displayed.
```

Mit `snapmirror resync` kan nun das Delta von der neuen Quelle synchronisiert werden:
```bash
Cluster1::> snapmirror resync -destination-path c1_svm2: -source-path svmdr-dest:                                                                                                         Warning: All data newer than Snapshot copy
         "vserverdr.0.c2be3bfb-5299-11f0-b4ee-005056990301.2025-06-27_071000" on Vserver
         "c1_svm2" will be deleted.
Do you want to continue? {y|n}: y

Cluster1::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
svmdr-dest: XDP  c1_svm2:     Broken-off Transferring -        true    -
```


Nach einiger Zeit erscheint die Beziehung als `idle` und Healthy als`true`:
```bash
Cluster1::> snapmirror show
                                                                       Progress
Source            Destination Mirror  Relationship   Total             Last
Path        Type  Path        State   Status         Progress  Healthy Updated
----------- ---- ------------ ------- -------------- --------- ------- --------
c2_svm1:c1_svm1_vol1_allsnap_dest XDP c1_svm1:c1_svm1_vol1 Snapmirrored Idle - true -
svmdr-dest: XDP  c1_svm2:     Snapmirrored Idle      -         true    -
2 entries were displayed.
```

Die SVM ist nun `dp-destination` und weiterhin `stopped`
```bash
Cluster1::> vserver show
                               Admin      Operational Root
Vserver     Type    Subtype    State      State       Volume     Aggregate
----------- ------- ---------- ---------- ----------- ---------- ----------
Cluster1    admin   -          -          -           -          -
Cluster1-01 node    -          -          -           -          -
Cluster1-02 node    -          -          -           -          -
c1_svm1     data    default    running    running     c1_svm1_   n1_hdd_1
                                                      root
c1_svm2     data    dp-destination        stopped     c1_svm2_   n2_hdd_1
                               stopped                root
c1_svm3     data    default    running    running     c1_svm3_   n1_hdd_2
                                                      root
6 entries were displayed.




