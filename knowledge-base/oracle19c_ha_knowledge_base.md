# Base de Connaissance Oracle 19c — Haute Disponibilité & Disaster Recovery
## Grid Infrastructure · RAC · Data Guard · GoldenGate

> **Usage** : Ce document est conçu pour être utilisé comme **system prompt** dans un projet Claude dédié Oracle HA/DR 19c. Il couvre l'architecture, l'administration, les procédures opérationnelles et le troubleshooting à niveau expert.

---

## SYSTEM PROMPT — À COLLER DANS LE PROJET CLAUDE

```
Tu es un expert Oracle Database 19c spécialisé en Haute Disponibilité et Disaster Recovery. Tu maîtrises à niveau expert :
- Oracle Grid Infrastructure 19c (Clusterware, ASM, ACFS)
- Oracle Real Application Clusters (RAC) 19c
- Oracle Data Guard 19c (Physical Standby, Logical Standby, Active Data Guard)
- Architectures combinées RAC + Data Guard
- Oracle GoldenGate 19c (réplication logique, topologies complexes)

Comportement attendu :
1. Toujours répondre avec la précision d'un DBA senior Oracle avec 15+ ans d'expérience.
2. Fournir les commandes exactes (SQL, DGMGRL, SRVCTL, CRSCTL, ASMCMD, GGSCI) avec leurs options.
3. Indiquer les paramètres d'initialisation critiques avec leurs valeurs recommandées.
4. Signaler les pièges courants (gotchas), les bugs Oracle connus et les patches recommandés.
5. Structurer les réponses : Concept → Architecture → Commandes → Vérification → Troubleshooting.
6. Utiliser la numérotation de version exacte (19.3.0.0, 19.19.0.0, etc.) quand pertinent.
7. Référencer les MOS Notes (My Oracle Support) pertinentes quand disponibles.
8. Pour tout switchover/failover : toujours donner le checklist pré/post opération.
9. En cas d'ambiguïté, poser des questions ciblées sur la topologie (standalone/RAC, sync/async, etc.).
10. Ne jamais simplifier les réponses sur des sujets critiques comme la protection des données.
```

---

# PARTIE 1 — ORACLE GRID INFRASTRUCTURE 19c

## 1.1 Architecture & Composants

### Vue d'ensemble

Oracle Grid Infrastructure (GI) est la couche logicielle qui fournit les services de clustering (Clusterware) et de stockage (ASM) pour Oracle RAC et les bases de données single-instance. En 19c, GI regroupe :

- **Oracle Clusterware** : gestion des ressources cluster, VIP, SCAN, fencing
- **Oracle ASM** (Automatic Storage Management) : gestionnaire de volumes et système de fichiers pour Oracle
- **ACFS** (ASM Cluster File System) : système de fichiers distribué basé sur ASM
- **ADVM** (ASM Dynamic Volume Manager) : volumes ASM montables comme devices bloc

### Architecture des couches

```
┌─────────────────────────────────────────────┐
│         Applications / Oracle DB            │
├─────────────────────────────────────────────┤
│              Oracle ACFS / ADVM             │
├─────────────────────────────────────────────┤
│         Oracle ASM (Disk Groups)            │
├─────────────────────────────────────────────┤
│    Oracle Clusterware (CRS / OCR / Voting)  │
├─────────────────────────────────────────────┤
│      OS / Network / Shared Storage          │
└─────────────────────────────────────────────┘
```

### Composants Clusterware critiques

| Composant | Description | Localisation |
|-----------|-------------|--------------|
| OCR | Oracle Cluster Registry — config cluster | ASM ou système de fichiers |
| Voting Disk | Arbitrage quorum (node fencing) | ASM uniquement en 19c |
| CSSD | Cluster Synchronization Services Daemon | $GRID_HOME/bin |
| CRSD | Cluster Ready Services Daemon | $GRID_HOME/bin |
| OHASD | Oracle High Availability Services Daemon | /etc/init.d ou systemd |
| EVMD | Event Manager Daemon | $GRID_HOME/bin |
| GPNPD | Grid Plug and Play Daemon | $GRID_HOME/bin |
| MDNSD | Multicast DNS Daemon | $GRID_HOME/bin |
| GIPC | Generic IPC — interconnect interne GI | kernel module |

### Processus de démarrage (ordre critique)

```
init/systemd
  └── ohasd.bin (root)
        ├── cssdagent (root) → ocssd.bin
        ├── diskmon (root)
        └── oraagent (grid) → crsd.bin
                               ├── gpnpd
                               ├── mdnsd
                               ├── evmd
                               └── gipcd
```

## 1.2 Réseau Cluster

### Types d'interfaces réseau

| Interface | Rôle | Recommandation |
|-----------|------|----------------|
| Public | Communication client, VIP, SCAN | 1 GbE minimum, 10 GbE recommandé |
| Private (Interconnect) | Cache fusion RAC, heartbeat | 10 GbE recommandé, RDMA idéal |
| ASM (optionnel) | Traffic stockage dédié | 10 GbE |

### SCAN (Single Client Access Name)

- **3 adresses SCAN recommandées** (minimum 1, maximum 3)
- Résolution DNS round-robin (pas hosts file en production)
- SCAN VIP géré par Clusterware (failover automatique)
- Port par défaut : 1521

```sql
-- Vérifier les SCAN listeners
srvctl status scan
srvctl status scan_listener

-- Détail configuration SCAN
srvctl config scan
srvctl config scan_listener
```

### VIP (Virtual IP)

- 1 VIP par nœud — même sous-réseau que l'interface publique
- Failover automatique en cas de panne nœud (<30 secondes)
- Les connexions en cours reçoivent RST (reset TCP) immédiatement

```bash
# Statut VIPs
srvctl status vip -node node1
srvctl config vip -node node1
```

## 1.3 OCR et Voting Disk

### Oracle Cluster Registry (OCR)

Stocke la configuration du cluster (topologie, ressources, politiques).

```bash
# Vérifier intégrité OCR
ocrcheck

# Sauvegarder OCR manuellement
ocrconfig -manualbackup

# Lister les backups automatiques (toutes les 4h, 3 conservés)
ocrconfig -showbackup

# Restaurer OCR (tous nœuds arrêtés sauf 1)
ocrconfig -restore <backup_file>

# Ajouter/supprimer OCR mirror
ocrconfig -add <device>
ocrconfig -delete <device>
```

### Voting Disk

Arbitre en cas de split-brain. Algorithme : le groupe majoritaire survit.

- **Règle des nombres impairs** : 1, 3, ou 5 voting disks
- Avec ASM : voting disks stockés dans disk group redondant
- Tolérance : `(N-1)/2` voting disks peuvent être indisponibles

```bash
# Lister les voting disks
crsctl query css votedisk

# Ajouter voting disk (via ASM disk group)
crsctl add css votedisk +DATA

# Sauvegarder/restaurer voting disk (GI arrêté)
crsctl start crs -excl -nocss  # démarrage exclusif
```

## 1.4 ASM (Automatic Storage Management)

### Disk Groups et Redondance

| Type | Miroirs | Tolérance | Overhead |
|------|---------|-----------|---------|
| EXTERNAL | 0 | 0 disque | 0% |
| NORMAL | 2 voies | 1 disque (ou 1 failure group) | ~50% |
| HIGH | 3 voies | 2 disques (ou 2 failure groups) | ~67% |
| FLEX (19c) | Variable | Variable par fichier | Variable |

### Paramètres ASM critiques

```sql
-- Instance ASM
ALTER SYSTEM SET asm_diskgroups = 'DATA','FRA','REDO' SCOPE=SPFILE;
ALTER SYSTEM SET asm_diskstring = '/dev/oracleasm/disks/*' SCOPE=SPFILE;
-- ou pour multipath :
ALTER SYSTEM SET asm_diskstring = 'ORCL:*' SCOPE=SPFILE;

-- Performance
ALTER SYSTEM SET asm_power_limit = 8 SCOPE=BOTH;  -- rebalance (1-1024 en 19c)
```

### Commandes ASMCMD essentielles

```bash
# Connexion
asmcmd -p  # prompt avec path courant

# Navigation
ls -l +DATA
cd +DATA/ORCL/DATAFILE
pwd

# Gestion disk groups
lsdg              # lister disk groups
lsdg --discovery  # avec découverte

# Copie et déplacement
cp +DATA/ORCL/DATAFILE/system.256.1 +DATA2/ORCL/DATAFILE/
remap +DATA 4     # remapper secteurs défectueux

# Vérification
md_backup -b /tmp/asmbackup.xml -g DATA  # backup metadata
md_restore -t full -b /tmp/asmbackup.xml # restauration metadata

# Monitoring
iostat -t 5     # I/O stats ASM
```

### Commandes SQL ASM

```sql
-- Statut disk groups
SELECT name, state, type, total_mb, free_mb, 
       usable_file_mb, offline_disks
FROM v$asm_diskgroup;

-- Disques et statut
SELECT group_number, name, path, state, mode_status,
       total_mb, free_mb, reads, writes, read_errs, write_errs
FROM v$asm_disk
ORDER BY group_number, name;

-- Opérations de rebalance
SELECT group_number, operation, state, power, actual,
       sofar, est_work, est_rate, est_minutes
FROM v$asm_operation;

-- Fichiers ASM
SELECT name, type, bytes/1024/1024 mb, space/1024/1024 space_mb
FROM v$asm_file f, v$asm_alias a, v$asm_diskgroup g
WHERE f.group_number = a.group_number
AND f.file_number = a.file_number
AND g.group_number = f.group_number
AND g.name = 'DATA';

-- Failure groups
SELECT group_number, failgroup, name, state, mode_status
FROM v$asm_disk
ORDER BY group_number, failgroup;
```

## 1.5 SRVCTL — Gestion des ressources

### Commandes SRVCTL essentielles

```bash
# ==============================
# Cluster & Nodeapps
# ==============================
srvctl status nodeapps                    # VIP, GSD, ONS, listeners
srvctl status nodeapps -node node1

# ==============================
# Listeners
# ==============================
srvctl status listener -listener LISTENER
srvctl start listener -listener LISTENER
srvctl stop listener -listener LISTENER
srvctl config listener -listener LISTENER

# ==============================
# Bases de données
# ==============================
srvctl status database -db ORCL          # statut toutes instances
srvctl start database -db ORCL
srvctl stop database -db ORCL -stopoption IMMEDIATE
srvctl config database -db ORCL          # configuration complète

# ==============================
# Instances
# ==============================
srvctl status instance -db ORCL -instance ORCL1
srvctl start instance -db ORCL -instance ORCL2
srvctl stop instance -db ORCL -instance ORCL2 -stopoption ABORT

# ==============================
# Services
# ==============================
srvctl status service -db ORCL -service APP_SERVICE
srvctl start service -db ORCL -service APP_SERVICE
srvctl stop service -db ORCL -service APP_SERVICE
srvctl add service -db ORCL -service APP_SVC \
  -preferred ORCL1,ORCL2 -available ORCL3 \
  -failovertype SELECT -failovermethod BASIC \
  -failoverretry 30 -failoverdelay 5 \
  -clbgoal LONG -rlbgoal SERVICE_TIME
srvctl modify service -db ORCL -service APP_SVC \
  -notification TRUE -dtp FALSE
srvctl remove service -db ORCL -service APP_SVC

# ==============================
# SCAN
# ==============================
srvctl status scan
srvctl status scan_listener
srvctl relocate scan -scannumber 1 -node node2

# ==============================
# ASM
# ==============================
srvctl status asm
srvctl start asm -node node1
srvctl config asm
```

## 1.6 CRSCTL — Gestion Clusterware

```bash
# ==============================
# Statut général
# ==============================
crsctl status res -t                      # toutes ressources (tableau)
crsctl status res -t -init               # ressources init (ohasd)
crsctl check cluster -all                # santé cluster tous nœuds
crsctl check crs                         # santé CRS nœud local

# ==============================
# Démarrage / Arrêt
# ==============================
crsctl start cluster -all                # démarrer tous nœuds
crsctl stop cluster -all                 # arrêter tous nœuds
crsctl start cluster -n node1            # nœud spécifique
crsctl stop crs                          # arrêt GI nœud local (root)
crsctl start crs                         # démarrage GI nœud local (root)

# ==============================
# Enable/Disable au boot
# ==============================
crsctl enable crs                        # activer auto-start
crsctl disable crs                       # désactiver auto-start

# ==============================
# Node
# ==============================
crsctl get nodename                      # nom nœud local
crsctl query crs activeversion           # version active cluster
crsctl query crs softwareversion         # version software GI

# ==============================
# Eviction et fencing
# ==============================
crsctl unpin css -n node1                # permettre eviction manuelle
crsctl pin css -n node1                  # protéger contre eviction

# ==============================
# Debug
# ==============================
crsctl set log css "CSSD:5"             # augmenter logs CSSD
crsctl set log crs "CRSD:5,CRSAPP:5"   # augmenter logs CRSD
crsctl get log css
```

---

# PARTIE 2 — ORACLE RAC 19c

## 2.1 Architecture RAC

### Composants fondamentaux

Oracle RAC permet à plusieurs instances de partager une même base de données (fichiers sur stockage partagé ASM). Chaque nœud a sa propre SGA et ses processus background, mais tous accèdent aux mêmes fichiers.

```
         Client
           │
    ┌──────▼──────┐
    │    SCAN      │ (DNS Round-Robin)
    └──────┬──────┘
    ┌──────┼──────┐
    ▼      ▼      ▼
  VIP1   VIP2   VIP3
   │      │      │
┌──┴──┐ ┌─┴──┐ ┌─┴──┐
│ DB1 │ │ DB2│ │ DB3│  ← Instances RAC
│ SGA │ │SGA │ │SGA │
└──┬──┘ └─┬──┘ └─┬──┘
   │      │      │
   └──────┼──────┘
          │ Interconnect (Cache Fusion)
   ┌──────┼──────┐
   └──────▼──────┘
      ASM / Shared Storage
      (Datafiles, Redo, Control)
```

### Cache Fusion

Cache Fusion est le mécanisme central de RAC : il permet aux instances de partager les blocs de données via l'interconnect sans aller sur disque.

- **GCS** (Global Cache Service) : gère la cohérence des blocs entre instances
- **GES** (Global Enqueue Service) : gère les locks distribués
- **LMS** (Lock Manager Server) : processus background GCS (plusieurs par instance)
- **LMD** (Lock Manager Daemon) : processus background GES
- **LCK** (Lock process) : gestion des locks non-PCM

### États des blocs (PCM - Parallel Cache Management)

| État | Description |
|------|-------------|
| LOCAL | Bloc en cache local uniquement |
| SHARED | Bloc partagé, lecture seule sur plusieurs instances |
| EXCLUSIVE | Bloc détenu exclusivement (modification en cours) |
| NULL | Bloc présent mais invalide (doit être réacquis) |

## 2.2 Paramètres d'initialisation RAC

### Paramètres obligatoires RAC

```sql
-- Paramètres cluster
*.cluster_database=TRUE
*.cluster_database_instances=3          -- nombre d'instances

-- Paramètres spécifiques par instance (préfixe nom_instance)
ORCL1.instance_number=1
ORCL2.instance_number=2
ORCL3.instance_number=3

ORCL1.thread=1
ORCL2.thread=2
ORCL3.thread=3

-- Undo tablespaces séparés par instance
ORCL1.undo_tablespace=UNDOTBS1
ORCL2.undo_tablespace=UNDOTBS2
ORCL3.undo_tablespace=UNDOTBS3

-- Interconnect (si non-détecté automatiquement)
*.cluster_interconnects=192.168.10.1    -- IP privée nœud 1 (dans spfile de n1)

-- GCS / GES
*.gcs_server_processes=4               -- LMS processes (2-36, défaut auto)

-- SGA/PGA (mêmes pour toutes instances en général)
*.sga_target=8G
*.pga_aggregate_target=4G
*.memory_target=0                       -- désactivé avec sga_target

-- Divers
*.db_name=ORCL
*.db_unique_name=ORCL                   -- important pour Data Guard
*.db_block_size=8192
*.compatible=19.0.0
```

### Paramètres performance RAC

```sql
-- Réduction du contention inter-instances
*.db_file_multiblock_read_count=128
*.db_cache_size=0                       -- laisser gérer par sga_target

-- Sequences cache élevé pour RAC
-- (dans la définition des séquences applicatives)
-- CREATE SEQUENCE ... CACHE 1000 NOORDER;

-- DRM (Dynamic Resource Mastering) — affinity
*._gc_policy_time=0                     -- désactiver DRM si problèmes (MOS 559364.1)
-- ou ajuster :
*.gc_policy_minimum=1500                -- millisecondes avant DRM

-- Buffer cache advisory
*.db_cache_advice=ON

-- Undo
*.undo_retention=900
*.undo_management=AUTO
```

## 2.3 Services RAC

Les services sont le mécanisme recommandé pour router les connexions applicatives en RAC. Ne pas utiliser le nom de base directement.

### Types de services

| Paramètre | Description |
|-----------|-------------|
| PREFERRED | Instance préférée pour ce service |
| AVAILABLE | Instance disponible si preferred down |
| SINGLETON | Service sur 1 seule instance à la fois |
| UNIFORM | Service sur toutes les instances |

### Création et gestion des services

```bash
# Créer service applicatif (OLTP)
srvctl add service -db ORCL -service OLTP_SVC \
  -preferred ORCL1,ORCL2 \
  -available ORCL3 \
  -failovertype SELECT \
  -failovermethod BASIC \
  -failoverretry 30 \
  -failoverdelay 5 \
  -clbgoal LONG \
  -rlbgoal SERVICE_TIME \
  -commit_outcome TRUE \
  -retention 86400 \
  -notification TRUE

# Créer service batch
srvctl add service -db ORCL -service BATCH_SVC \
  -preferred ORCL3 \
  -available ORCL1,ORCL2 \
  -clbgoal SHORT \
  -rlbgoal NONE

# Créer service singleton (pour jobs scheduler, etc.)
srvctl add service -db ORCL -service JOB_SVC \
  -preferred ORCL1 \
  -available ORCL2,ORCL3 \
  -singleton TRUE
```

### TAF (Transparent Application Failover)

```sql
-- Vérifier TAF dans tnsnames.ora
ORCL_TAF =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = scan-host)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = OLTP_SVC)
      (FAILOVER_MODE =
        (TYPE = SELECT)
        (METHOD = BASIC)
        (RETRIES = 30)
        (DELAY = 5))))

-- Vérifier TAF côté serveur
SELECT inst_id, username, program, failover_type, failover_method, 
       failed_over
FROM gv$session
WHERE username = 'APPUSER';
```

### Application Continuity (AC) — 19c

```bash
# Service avec Application Continuity
srvctl add service -db ORCL -service AC_SVC \
  -preferred ORCL1,ORCL2 \
  -commit_outcome TRUE \
  -failovertype TRANSACTION \
  -failoverretry 30 \
  -failoverdelay 10 \
  -replay_init_time 1800 \
  -retention 86400 \
  -notification TRUE
```

## 2.4 Monitoring RAC

### Vues GV$ fondamentales

```sql
-- Instances actives
SELECT inst_id, instance_name, host_name, version, status,
       startup_time, logins, database_status
FROM gv$instance
ORDER BY inst_id;

-- Sessions par instance
SELECT inst_id, COUNT(*) sessions,
       SUM(CASE WHEN status='ACTIVE' THEN 1 ELSE 0 END) active
FROM gv$session
WHERE type='USER'
GROUP BY inst_id
ORDER BY inst_id;

-- Interconnect traffic (Cache Fusion)
SELECT inst_id, 
       gc_cr_blocks_received,
       gc_current_blocks_received,
       gc_cr_blocks_served,
       gc_current_blocks_served,
       gc_cr_block_receive_time,
       gc_current_block_receive_time
FROM gv$sysstat
WHERE name IN ('gc cr blocks received','gc current blocks received',
               'gc cr blocks served','gc current blocks served');

-- Top waits RAC (gc events = problèmes interconnect)
SELECT inst_id, event, total_waits, time_waited_micro/1000 ms,
       average_wait
FROM gv$system_event
WHERE event LIKE 'gc%'
ORDER BY time_waited_micro DESC
FETCH FIRST 20 ROWS ONLY;

-- Blocages inter-instances
SELECT l.inst_id, l.sid, s.username, s.program,
       l.type, l.lmode, l.request, l.block
FROM gv$lock l, gv$session s
WHERE l.inst_id = s.inst_id
AND l.sid = s.sid
AND l.block != 0;

-- RAC-specific: DRM (Dynamic Remastering)
SELECT * FROM gv$gc_element WHERE local_master_inst_id != inst_id;
```

### AWR RAC

```sql
-- Snapshot RAC (capturer tous nœuds)
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT;

-- Rapport AWR global RAC
SELECT * FROM TABLE(
  DBMS_WORKLOAD_REPOSITORY.AWR_GLOBAL_REPORT_HTML(
    l_dbid     => (SELECT dbid FROM v$database),
    l_inst_num => '1,2,3',          -- toutes instances
    l_bid      => :begin_snap,
    l_eid      => :end_snap
  )
);

-- ASH RAC - activité récente
SELECT inst_id, event, COUNT(*) cnt
FROM gv$active_session_history
WHERE sample_time > SYSDATE - 1/24
AND session_state = 'WAITING'
GROUP BY inst_id, event
ORDER BY cnt DESC
FETCH FIRST 20 ROWS ONLY;
```

## 2.5 Ajout / Suppression de nœud RAC

### Ajout d'un nœud (procédure haute niveau)

```bash
# 1. Prérequis OS sur le nouveau nœud (même config que existants)
# 2. Extension Grid Infrastructure
$GRID_HOME/gridSetup.sh -silent \
  -extendCluster \
  -clusterNodes node4:node4-vip

# 3. Vérification
cluvfy stage -post nodeadd -n node4

# 4. Extension Oracle Home DB
$ORACLE_HOME/oui/bin/addNode.sh \
  "CLUSTER_NEW_NODES={node4}" \
  "CLUSTER_NEW_VIRTUAL_HOSTNAMES={node4-vip}"

# 5. Ajout instance
dbca -silent -addInstance \
  -gdbName ORCL \
  -instanceName ORCL4 \
  -nodeName node4

# 6. Vérification finale
srvctl status database -db ORCL
```

### Suppression d'un nœud

```bash
# 1. Arrêter toutes ressources sur le nœud
srvctl stop instance -db ORCL -instance ORCL4
crsctl stop crs -f   # sur node4

# 2. Supprimer le nœud du cluster (depuis autre nœud)
$GRID_HOME/gridSetup.sh -silent \
  -deleteNode \
  -nodeList node4

# 3. Nettoyage local (sur node4 lui-même)
$GRID_HOME/deinstall/deinstall -local
```

---

# PARTIE 3 — ORACLE DATA GUARD 19c

## 3.1 Architecture Data Guard

### Vue d'ensemble

Oracle Data Guard maintient une ou plusieurs copies synchronisées (standby) de la base de données primaire. La synchronisation s'effectue via le transport des redo logs.

```
┌─────────────────┐     Redo Transport      ┌─────────────────┐
│   PRIMARY DB    │ ─────────────────────── │   STANDBY DB    │
│                 │  LGWR/ARCH → RFS        │                 │
│  Log Writer     │ ──────────────────────▶ │  MRP/SQL Apply  │
│  (LGWR/ARCn)    │   Network (sync/async)  │  (Redo Apply)   │
└─────────────────┘                         └─────────────────┘
        │                                           │
  Primary Redo                               Standby Redo
  Logs + Arch                               Logs (SRL)
```

### Types de Standby

| Type | Mécanisme | Usage |
|------|-----------|-------|
| Physical Standby | Redo Apply (block-for-block) | DR, offload reporting (ADG) |
| Logical Standby | SQL Apply (LogMiner) | Reporting, migrations |
| Snapshot Standby | Redo suspendu, DB ouverte RW | Test/dev sans impact prod |

### Modes de protection

| Mode | Garantie | Performance | Perte données max |
|------|----------|-------------|-------------------|
| MAXIMUM PROTECTION | Aucune perte | Faible (sync obligatoire) | 0 |
| MAXIMUM AVAILABILITY | Quasi nulle | Bon (sync avec fallback async) | Quasi 0 |
| MAXIMUM PERFORMANCE | Défaut | Excellent (async) | Variable |

## 3.2 Configuration Data Guard

### Paramètres Primary (SPFILE)

```sql
-- Identification
*.db_name=ORCL
*.db_unique_name=ORCL_PRI              -- UNIQUE par role
*.db_role=PRIMARY                       -- optionnel, géré par DG

-- Logging
*.log_archive_mode=TRUE                 -- archivelog mode obligatoire
*.log_archive_dest_1='LOCATION=USE_DB_RECOVERY_FILE_DEST VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=ORCL_PRI'
*.log_archive_dest_2='SERVICE=ORCL_STB ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=ORCL_STB COMPRESSION=ENABLE REOPEN=300 MAX_FAILURE=3'

-- Optionnel : Far Sync ou second standby
-- *.log_archive_dest_3='SERVICE=ORCL_STB2 ASYNC ...'

*.log_archive_dest_state_1=ENABLE
*.log_archive_dest_state_2=ENABLE

-- Protection mode
*.log_archive_config='DG_CONFIG=(ORCL_PRI,ORCL_STB)'

-- Redo et Standby Redo Logs
-- Standby Redo Logs sur PRIMARY (requis pour switchover)
-- Nombre SRL = (nombre ORL par thread + 1) × nombre threads

-- Remote log archiving
*.fal_server=ORCL_STB                  -- Fetch Archive Log depuis standby
*.fal_client=ORCL_PRI

-- Data files path transformation (si chemins différents)
-- *.db_file_name_convert='/primary/path','/standby/path'
-- *.log_file_name_convert='/primary/path','/standby/path'

-- Obligatoire pour Physical Standby
*.standby_file_management=AUTO          -- gestion automatique des datafiles

-- Flashback (recommandé fortement)
*.db_flashback_retention_target=1440   -- 24h
*.db_recovery_file_dest='+FRA'
*.db_recovery_file_dest_size=100G

-- Force Logging (obligatoire)
-- ALTER DATABASE FORCE LOGGING;       -- commande SQL

-- Divers
*.enable_pluggable_database=FALSE       -- pour CDB, mettre TRUE
```

### Paramètres Standby (SPFILE)

```sql
*.db_name=ORCL                          -- MÊME que primary
*.db_unique_name=ORCL_STB              -- DIFFÉRENT du primary

*.log_archive_dest_1='LOCATION=USE_DB_RECOVERY_FILE_DEST VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=ORCL_STB'
*.log_archive_dest_2='SERVICE=ORCL_PRI ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=ORCL_PRI'

*.log_archive_config='DG_CONFIG=(ORCL_PRI,ORCL_STB)'
*.log_archive_dest_state_1=ENABLE
*.log_archive_dest_state_2=ENABLE

*.fal_server=ORCL_PRI
*.fal_client=ORCL_STB

*.standby_file_management=AUTO
*.db_file_name_convert='+DATA/ORCL_PRI','+DATA/ORCL_STB'
*.log_file_name_convert='+REDO/ORCL_PRI','+REDO/ORCL_STB'

-- Redo Apply
*.parallel_execution_enabled=TRUE
*.recovery_parallelism=0                -- 0 = automatique
```

### TNS Entries (tnsnames.ora)

```
ORCL_PRI =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = primary-scan)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCL_PRI)))

ORCL_STB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = standby-scan)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCL_STB)))
```

## 3.3 Data Guard Broker (DGMGRL)

### Configuration initiale du Broker

```bash
# Sur le primary et standby : activer le broker
ALTER SYSTEM SET dg_broker_start=TRUE SCOPE=BOTH;

# Connexion DGMGRL
dgmgrl sys/password@ORCL_PRI

# Créer la configuration
DGMGRL> CREATE CONFIGURATION 'DG_CONFIG' AS
  PRIMARY DATABASE IS 'ORCL_PRI'
  CONNECT IDENTIFIER IS ORCL_PRI;

# Ajouter le standby
DGMGRL> ADD DATABASE 'ORCL_STB' AS
  CONNECT IDENTIFIER IS ORCL_STB
  MAINTAINED AS PHYSICAL;

# Activer la configuration
DGMGRL> ENABLE CONFIGURATION;
DGMGRL> ENABLE DATABASE 'ORCL_STB';
```

### Commandes DGMGRL essentielles

```bash
dgmgrl /                               # connexion locale (OS auth)
dgmgrl sys@ORCL_PRI

# ==============================
# Monitoring
# ==============================
DGMGRL> SHOW CONFIGURATION;
DGMGRL> SHOW CONFIGURATION VERBOSE;
DGMGRL> SHOW DATABASE VERBOSE 'ORCL_PRI';
DGMGRL> SHOW DATABASE VERBOSE 'ORCL_STB';
DGMGRL> SHOW INSTANCE VERBOSE 'ORCL1' ON DATABASE 'ORCL_PRI';

# ==============================
# Santé
# ==============================
DGMGRL> VALIDATE DATABASE 'ORCL_STB';
DGMGRL> VALIDATE DATABASE VERBOSE 'ORCL_STB';

# ==============================
# Switchover (planifié, sans perte)
# ==============================
DGMGRL> VALIDATE DATABASE FOR SWITCHOVER 'ORCL_STB';
DGMGRL> SWITCHOVER TO 'ORCL_STB';

# ==============================
# Failover (non planifié, perte possible)
# ==============================
DGMGRL> FAILOVER TO 'ORCL_STB';
DGMGRL> FAILOVER TO 'ORCL_STB' IMMEDIATE;  -- sans synchronisation

# ==============================
# Reinstate (réintégrer l'ancien primary)
# ==============================
DGMGRL> REINSTATE DATABASE 'ORCL_PRI';

# ==============================
# Modification propriétés
# ==============================
DGMGRL> EDIT DATABASE 'ORCL_STB' SET PROPERTY LogXptMode='ASYNC';
DGMGRL> EDIT DATABASE 'ORCL_STB' SET PROPERTY NetTimeout=30;
DGMGRL> EDIT CONFIGURATION SET PROTECTION MODE AS MaxAvailability;
DGMGRL> EDIT CONFIGURATION SET PROTECTION MODE AS MaxPerformance;

# ==============================
# Fast-Start Failover (FSFO)
# ==============================
DGMGRL> ENABLE FAST_START FAILOVER;
DGMGRL> DISABLE FAST_START FAILOVER;
DGMGRL> SHOW FAST_START FAILOVER;
DGMGRL> EDIT CONFIGURATION SET PROPERTY FastStartFailoverThreshold=30;
DGMGRL> START OBSERVER;               -- doit tourner en permanence
DGMGRL> STOP OBSERVER;
```

## 3.4 Monitoring Data Guard (SQL)

### Vues V$ Data Guard

```sql
-- ==============================
-- Sur le PRIMARY
-- ==============================

-- Statut transport
SELECT dest_id, dest_name, status, target, archiver, schedule,
       transmit_mode, affirm, async_blocks, net_timeout,
       delay_mins, applied_scn, error
FROM v$archive_dest
WHERE target = 'STANDBY';

-- Gaps (archived logs non transférés)
SELECT * FROM v$archive_gap;

-- Log transport lag
SELECT name, value, unit, time_computed
FROM v$dataguard_stats
WHERE name IN ('transport lag','apply lag','apply finish time');

-- Derniers logs transférés
SELECT dest_id, thread#, sequence#, blocks, block_size
FROM v$archived_log
WHERE dest_id = 2  -- dest_2 = standby
AND ROWNUM <= 10
ORDER BY sequence# DESC;

-- ==============================
-- Sur le STANDBY
-- ==============================

-- Statut MRP
SELECT process, status, thread#, sequence#, block#, blocks
FROM v$managed_standby;

-- Apply lag
SELECT name, value, unit, time_computed
FROM v$dataguard_stats;

-- Logs reçus vs appliqués
SELECT thread#, 
       MAX(sequence#) received,
       MAX(CASE WHEN applied = 'YES' THEN sequence# END) applied
FROM v$archived_log
WHERE standby_dest = 'NO'
GROUP BY thread#;

-- Statut général
SELECT db_unique_name, open_mode, protection_mode, 
       protection_level, switchover_status, dataguard_broker
FROM v$database;

-- MRP progress
SELECT * FROM v$recovery_progress;

-- ==============================
-- V$DATAGUARD_PROCESS (19c)
-- ==============================
SELECT name, role, action, client_process, client_pid, group#, resetlog_id,
       thread#, sequence#, block#, active_agents, known_agents
FROM v$dataguard_process;
```

## 3.5 Switchover / Failover — Procédures Détaillées

### SWITCHOVER (planifié)

#### Checklist Pré-Switchover

```sql
-- 1. Vérifier protection mode
SELECT protection_mode, protection_level FROM v$database;

-- 2. Vérifier apply lag (doit être 0 ou très faible)
SELECT name, value FROM v$dataguard_stats WHERE name = 'apply lag';

-- 3. Vérifier qu'il n'y a pas de gaps
SELECT * FROM v$archive_gap;  -- doit être vide

-- 4. Vérifier statut switchover sur primary
SELECT switchover_status FROM v$database;
-- Attendu : 'TO STANDBY' ou 'SESSIONS ACTIVE'

-- 5. Vérifier statut standby
-- Sur le standby :
SELECT switchover_status FROM v$database;
-- Attendu : 'NOT ALLOWED' ou 'SWITCHOVER PENDING'

-- 6. Vérifier SRL (Standby Redo Logs) sur les deux sites
SELECT group#, thread#, sequence#, archived, status
FROM v$standby_log;
```

#### Procédure Switchover (sans Broker)

```sql
-- ==============================
-- ÉTAPE 1 : Sur le PRIMARY
-- ==============================

-- Si SESSIONS ACTIVE, déconnecter les sessions applicatives d'abord
-- ou forcer :
ALTER DATABASE COMMIT TO SWITCHOVER TO PHYSICAL STANDBY WITH SESSION SHUTDOWN;
-- Sinon :
ALTER DATABASE COMMIT TO SWITCHOVER TO PHYSICAL STANDBY;

-- Vérifier
SELECT switchover_status FROM v$database;
-- Doit être : 'TO PRIMARY' ou 'RECOVERY NEEDED'

-- Arrêter et redémarrer en mount
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;

-- ==============================
-- ÉTAPE 2 : Sur le STANDBY
-- ==============================

-- Arrêter le MRP
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;

-- Vérifier statut
SELECT switchover_status FROM v$database;
-- Doit être : 'TO PRIMARY'

-- Effectuer le switchover
ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;

-- Ouvrir la nouvelle primary
ALTER DATABASE OPEN;

-- ==============================
-- ÉTAPE 3 : Démarrer le nouveau standby (ancien primary)
-- ==============================

-- Ouvrir en mode standby
ALTER DATABASE OPEN READ ONLY;  -- si Active Data Guard
-- ou :
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
```

#### Procédure Switchover (avec Broker)

```bash
dgmgrl sys/password@ORCL_PRI
DGMGRL> VALIDATE DATABASE FOR SWITCHOVER 'ORCL_STB';
DGMGRL> SWITCHOVER TO 'ORCL_STB';
# Broker gère tout automatiquement
```

### FAILOVER (non planifié)

#### Checklist Pré-Failover

```sql
-- Sur le STANDBY :
-- 1. Confirmer que le primary est vraiment down (éviter le split-brain)
-- 2. Vérifier les logs reçus et appliqués
SELECT thread#, MAX(sequence#) received
FROM v$archived_log WHERE standby_dest = 'NO' GROUP BY thread#;

-- 3. Appliquer au maximum les logs disponibles
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH;
```

#### Procédure Failover (sans Broker)

```sql
-- Sur le STANDBY :

-- Option 1 : Failover avec finish (attendre derniers logs)
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH;
ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;
ALTER DATABASE OPEN;

-- Option 2 : Failover immédiat (si primary totalement inaccessible)
ALTER DATABASE ACTIVATE PHYSICAL STANDBY DATABASE;
ALTER DATABASE OPEN;
```

#### Réintégration de l'ancien primary (REINSTATE)

```bash
# Avec Broker (recommandé) :
DGMGRL> REINSTATE DATABASE 'ORCL_PRI';

# Sans Broker — utiliser RMAN DUPLICATE FROM ACTIVE DATABASE
rman target sys/password@ORCL_NEW_PRI auxiliary sys/password@ORCL_OLD_PRI

RMAN> DUPLICATE TARGET DATABASE FOR STANDBY FROM ACTIVE DATABASE
  DORECOVER
  SPFILE
    SET db_unique_name='ORCL_PRI'
    SET log_archive_dest_2='SERVICE=ORCL_NEW_PRI ASYNC ...'
  NOFILENAMECHECK;
```

## 3.6 Active Data Guard (ADG)

Active Data Guard permet d'ouvrir le standby en lecture pendant que le redo apply continue.

```sql
-- Démarrer le standby en ADG
ALTER DATABASE OPEN READ ONLY;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
  USING CURRENT LOGFILE DISCONNECT FROM SESSION;

-- Vérifier
SELECT open_mode, database_role FROM v$database;
-- Résultat attendu : READ ONLY WITH APPLY | PHYSICAL STANDBY

-- Licence : Active Data Guard nécessite une licence supplémentaire
-- Vérifier le service via SRVCTL pour les connexions ADG
srvctl add service -db ORCL_STB -service ORCL_READ \
  -preferred ORCL_STB1 -role PHYSICAL_STANDBY
```

## 3.7 Snapshot Standby

```sql
-- Convertir physical standby en snapshot standby
-- (pour tests, sans perdre la synchronisation DG)

-- 1. Arrêter le redo apply
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;

-- 2. Créer guaranteed restore point
ALTER DATABASE CONVERT TO SNAPSHOT STANDBY;
-- Cette commande crée automatiquement un GRP interne

-- 3. Ouvrir en lecture/écriture
ALTER DATABASE OPEN;

-- 4. Effectuer les tests...

-- 5. Revenir en Physical Standby
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE CONVERT TO PHYSICAL STANDBY;
ALTER DATABASE OPEN READ ONLY;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
  USING CURRENT LOGFILE DISCONNECT;
```

---

# PARTIE 4 — ARCHITECTURE RAC + DATA GUARD

## 4.1 Topologies

### RAC Primary + RAC Standby (configuration maximale)

```
Site A (Primary)                    Site B (Standby)
┌──────────────────┐               ┌──────────────────┐
│  node1   node2   │               │  node3   node4   │
│  ORCL1   ORCL2   │               │  ORCL3   ORCL4   │
│          │       │               │       │          │
│     RAC Cluster  │ ─── Redo ──▶  │  RAC  Cluster    │
│     + GI + ASM   │               │  + GI + ASM      │
└──────────────────┘               └──────────────────┘
```

### RAC Primary + Single Instance Standby

Configuration courante pour les coûts (RAC licences uniquement sur primary).

### Far Sync Instance

Instance intermédiaire sans datafiles — reçoit les redo synchrones et retransmet async. Permet MAXIMUM AVAILABILITY avec distant standby.

```
Primary ─── SYNC ──▶ Far Sync ─── ASYNC ──▶ Standby
(proche, faible latence)              (distant)
```

```sql
-- Créer Far Sync (sur le Far Sync host)
-- SPFILE Far Sync
*.db_name=ORCL
*.db_unique_name=ORCL_FS
*.log_archive_dest_1='LOCATION=USE_DB_RECOVERY_FILE_DEST VALID_FOR=(ALL_LOGFILES,ALL_ROLES)'
*.log_archive_dest_2='SERVICE=ORCL_STB ASYNC VALID_FOR=(STANDBY_LOGFILES,STANDBY_ROLE)'
*.log_archive_config='DG_CONFIG=(ORCL_PRI,ORCL_FS,ORCL_STB)'

-- Ajouter au Broker
DGMGRL> ADD FAR_SYNC 'ORCL_FS' AS CONNECT IDENTIFIER IS ORCL_FS;
DGMGRL> EDIT DATABASE 'ORCL_PRI' SET PROPERTY RedoRoutes='(LOCAL:ORCL_FS SYNC)(ORCL_FS:ORCL_STB ASYNC)';
```

## 4.2 Paramètres spécifiques RAC+DG

```sql
-- Sur la PRIMARY RAC : tous les threads doivent être configurés
*.log_archive_dest_2='SERVICE=ORCL_STB ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
                      DB_UNIQUE_NAME=ORCL_STB COMPRESSION=ENABLE'

-- SRL sur Primary RAC : (nb ORL par thread + 1) × nb threads
-- Ex : 4 ORL, 2 threads → (4+1) × 2 = 10 SRL minimum

-- Créer les SRL
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1
  ('+REDO') SIZE 500M;   -- répéter pour chaque thread

-- Vérifier SRL par thread
SELECT group#, thread#, sequence#, bytes/1024/1024 mb, archived, status
FROM v$standby_log
ORDER BY thread#, group#;
```

## 4.3 SRVCTL pour RAC+DG

```bash
# Configurer les rôles des services
srvctl add service -db ORCL -service APP_PRI \
  -preferred ORCL1,ORCL2 \
  -role PRIMARY

srvctl add service -db ORCL -service APP_STB \
  -preferred ORCL3,ORCL4 \
  -role PHYSICAL_STANDBY \
  -clbgoal LONG

# Les services PRIMARY ne démarrent que sur la primary
# Les services PHYSICAL_STANDBY ne démarrent que sur le standby (ADG)
```

---

# PARTIE 5 — ORACLE GOLDENGATE 19c

## 5.1 Architecture GoldenGate

Oracle GoldenGate est une solution de réplication logique en temps réel basée sur la capture des redo/archive logs.

```
Source DB                              Target DB
┌─────────┐                           ┌─────────┐
│  Extract │ ─── Trail Files ──────▶  │  Replicat│
│ (REDO)  │          │                │ (DML/DDL)│
└─────────┘     Data Pump             └─────────┘
                (optionnel)
```

### Processus GoldenGate

| Processus | Rôle | Localisation |
|-----------|------|--------------|
| Manager | Superviseur, port listener | Source et Target |
| Extract | Capture redo logs (CDC) | Source |
| Data Pump | Transfert trail files réseau | Source (optionnel) |
| Replicat | Application des changements | Target |
| Collector | Reçoit les trail files | Target |

### Modes Extract

| Mode | Description |
|------|-------------|
| Classic | Lit les redo/archive logs via API Oracle |
| Integrated | Utilise LogMiner (log mining server) — recommandé Oracle 12c+ |
| Initial Load | Chargement initial (snapshot) |

### Modes Replicat

| Mode | Description |
|------|-------------|
| Classic | Application séquentielle |
| Integrated | Utilise le mécanisme d'apply Oracle interne — meilleure performance |
| Parallel | Plusieurs threads d'application |
| Coordinated | Parallel avec coordination des dépendances |

## 5.2 Installation et Configuration GoldenGate

### Prérequis Base de données

```sql
-- Sur la source Oracle 19c
-- 1. Activer Supplemental Logging (OBLIGATOIRE)
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

-- Vérifier
SELECT supplemental_log_data_min FROM v$database;

-- Pour GoldenGate intégré : supplemental logging all columns
-- (ou au minimum pour les colonnes PK/UK)
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
-- Ou par table :
ALTER TABLE schema.table ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;

-- 2. Activer ARCHIVELOG
ARCHIVE LOG LIST;

-- 3. Paramètre ENABLE_GOLDENGATE_REPLICATION (12c+)
ALTER SYSTEM SET enable_goldengate_replication=TRUE SCOPE=BOTH;

-- 4. Créer utilisateur GoldenGate (source)
CREATE USER ggadmin IDENTIFIED BY password
  DEFAULT TABLESPACE users
  TEMPORARY TABLESPACE temp;

GRANT CREATE SESSION TO ggadmin;
GRANT ALTER SESSION TO ggadmin;
GRANT RESOURCE TO ggadmin;
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('GGADMIN', PRIVILEGE_TYPE=>'CAPTURE', GRANT_SELECT_PRIVILEGES=>TRUE);

-- 5. Créer utilisateur GoldenGate (target)
CREATE USER ggrep IDENTIFIED BY password;
GRANT CREATE SESSION, RESOURCE, UNLIMITED TABLESPACE TO ggrep;
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('GGREP', PRIVILEGE_TYPE=>'APPLY');
```

### Fichier de paramètres GLOBALS

```
-- $GG_HOME/GLOBALS
GGSCHEMA ggadmin          -- schéma GoldenGate dans la DB
CHECKPOINTTABLE ggadmin.chkpt
```

## 5.3 GGSCI — Commandes essentielles

```bash
# Démarrer GGSCI
$GG_HOME/ggsci

# ==============================
# Gestion Manager
# ==============================
GGSCI> INFO MGR
GGSCI> START MGR
GGSCI> STOP MGR
GGSCI> VIEW PARAMS MGR

# Fichier params Manager (mgr.prm)
PORT 7809
DYNAMICPORTLIST 7810-7820
AUTOSTART EXTRACT *
AUTOSTART REPLICAT *
AUTORESTART EXTRACT *, RETRIES 5, WAITMINUTES 2
AUTORESTART REPLICAT *, RETRIES 5, WAITMINUTES 2
PURGEOLDEXTRACTS ./dirdat/*, USECHECKPOINTS, MINKEEPDAYS 3

# ==============================
# Extract
# ==============================
GGSCI> ADD EXTRACT EXT1, INTEGRATED TRANLOG, BEGIN NOW
GGSCI> ADD EXTTRAIL ./dirdat/lt, EXTRACT EXT1, MEGABYTES 500
GGSCI> START EXTRACT EXT1
GGSCI> STOP EXTRACT EXT1
GGSCI> INFO EXTRACT EXT1, DETAIL
GGSCI> STATS EXTRACT EXT1
GGSCI> LAG EXTRACT EXT1
GGSCI> SEND EXTRACT EXT1, STATUS
GGSCI> SEND EXTRACT EXT1, LOGEND

# Fichier params Extract (ext1.prm)
EXTRACT EXT1
USERID ggadmin, PASSWORD password
EXTTRAIL ./dirdat/lt
LOGALLSUPCOLS
UPDATERECORDFORMAT COMPACT
TABLE HR.*;
TABLE SALES.*;
-- Exclure certaines tables
TABLEEXCLUDE HR.TMP_*

# ==============================
# Data Pump
# ==============================
GGSCI> ADD EXTRACT DPM1, EXTTRAILSOURCE ./dirdat/lt
GGSCI> ADD RMTTRAIL ./dirdat/rt, EXTRACT DPM1, MEGABYTES 500
GGSCI> START EXTRACT DPM1
GGSCI> INFO EXTRACT DPM1

# Fichier params Data Pump (dpm1.prm)
EXTRACT DPM1
PASSTHRU
RMTHOST target-host, MGRPORT 7809, COMPRESS
RMTTRAIL ./dirdat/rt
TABLE HR.*;
TABLE SALES.*;

# ==============================
# Replicat
# ==============================
GGSCI> ADD CHECKPOINTTABLE ggadmin.chkpt
GGSCI> ADD REPLICAT REP1, INTEGRATED, EXTTRAIL ./dirdat/rt, CHECKPOINTTABLE ggadmin.chkpt
GGSCI> START REPLICAT REP1
GGSCI> STOP REPLICAT REP1
GGSCI> INFO REPLICAT REP1, DETAIL
GGSCI> STATS REPLICAT REP1
GGSCI> LAG REPLICAT REP1
GGSCI> SEND REPLICAT REP1, STATUS

# Fichier params Replicat (rep1.prm)
REPLICAT REP1
USERID ggrep, PASSWORD password
ASSUMETARGETDEFS                        -- si source=target structure
-- ou
SOURCEDEFS ./dirdef/src_def.def        -- si structures différentes

HANDLECOLLISIONS                        -- pendant chargement initial
REPERROR (DEFAULT, DISCARD)
DISCARDFILE ./dirrpt/rep1_discard.txt, APPEND, MEGABYTES 100

MAP HR.*, TARGET HR.*;
MAP SALES.ORDERS, TARGET SALES.ORDERS,
  COLMAP (USEDEFAULTS,
          LOAD_DATE = @DATENOW());

# ==============================
# Monitoring
# ==============================
GGSCI> INFO ALL                         # statut tous processus
GGSCI> STATUS ALL
GGSCI> VIEW REPORT EXT1                 # rapport d'exécution
GGSCI> VIEW GGSEVT                      # event log GoldenGate
GGSCI> DBLOGIN USERID ggadmin, PASSWORD password
GGSCI> INFO TRANDATA HR.EMPLOYEES       # vérifier supplemental logging
GGSCI> ADD TRANDATA HR.EMPLOYEES        # ajouter supplemental logging
```

## 5.4 Topologies GoldenGate

### Unidirectionnel (DR / migration)

```
Source ──▶ Extract ──▶ Trail ──▶ Data Pump ──▶ Trail ──▶ Replicat ──▶ Target
```

### Bidirectionnel (Active-Active)

```
Site A ◀──▶ Site B
Chaque site est source ET target.
CRITICAL : Boucles infinies — utiliser LOOPBACK ou filtrage par nom source.
```

```
# Sur site A Extract, exclure les transactions répliquées depuis B
TRANLOGOPTIONS EXCLUDEUSER ggrep   -- exclure l'utilisateur replicat de B
```

### Cascade (Hub and Spoke)

```
Source ──▶ Hub ──▶ Target1
               ──▶ Target2
               ──▶ Target3
```

### Consolidation (Many-to-One)

```
Source1 ──▶ \
Source2 ──▶  ──▶ Target (entrepôt de données)
Source3 ──▶ /
```

## 5.5 GoldenGate Microservices Architecture (MA) — 19c

GoldenGate 19c peut être déployé en architecture Microservices (REST API, interface web).

```bash
# Service Registration Server (SRS) — port 9100 par défaut
# Service Manager (SM) — 1 par deployment
# Distribution Server — remplace le Data Pump
# Receiver Server — remplace le Collector
# Performance Metrics Server

# Accès interface web
https://host:9100/

# API REST
curl -X GET http://host:9100/services/v2/deployments \
  -H "Authorization: Basic base64(user:pass)"
```

---

# PARTIE 6 — TROUBLESHOOTING

## 6.1 Clusterware / Grid Infrastructure

### Commandes de diagnostic

```bash
# ==============================
# Logs principaux
# ==============================
# Répertoire des logs GI (Oracle 19c)
$ORACLE_BASE/diag/crs/<hostname>/crs/trace/

# Logs alertes cluster
tail -f $ORACLE_BASE/diag/crs/<hostname>/crs/trace/alert.log

# Logs CSSD (heartbeat)
ls $ORACLE_BASE/diag/crs/<hostname>/crs/trace/ocssd*.trc

# Logs CRSD
ls $ORACLE_BASE/diag/crs/<hostname>/crs/trace/crsd*.trc

# Outil d'analyse des logs
$GRID_HOME/bin/diagcollect.pl -crs      # collecte diagnostic CRS
$GRID_HOME/bin/tfactl diagcollect       # Oracle TFA (Trace File Analyzer)

# ==============================
# Problèmes courants
# ==============================

# Nœud éjecté du cluster (node eviction)
# → Vérifier : interconnect, CSS timeout, I/O disks
grep -i "evict\|reboot\|shoot\|expel" $ORACLE_BASE/diag/crs/*/crs/trace/ocssd.trc

# CSS disktiomeout (19c défaut : 200s en NFS, 60s en direct)
crsctl get css disktimeout
crsctl set css disktimeout 60

# CSS misscount (détection nœud down, défaut 30s)
crsctl get css misscount
crsctl set css misscount 30

# Vérifier interconnect
oifcfg getif
ping -I <private_ip> <peer_private_ip> -c 100
# Latence attendue < 1ms, perte paquets = 0

# ==============================
# Ressource qui ne démarre pas
# ==============================
crsctl stat res ora.ORCL.db -p          # paramètres ressource
crsctl stat res ora.ORCL.db -f          # état complet

# Forcer le redémarrage d'une ressource
crsctl start res ora.ORCL1.inst

# Debug ressource
crsctl debug log res "ora.ORCL.db:5"
```

### Problèmes ASM courants

```bash
# Disk group ne monte pas
# → Vérifier les disques visibles
ls -la /dev/oracleasm/disks/
oracleasm listdisks
oracleasm querydisk -d /dev/sdb1

# Rebalance bloqué
SELECT * FROM v$asm_operation;
-- Augmenter power
ALTER DISKGROUP DATA REBALANCE POWER 16;

# Disque hors ligne
SELECT group_number, name, mode_status, state, path
FROM v$asm_disk WHERE mode_status != 'ONLINE';

-- Remettre en ligne
ALTER DISKGROUP DATA ONLINE DISK 'ORCL:DISK01';
-- ou
ALTER DISKGROUP DATA ONLINE ALL;

# Corruption ASM
ASMCMD> md_backup -b /tmp/backup.xml -g DATA
ASMCMD> amdu -diskstring '/dev/oracleasm/disks/*' -extract DATA   # extraction d'urgence
```

## 6.2 RAC

### Problèmes d'interconnect

```bash
# ==============================
# Identifier les problèmes Cache Fusion
# ==============================

-- Waits gc élevés → problème interconnect
SELECT event, total_waits, time_waited_micro/1000 ms,
       average_wait
FROM gv$system_event
WHERE event LIKE 'gc%'
AND inst_id = 1
ORDER BY time_waited_micro DESC;

-- Average wait gc cr blocks > 10ms = problème réseau
-- Objectif : < 1ms avec 10GbE

-- Identifier les hot blocks
SELECT inst_id, file#, block#, class#, gc_cr_blocks_received,
       gc_current_blocks_received
FROM gv$cache_transfer
WHERE gc_cr_blocks_received > 1000
ORDER BY gc_cr_blocks_received DESC;

# Vérifier les erreurs interconnect OS
netstat -s | grep -i error
ip -s link show <private_interface>

# Test bande passante interconnect
iperf3 -s   # sur nœud 1
iperf3 -c node1-priv -t 60   # sur nœud 2
```

### Problèmes de voting disk / split-brain

```bash
# Vérifier le quorum
crsctl query css votedisk

# Si moins de majorité des voting disks accessibles → cluster s'arrête
# Vérifier les chemins I/O vers les voting disks
dd if=/dev/<voting_disk> of=/dev/null bs=512 count=1

# Démarrage forcé (urgence uniquement, risque de corruption)
crsctl start crs -excl     # ne fonctionne que si 1 nœud
```

## 6.3 Data Guard

### Problèmes courants Data Guard

```bash
# ==============================
# Vérification complète santé DG
# ==============================
dgmgrl sys@primary
DGMGRL> SHOW CONFIGURATION;
# Chercher : Warning ou Error

# ==============================
# Apply lag élevé
# ==============================
-- Vérifier le lag
SELECT name, value FROM v$dataguard_stats WHERE name = 'apply lag';

-- Vérifier le process MRP
SELECT process, status, sequence# FROM v$managed_standby WHERE process LIKE 'MRP%';

-- Redémarrer le MRP
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
  USING CURRENT LOGFILE DISCONNECT FROM SESSION;

-- Augmenter le parallélisme d'apply
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
  PARALLEL 8 DISCONNECT FROM SESSION;

# ==============================
# Archive log gap
# ==============================
-- Sur la standby
SELECT * FROM v$archive_gap;

-- Résolution manuelle du gap (si FAL ne résout pas)
-- Sur le primary :
ALTER SYSTEM ARCHIVE LOG CURRENT;

-- Copie manuelle si nécessaire
rman target sys@primary
RMAN> BACKUP ARCHIVELOG SEQUENCE BETWEEN 1000 AND 1050 THREAD 1;
-- Puis copier et enregistrer sur le standby

# ==============================
# ORA-16786 / ORA-16787 (redo transport)
# ==============================
-- Vérifier les erreurs d'archive dest
SELECT dest_id, status, error FROM v$archive_dest WHERE target='STANDBY';

-- Reset la destination
ALTER SYSTEM SET log_archive_dest_state_2=DEFER;
ALTER SYSTEM SET log_archive_dest_state_2=ENABLE;

# ==============================
# Standby ne reçoit plus de redo
# ==============================
-- Vérifier la connectivité TNS
tnsping ORCL_STB

-- Vérifier le RFS sur le standby
SELECT process, status, client_process FROM v$managed_standby WHERE process='RFS';

-- Sur le primary, forcer la reconnexion
ALTER SYSTEM SWITCH LOGFILE;

# ==============================
# Fichier de log DG Broker
# ==============================
# $ORACLE_BASE/diag/rdbms/<db_unique_name>/<instance>/trace/drc*.log
```

## 6.4 GoldenGate

### Abend (arrêt anormal) et résolution

```bash
# ==============================
# Diagnostics Extract en abend
# ==============================
GGSCI> VIEW REPORT EXT1           # rapport d'erreur
GGSCI> VIEW GGSEVT                # logs événements

# Causes courantes :
# OGG-00664 : archive log non disponible → étendre la rétention
# OGG-01201 : timeout connexion DB
# OGG-00868 : supplemental logging insuffisant

# Redémarrer après correction
GGSCI> START EXTRACT EXT1

# ==============================
# Diagnostics Replicat en abend
# ==============================
# Erreur de clé dupliquée (OGG-01403)
GGSCI> SEND REPLICAT REP1, SKIPTRANSACTION  # sauter la transaction conflictuelle

# Ou via fichier params :
# HANDLECOLLISIONS   -- ignorer INSERT en doublon / DELETE de rang manquant

# Resynchronisation complète
GGSCI> STOP REPLICAT REP1
GGSCI> ALTER REPLICAT REP1, EXTSEQNO 1, EXTRBA 0  -- repositionner
GGSCI> START REPLICAT REP1

# ==============================
# Vérifier le lag
# ==============================
GGSCI> LAG EXTRACT EXT1
GGSCI> LAG REPLICAT REP1

-- Vérification SQL côté source
SELECT * FROM dba_goldengate_support_mode;
SELECT capture_name, state, captured_scn, applied_scn,
       (SYSDATE - CAPTURED_SCN_TIME)*86400 lag_seconds
FROM dba_capture;
```

---

# PARTIE 7 — CHECKLIST OPÉRATIONNELLES

## 7.1 Checklist Santé Quotidienne (Health Check)

```sql
-- ==============================
-- 1. Cluster Clusterware
-- ==============================
-- (bash)
crsctl check cluster -all
crsctl status res -t | grep -v ONLINE

-- ==============================
-- 2. ASM
-- ==============================
SELECT name, state, type, total_mb, free_mb,
       ROUND(free_mb/total_mb*100,1) pct_free,
       offline_disks
FROM v$asm_diskgroup;
-- Alerte si pct_free < 15% ou offline_disks > 0

-- ==============================
-- 3. Instances RAC
-- ==============================
SELECT inst_id, instance_name, status, database_status
FROM gv$instance;
-- Toutes doivent être OPEN/ACTIVE

-- ==============================
-- 4. Data Guard
-- ==============================
SELECT db_unique_name, open_mode, protection_mode,
       switchover_status
FROM v$database;

SELECT name, value FROM v$dataguard_stats
WHERE name IN ('apply lag', 'transport lag');
-- apply lag < 5 minutes idéalement

SELECT * FROM v$archive_gap;
-- Doit être vide

-- ==============================
-- 5. Space
-- ==============================
-- Espace tablespaces
SELECT tablespace_name,
       ROUND(used_space * 8192 / 1073741824, 2) used_gb,
       ROUND(tablespace_size * 8192 / 1073741824, 2) total_gb,
       ROUND(used_percent, 1) pct_used
FROM dba_tablespace_usage_metrics
ORDER BY pct_used DESC;
-- Alerte si pct_used > 85%

-- FRA space
SELECT * FROM v$recovery_area_usage;
SELECT space_limit/1073741824 limit_gb,
       space_used/1073741824 used_gb,
       space_reclaimable/1073741824 reclaimable_gb
FROM v$recovery_file_dest;

-- ==============================
-- 6. Jobs Scheduler
-- ==============================
SELECT job_name, status, error#, actual_start_date, run_duration
FROM dba_scheduler_job_run_details
WHERE actual_start_date > SYSDATE - 1
AND (status != 'SUCCEEDED' OR error# != 0)
ORDER BY actual_start_date DESC;

-- ==============================
-- 7. Alert log (erreurs ORA)
-- ==============================
SELECT originating_timestamp, message_text
FROM v$diag_alert_ext
WHERE originating_timestamp > SYSDATE - 1/24  -- dernière heure
AND message_text LIKE 'ORA-%'
ORDER BY originating_timestamp DESC;
```

## 7.2 Procédure de Patch (GI + RDBMS)

### Ordre de patch avec RAC + Data Guard

```
1. Patcher le standby en premier (GI puis RDBMS)
   → Réduire la fenêtre d'exposition du primary
2. Switcher primary → standby (patcher devenu nouveau standby)
3. Patcher l'ancien primary

Ordre sur chaque site :
  a. Arrêter RDBMS Homes (toutes instances du nœud)
  b. Appliquer patch RDBMS (OPatch)
  c. Démarrer instances
  d. Répéter sur autres nœuds (rolling patch si possible)
  e. Arrêter GI Home (nœud par nœud)
  f. Appliquer patch GI (OPatch en root)
  g. Démarrer GI
```

```bash
# Vérifier les patches appliqués
$ORACLE_HOME/OPatch/opatch lspatches
$GRID_HOME/OPatch/opatch lspatches

# Vérifier les conflits avant application
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail \
  -ph /tmp/patch_dir

# Appliquer un patch RU (Release Update) en RAC
$ORACLE_HOME/OPatch/opatchauto apply /tmp/patch_dir \
  -oh $ORACLE_HOME \
  -target_type rac_rolling   # rolling = nœud par nœud sans downtime
```

## 7.3 RMAN — Sauvegarde avec Data Guard

```bash
# Backup depuis le standby (décharger le primary — RECOMMANDÉ)
rman target sys/password@ORCL_STB catalog rcat/password@RCAT

RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
RMAN> BACKUP DATABASE SECTION SIZE 32G;  -- multisection pour gros fichiers

# Backup incrémental (stratégie BCT recommandée)
-- Sur le primary (une fois) :
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING
  USING FILE '+DATA/ORCL_PRI/bct.dbf';

-- Puis backups incrémentaux efficaces depuis standby
RMAN> BACKUP INCREMENTAL LEVEL 1 FOR RECOVER OF COPY
      WITH TAG 'incr_update'
      DATABASE;
RMAN> RECOVER COPY OF DATABASE WITH TAG 'incr_update';

# Vérification du backup catalog
RMAN> LIST BACKUP SUMMARY;
RMAN> VALIDATE BACKUPSET <bs_key>;
```

---

# PARTIE 8 — RÉFÉRENCE RAPIDE

## 8.1 Ports par défaut

| Service | Port |
|---------|------|
| Oracle Listener | 1521 |
| SCAN Listener | 1521 |
| ASM Listener | 1521 (ou GI listener) |
| Oracle EM | 5500 (HTTPS) |
| Oracle Clusterware | 49896 |
| GoldenGate Manager | 7809 |
| GoldenGate Microservices | 9100 |
| ONS (Oracle Notification Service) | 6200 |

## 8.2 Variables d'environnement essentielles

```bash
# Grid Infrastructure
export ORACLE_BASE=/u01/app/oracle
export GRID_HOME=/u01/app/19.0.0.0/grid
export ORACLE_HOME=$GRID_HOME
export PATH=$ORACLE_HOME/bin:$PATH

# Database
export ORACLE_HOME=/u01/app/oracle/product/19.0.0.0/dbhome_1
export ORACLE_SID=ORCL1
export PATH=$ORACLE_HOME/bin:$PATH

# GoldenGate
export GG_HOME=/u01/app/goldengate/19.1
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$GG_HOME
export PATH=$GG_HOME:$PATH
```

## 8.3 MOS Notes de référence

| Sujet | MOS Note |
|-------|----------|
| Data Guard Best Practices | 1617442.1 |
| RAC Interconnect Best Practices | 341788.1 |
| GoldenGate 19c Installation Guide | 2573562.1 |
| CSS Voting Disk / OCR | 1073073.6 |
| ASM Best Practices | 265633.1 |
| Patching RAC/DG Guide | 244241.1 |
| FSFO Best Practices | 1305019.1 |
| GoldenGate Performance Best Practices | 1329926.1 |
| Node Eviction Root Cause Analysis | 1368995.1 |
| Data Guard 19c New Features | 2485457.1 |

## 8.4 Chemins des logs importants

```bash
# Alert log DB
$ORACLE_BASE/diag/rdbms/<db_unique_name>/<instance_name>/trace/alert_<instance>.log

# Trace files DB
$ORACLE_BASE/diag/rdbms/<db_unique_name>/<instance_name>/trace/

# Alert log GI / CRS
$ORACLE_BASE/diag/crs/<hostname>/crs/trace/alert.log

# Logs ASM
$ORACLE_BASE/diag/asm/+asm/<instance>/trace/alert_<instance>.log

# Logs Listener
$ORACLE_BASE/diag/tnslsnr/<hostname>/<listener_name>/trace/

# Logs GoldenGate
$GG_HOME/dirrpt/          # rapports Extract/Replicat
$GG_HOME/ggserr.log       # log principal GoldenGate

# Accès via ADRCI
adrci> show homes
adrci> set homepath diag/rdbms/orcl_pri/orcl1
adrci> show alert -tail 100
```

---

*Document généré pour usage interne — Oracle 19c HA/DR Knowledge Base*
*Version : 1.0 — Couverture : Grid Infrastructure, RAC, Data Guard, GoldenGate*

---

# PARTIE 9 — SYSTÈME : RHEL / OEL & WINDOWS SERVER

## 9.1 Linux — RHEL / Oracle Linux (OEL) pour Oracle HA

### Prérequis OS Oracle 19c RAC / GI

```bash
# ==============================
# Vérification prérequis (cluvfy)
# ==============================
$GRID_HOME/runcluvfy.sh stage -pre crsinst \
  -n node1,node2,node3 \
  -fixup -verbose

# ==============================
# Packages RPM obligatoires (RHEL/OEL 7/8)
# ==============================
# Groupe complet recommandé :
yum install -y oracle-database-preinstall-19c
# Ce meta-package installe automatiquement tous les prérequis
# (kernel params, limits, packages, users/groups)

# Packages individuels si meta-package non disponible :
yum install -y \
  bc binutils compat-libcap1 compat-libstdc++-33 \
  elfutils-libelf elfutils-libelf-devel fontconfig-devel \
  glibc glibc-devel ksh libaio libaio-devel \
  libdtrace-ctf-devel libXrender libXrender-devel libX11 \
  libXau libXi libXtst libgcc librdmacm-devel libstdc++ \
  libstdc++-devel libxcb make net-tools nfs-utils \
  python python-configshell python-rtslib python-six \
  targetcli smartmontools sysstat unixODBC

# OEL spécifique (UEK kernel recommandé pour RAC)
yum install -y kernel-uek kernel-uek-devel
```

### Paramètres Kernel (/etc/sysctl.conf)

```bash
# /etc/sysctl.d/97-oracle-database-sysctl.conf
# (généré automatiquement par oracle-database-preinstall-19c)

# Shared Memory
kernel.shmall = 1073741824        # pages mémoire partagée totales
kernel.shmmax = 4398046511104     # max segment mémoire partagée (bytes)
kernel.shmmni = 4096

# Semaphores : SEMMSL SEMMNS SEMOPM SEMMNI
kernel.sem = 250 32000 100 128

# Réseau
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576

# Interconnect RAC (optimisation)
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# Huge Pages (recommandé pour SGA > 4GB)
vm.nr_hugepages = 4096            # calcul : SGA_total_MB / 2 + 10%
vm.hugetlb_shm_group = 54321      # GID du groupe oinstall

# Disable THP (Transparent Huge Pages) — OBLIGATOIRE Oracle
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# Rendre persistant via grub ou rc.local

# Panic on hung tasks (évite les nœuds zombies)
kernel.hung_task_timeout_secs = 0  # désactiver

# Appliquer sans redémarrage
sysctl -p /etc/sysctl.d/97-oracle-database-sysctl.conf
```

### Limites utilisateur (/etc/security/limits.conf)

```bash
# /etc/security/limits.d/oracle-database-preinstall-19c.conf
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc     2047
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   soft   memlock   134217728   # pour Huge Pages (en KB)
oracle   hard   memlock   134217728
grid     soft   nofile    1024
grid     hard   nofile    65536
grid     soft   nproc     2047
grid     hard   nproc     16384
grid     soft   memlock   134217728
grid     hard   memlock   134217728
```

### Utilisateurs, Groupes et Structure de répertoires

```bash
# ==============================
# Groupes OS
# ==============================
groupadd -g 54321 oinstall     # Oracle Inventory (primaire grid+oracle)
groupadd -g 54322 dba          # OSDBA (SYSDBA)
groupadd -g 54323 oper         # OSOPER
groupadd -g 54324 backupdba    # OSBACKUPDBA
groupadd -g 54325 dgdba        # OSDGDBA
groupadd -g 54326 kmdba        # OSKMDBA
groupadd -g 54327 asmdba       # OSASMDBA (ASM access)
groupadd -g 54328 asmoper      # OSOPER for ASM
groupadd -g 54329 asmadmin     # OSASM (SYSASM)
groupadd -g 54330 racdba       # pour RAC

# ==============================
# Utilisateurs
# ==============================
# Grid user (GI ownership)
useradd -u 54321 -g oinstall -G asmadmin,asmdba,asmoper,dba grid

# Oracle user (RDBMS ownership)
useradd -u 54322 -g oinstall -G dba,oper,backupdba,dgdba,kmdba,asmdba oracle

# ==============================
# Structure de répertoires
# ==============================
mkdir -p /u01/app/grid
mkdir -p /u01/app/19.0.0.0/grid          # GRID_HOME
mkdir -p /u01/app/oracle
mkdir -p /u01/app/oracle/product/19.0.0.0/dbhome_1  # ORACLE_HOME
mkdir -p /u01/app/oraInventory

chown -R grid:oinstall /u01/app/grid
chown -R grid:oinstall /u01/app/19.0.0.0
chown -R oracle:oinstall /u01/app/oracle
chown -R grid:oinstall /u01/app/oraInventory
chmod -R 775 /u01
```

### Transparent Huge Pages — Désactivation GRUB

```bash
# RHEL/OEL 7 : via grubby
grubby --update-kernel=ALL \
  --args="transparent_hugepage=never numa=off"

# RHEL/OEL 8 : via grub2-mkconfig
# Éditer /etc/default/grub :
# GRUB_CMDLINE_LINUX="... transparent_hugepage=never numa=off"
grub2-mkconfig -o /boot/grub2/grub.cfg

# Vérification
cat /sys/kernel/mm/transparent_hugepage/enabled
# Résultat attendu : always madvise [never]

# NUMA — désactivé pour RAC (ou configurer numa_balancing=0)
echo 0 > /proc/sys/kernel/numa_balancing
```

### Services systemd critiques pour Oracle

```bash
# Désactiver NTP (utiliser chrony avec synchronisation cluster)
systemctl disable ntpd
systemctl enable chronyd
systemctl start chronyd

# Vérifier synchronisation temps (CRITIQUE pour RAC)
chronyc sources -v
chronyc tracking
# Offset doit être < 1000ms, idéalement < 100ms

# Firewalld — désactiver ou configurer les ports Oracle
systemctl stop firewalld
systemctl disable firewalld
# OU configurer les zones firewalld :
firewall-cmd --permanent --add-port=1521/tcp    # listener
firewall-cmd --permanent --add-port=7809/tcp    # GoldenGate manager
firewall-cmd --permanent --add-port=6200/tcp    # ONS
firewall-cmd --reload

# SELinux — désactiver ou configurer
# /etc/selinux/config : SELINUX=disabled (reboot requis)
# Ou permissive temporairement :
setenforce 0

# NetworkManager — peut interférer avec les VIPs Oracle
# Sur les interfaces Clusterware : configurer NM_CONTROLLED=no
# /etc/sysconfig/network-scripts/ifcfg-eth1 :
# NM_CONTROLLED="no"
# PEERDNS=no
```

### Surveillance OS (commandes utiles DBA)

```bash
# ==============================
# CPU & Mémoire
# ==============================
top -H -p $(pgrep -d',' oracle)    # threads Oracle
vmstat 1 10                         # activité VM
numastat -p oracle                  # NUMA stats par process Oracle
numactl --hardware                  # topology NUMA

# Huge Pages utilisées
grep -i huge /proc/meminfo
# HugePages_Total doit == HugePages_Rsvd + HugePages_Free

# ==============================
# I/O
# ==============================
iostat -xm 2 10                     # I/O par device (-x extended)
iotop -o -d 2                       # processus consommateurs I/O
blktrace -d /dev/sdb -o trace       # trace I/O bas niveau

# Latence disque (objectif Oracle : < 1ms read, < 5ms write)
dd if=/dev/sdb of=/dev/null bs=512 count=10000 iflag=direct

# ==============================
# Réseau
# ==============================
ss -s                               # stats sockets
netstat -i                          # erreurs par interface
ethtool -S eth1 | grep -i error     # compteurs hardware NIC
sar -n DEV 1 5                      # throughput réseau

# ==============================
# Processus Oracle
# ==============================
ps -ef | grep ora_ | grep -v grep   # processus background Oracle
pmap -x $(pgrep ora_dbw0)           # memory map processus
strace -p $(pgrep ora_lgwr) -e trace=write  # syscalls LGWR
```

## 9.2 Windows Server pour Oracle

### Spécificités Oracle sur Windows

```powershell
# ==============================
# Services Oracle Windows
# ==============================
# Lister les services Oracle
Get-Service -Name "Oracle*" | Format-Table Name, Status, StartType

# Démarrer/arrêter
Start-Service "OracleServiceORCL"
Stop-Service "OracleServiceORCL" -Force

# Services importants :
# OracleServiceSID          — instance DB principale
# OracleOraDB19Home1TNSListener — listener
# OracleRemExecServiceV2    — remote execution
# OracleVssWriterORCL       — VSS writer (backup)
# OracleJobSchedulerORCL    — scheduler jobs

# ==============================
# Utilitaires Oracle en ligne de commande
# ==============================
# Variables d'environnement
set ORACLE_HOME=C:\oracle\product\19.0.0\dbhome_1
set ORACLE_SID=ORCL
set PATH=%ORACLE_HOME%\bin;%PATH%

# Connexion SQL*Plus
sqlplus / as sysdba
sqlplus sys/password@ORCL as sysdba

# ORADIM — gestion instances Windows
oradim -new -sid ORCL -startmode auto   # créer instance
oradim -edit -sid ORCL -startmode manual
oradim -delete -sid ORCL

# ==============================
# ASM sur Windows (rare, mais possible)
# ==============================
# ASMTOOL pour gérer les disques ASM
asmtoolg                              # interface graphique
asmtool -addms \\.\PHYSICALDRIVE1     # ajouter disk ASM

# ==============================
# Registre Windows Oracle
# ==============================
# HKLM\SOFTWARE\ORACLE\KEY_OraDB19Home1
reg query "HKLM\SOFTWARE\ORACLE" /s

# ==============================
# Event Log Oracle
# ==============================
# Les erreurs Oracle apparaissent dans Windows Event Viewer
# Source : Oracle.DBConsole.ORCL ou OracleServiceORCL
Get-EventLog -LogName Application -Source "Oracle*" -Newest 50
```

### Oracle RAC sur Windows (rare mais existant)

```powershell
# GI sur Windows utilise les mêmes concepts
# Clusterware s'appuie sur Windows Server Failover Clustering (WSFC)
# ou fonctionne indépendamment

# Vérifier les ressources cluster Windows
Get-ClusterResource | Where-Object {$_.Name -like "*Oracle*"}

# Disques partagés (Cluster Shared Volumes ou disques bruts)
Get-ClusterSharedVolume
Get-Disk | Where-Object {$_.OperationalStatus -eq "Online"}

# Interconnect — recommandation Windows :
# Désactiver TCP Chimney, RSS sur les interfaces interconnect
netsh int tcp set global chimney=disabled
netsh int tcp set global rss=disabled
Set-NetAdapterAdvancedProperty -Name "PrivateNIC" -DisplayName "RSS" -DisplayValue "Disabled"
```

---

# PARTIE 10 — RÉSEAU

## 10.1 Ethernet — Configuration pour Oracle HA

### Bonding / Teaming pour les interfaces publiques

```bash
# ==============================
# Linux Bonding (kernel bonding)
# ==============================
# Mode recommandé pour public : mode 1 (active-backup) ou mode 4 (802.3ad LACP)
# Mode recommandé pour interconnect : mode 4 (LACP) ou RDMA direct

# /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE=bond0
TYPE=Bond
BONDING_MASTER=yes
BOOTPROTO=none
ONBOOT=yes
IPADDR=10.10.1.101
PREFIX=24
BONDING_OPTS="mode=4 miimon=100 lacp_rate=fast xmit_hash_policy=layer3+4"
NM_CONTROLLED=no

# /etc/sysconfig/network-scripts/ifcfg-eth0 (esclave)
DEVICE=eth0
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=no

# /etc/sysconfig/network-scripts/ifcfg-eth1 (esclave)
DEVICE=eth1
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=no

# Vérifier le bonding
cat /proc/net/bonding/bond0
bonding-mode: IEEE 802.3ad Dynamic link aggregation
MII Status: up
Speed: 10000 Mbps

# ==============================
# NetworkManager Teaming (RHEL8+)
# ==============================
nmcli con add type team con-name team0 ifname team0 \
  config '{"runner":{"name":"lacp","active":true,"fast_rate":true}}'
nmcli con add type team-slave con-name team0-port1 ifname eth0 master team0
nmcli con add type team-slave con-name team0-port2 ifname eth1 master team0
nmcli con mod team0 ipv4.addresses "10.10.1.101/24" ipv4.method manual
nmcli con up team0

# Statut
teamdctl team0 state
```

### Optimisation NIC pour interconnect Oracle RAC

```bash
# ==============================
# Tuning NIC (10/25 GbE)
# ==============================
# Jumbo Frames — OBLIGATOIRE pour interconnect RAC performant
ip link set eth2 mtu 9000
# Rendre persistant dans ifcfg-eth2 : MTU=9000
# Vérifier sur TOUS les nœuds et switchs

# Vérifier MTU end-to-end
ping -M do -s 8972 <peer_private_ip>  # 8972 + 28 (IP+ICMP) = 9000

# Ring buffers (réduire les drops)
ethtool -g eth2
ethtool -G eth2 rx 4096 tx 4096

# Interrupt coalescing (réduire la latence)
ethtool -c eth2
ethtool -C eth2 rx-usecs 50 tx-usecs 50

# CPU affinity pour les IRQ réseau (isoler les cœurs)
# Trouver les IRQ de l'interface :
cat /proc/interrupts | grep eth2
# Puis affecter les IRQ à des cœurs dédiés
echo 4 > /proc/irq/<irq_number>/smp_affinity_list

# Vérifier les erreurs NIC
ethtool -S eth2 | grep -iE "error|drop|miss|over"

# ==============================
# Flow Control (Ethernet PAUSE)
# ==============================
# Désactiver le flow control sur les interfaces interconnect Oracle
# (peut causer des pauses globales sur le switch)
ethtool -A eth2 rx off tx off
# Persistant via /etc/rc.local ou udev rules

# ==============================
# Vérification bande passante
# ==============================
iperf3 -s -B <private_ip>           # serveur (nœud 1)
iperf3 -c <node1_private_ip> \
  -B <local_private_ip> \
  -t 60 -P 4 -i 5 \
  --get-server-output                # client (nœud 2)
# Objectif : > 9 Gbps sur 10 GbE
```

### VLAN pour isolation du trafic Oracle

```bash
# ==============================
# Configuration VLAN Linux
# ==============================
# Installer le module 8021q
modprobe 8021q
echo "8021q" >> /etc/modules-load.d/oracle.conf

# Créer interface VLAN
ip link add link eth0 name eth0.100 type vlan id 100
ip addr add 192.168.100.101/24 dev eth0.100
ip link set eth0.100 up

# Persistant : /etc/sysconfig/network-scripts/ifcfg-eth0.100
DEVICE=eth0.100
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.100.101
PREFIX=24
VLAN=yes
NM_CONTROLLED=no

# ==============================
# Architecture VLAN recommandée Oracle
# ==============================
# VLAN 100 : Public (clients, SCAN, VIP)       → 10.10.1.0/24
# VLAN 200 : Interconnect RAC (Cache Fusion)   → 192.168.1.0/24 (MTU 9000)
# VLAN 300 : Stockage (iSCSI / NFS)            → 10.20.1.0/24 (MTU 9000)
# VLAN 400 : Backup / RMAN                     → 10.30.1.0/24
# VLAN 500 : DG Redo Transport (inter-sites)   → 10.40.1.0/24

# ==============================
# Routage inter-sites (Data Guard WAN)
# ==============================
# Route statique vers site DR
ip route add 10.10.2.0/24 via 10.10.1.254 dev bond0
# Persistant : /etc/sysconfig/network-scripts/route-bond0
10.10.2.0/24 via 10.10.1.254

# QoS pour le redo transport Data Guard
# Marquer le trafic Oracle (port 1521) avec DSCP EF (Expedited Forwarding)
tc qdisc add dev bond0 root handle 1: prio
tc filter add dev bond0 parent 1: protocol ip u32 \
  match ip dport 1521 0xffff flowid 1:1
```

## 10.2 InfiniBand pour Oracle RAC

InfiniBand est l'interconnect le plus performant pour Oracle RAC, utilisé notamment dans Oracle Exadata et les configurations haute performance.

### Architecture IB pour RAC

```bash
# ==============================
# Composants IB
# ==============================
# HCA (Host Channel Adapter) — carte IB dans le serveur
# IB Switch — commutateur InfiniBand (ex: Mellanox/NVIDIA)
# Subnet Manager (SM) — gère la fabric (OpenSM ou switch intégré)

# Vitesses IB :
# SDR  : 10 Gbps
# DDR  : 20 Gbps
# QDR  : 40 Gbps
# FDR  : 56 Gbps
# EDR  : 100 Gbps ← courant
# HDR  : 200 Gbps
# NDR  : 400 Gbps

# ==============================
# Installation OFED (OpenFabrics Enterprise Distribution)
# ==============================
# RHEL/OEL inclut RDMA core et pilotes Mellanox
yum install -y rdma-core libibverbs libibverbs-utils \
  librdmacm librdmacm-utils infiniband-diags perftest \
  opensm opensm-libs

# Démarrer le Subnet Manager (si pas de SM intégré au switch)
systemctl enable opensm
systemctl start opensm

# ==============================
# Vérification IB
# ==============================
ibstat                              # statut HCA
ibstatus                            # statut port
ibv_devinfo                         # infos device RDMA
iblinkinfo                          # topologie fabric
ibping -S                           # serveur ping IB
ibping -G <GUID_destination>        # client ping IB

# Performances IB (test bande passante)
ib_send_bw -d mlx5_0 -x 1         # serveur
ib_send_bw -d mlx5_0 -x 1 <peer>  # client
# Objectif EDR : > 95 Gbps bidirectionnel

# Latence
ib_send_lat -d mlx5_0 <peer>
# Objectif : < 2 µs

# ==============================
# RDMA over Converged Ethernet (RoCE)
# RDMA sur Ethernet (alternative IB)
# ==============================
# RoCE v2 recommandé (routable, basé UDP/IP)
# Configuration Priority Flow Control (PFC) sur les switchs obligatoire

# ==============================
# Oracle RAC avec IB — configuration
# ==============================
# Oracle détecte automatiquement IB via oifcfg si interface configurée
oifcfg getif
oifcfg setif ib0/192.168.2.0:cluster_interconnect

# Vérifier que Oracle utilise bien IB pour Cache Fusion
-- SQL :
SELECT name, ip_address, is_public FROM v$cluster_interconnects;
-- interface IB doit apparaître ici

# Exadata utilise RDS (Reliable Datagram Sockets) sur IB — max performance
# Standard RAC sur IB utilise UDP
```

### Subnet Manager OpenSM

```bash
# Configuration OpenSM (/etc/rdma/opensm.conf ou /etc/opensm/opensm.conf)
# guid 0x0000000000000000   # auto-detect
# lmc 0                     # LID Mask Control
# sm_priority 0             # priorité SM (0-15, 15 = plus haute)

# Vérifier le SM actif
smpquery nodeinfo <switch_LID>
sminfo                      # infos SM

# Topologie fabric
ibtopology -o graph.dot
dot -Tpdf graph.dot -o ib_topology.pdf
```

---

# PARTIE 11 — STOCKAGE

## 11.1 Fibre Channel SAN

### Architecture FC pour Oracle RAC

```bash
# ==============================
# Composants FC
# ==============================
# HBA (Host Bus Adapter) — carte FC dans le serveur
# FC Switch (Fabric) — commutateur FC (Brocade, Cisco MDS)
# Storage Array — baie de stockage (NetApp, EMC, HPE, Pure)
# WWN (World Wide Name) — identifiant unique HBA/port (comme MAC pour FC)

# Vitesses FC :
# 8G FC, 16G FC, 32G FC ← standard actuel, 64G FC en montée

# ==============================
# Gestion HBA Linux
# ==============================
# Trouver les HBA et leurs WWN
ls /sys/class/fc_host/
cat /sys/class/fc_host/host0/port_name    # WWPN du port
cat /sys/class/fc_host/host0/node_name    # WWNN du nœud

# Lister les devices FC découverts
ls /sys/class/fc_remote_ports/

# Rescan HBA après ajout de LUN
echo "1" > /sys/class/fc_host/host0/issue_lip      # Loop Initialization Primitive
echo "- - -" > /sys/class/scsi_host/host0/scan     # rescan SCSI

# Ou via sg3_utils
rescan-scsi-bus.sh -a -r                            # rescan complet

# ==============================
# Multipath I/O — Device Mapper Multipath
# ==============================
# Configuration recommandée Oracle sur RHEL/OEL
# /etc/multipath.conf

defaults {
    user_friendly_names     no           # utiliser WWID pas mpathX
    find_multipaths         yes
    path_grouping_policy    multibus     # ou failover pour actif-passif
    path_checker            tur          # Test Unit Ready
    path_selector           "round-robin 0"
    failback                immediate
    no_path_retry           fail
    rr_min_io               100
    rr_weight               priorities
}

blacklist {
    devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
    devnode "^hd[a-z]"
    devnode "^cciss.*"
}

# Device spécifique (exemple NetApp ONTAP)
devices {
    device {
        vendor                  "NETAPP"
        product                 "LUN"
        path_grouping_policy    group_by_prio
        prio                    ontap
        path_checker            tur
        hardware_handler        "1 alua"
        failback                immediate
        rr_weight               uniform
        no_path_retry           queue
    }
}

# ==============================
# Commandes multipath
# ==============================
multipath -ll                           # afficher toutes les cartes multipath
multipath -v3 2>&1 | head -100          # debug découverte
multipathd show paths                   # statut chemins
multipathd show maps                    # statut cartes
multipathd reconfigure                  # recharger config
multipathd add path sdb                 # ajouter chemin manuellement

# Identifier un LUN Oracle ASM
ls -la /dev/mapper/                     # LUNs multipath
multipath -ll | grep -A5 <wwid>

# ==============================
# Vérification performance FC
# ==============================
iostat -x /dev/mapper/mpatha 2 10
# Objectif : await < 5ms, svctm < 2ms, %util < 80%

# Latence bas niveau
hdparm -t --direct /dev/mapper/mpatha   # throughput sequentiel
fio --filename=/dev/mapper/mpatha \
  --direct=1 --rw=randread --bs=8k \
  --numjobs=8 --runtime=60 --group_reporting \
  --name=oracle_sim                     # simulation Oracle workload
```

### Zones FC (Zoning)

```bash
# Le zoning FC se configure sur les switches FC (Brocade/Cisco)
# Best practice Oracle : Single Initiator / Single Target (SI/ST)
# Chaque HBA (initiateur) zoné avec un seul port storage (cible) par zone

# Exemple Brocade (via SSH au switch) :
# zoneadd "zone_node1_hba1_stor1", "21:00:00:24:ff:xx:xx:xx;50:0a:xx:xx:xx:xx:xx:xx"
# cfgadd "cfg_oracle", "zone_node1_hba1_stor1"
# cfgsave
# cfgenable "cfg_oracle"

# Cisco MDS (via NX-OS) :
# zone name zone_node1_hba1_stor1 vsan 10
#   member pwwn 21:00:00:24:ff:xx:xx:xx
#   member pwwn 50:0a:xx:xx:xx:xx:xx:xx
# zoneset name zoneset_oracle vsan 10
#   member zone_node1_hba1_stor1
# zoneset activate name zoneset_oracle vsan 10

# Vérifier les zones depuis Linux
systool -c fc_host -v                  # voir les fabric logins
cat /sys/class/fc_host/host0/fabric_name
```

## 11.2 iSCSI pour Oracle

### Configuration iSCSI Linux (Open-iSCSI)

```bash
# ==============================
# Installation et démarrage
# ==============================
yum install -y iscsi-initiator-utils
systemctl enable iscsid iscsi
systemctl start iscsid

# Configurer l'IQN de l'initiateur
cat /etc/iscsi/initiatorname.iscsi
# InitiatorName=iqn.1994-05.com.redhat:node1-oracle

# ==============================
# Découverte et connexion
# ==============================
# Découvrir les targets iSCSI sur la baie
iscsiadm -m discovery -t sendtargets -p <storage_ip>:3260

# Lister les targets découverts
iscsiadm -m node

# Se connecter à un target
iscsiadm -m node -T iqn.2000-01.com.netapp:... -p <ip>:3260 -l

# Login persistant (automatique au boot)
iscsiadm -m node -T iqn.2000-01.com.netapp:... \
  -p <ip>:3260 --op update \
  -n node.startup -v automatic

# ==============================
# Optimisations iSCSI pour Oracle
# ==============================
# /etc/iscsi/iscsid.conf — paramètres recommandés

# Timeouts
node.session.timeo.replacement_timeout = 120
node.conn[0].timeo.login_timeout = 15
node.conn[0].timeo.logout_timeout = 15
node.conn[0].timeo.noop_out_interval = 5
node.conn[0].timeo.noop_out_timeout = 5

# Queue depth (augmenter pour Oracle)
node.session.queue_depth = 128

# MaxRecvDataSegmentLength (MTU jumbo)
node.conn[0].iscsi.MaxRecvDataSegmentLength = 262144

# Jumbo frames sur les interfaces iSCSI (OBLIGATOIRE pour les perfs)
ip link set eth3 mtu 9000

# ==============================
# Multipath iSCSI
# ==============================
# Même configuration multipath.conf que FC
# Ajouter les sessions iSCSI redondantes :
iscsiadm -m node -T <iqn> -p <ip2>:3260 -l    # second chemin via IP différente

# Vérifier les sessions
iscsiadm -m session -P3

# ==============================
# Sécurité iSCSI (CHAP)
# ==============================
# /etc/iscsi/iscsid.conf :
# node.session.auth.authmethod = CHAP
# node.session.auth.username = iscsi_user
# node.session.auth.password = secret
# node.session.auth.username_in = storage_array
# node.session.auth.password_in = secret_in
```

## 11.3 NVMe-oF (NVMe over Fabrics)

NVMe-oF permet d'accéder à des SSDs NVMe distants avec une latence quasi-locale. Transports supportés : RDMA (RoCE/IB), TCP, FC-NVMe.

### Configuration NVMe-oF côté hôte

```bash
# ==============================
# Prérequis kernel
# ==============================
modprobe nvme-fabrics
modprobe nvme-rdma       # pour transport RDMA (RoCE/IB)
modprobe nvme-tcp        # pour transport TCP

# Vérifier les modules
lsmod | grep nvme

# ==============================
# Découverte NVMe-oF (RDMA/RoCE)
# ==============================
nvme discover -t rdma -a <storage_ip> -s 4420

# Connexion à un subsystème NVMe-oF
nvme connect -t rdma \
  -n nqn.2020-01.com.storage:nvme-subsystem1 \
  -a <storage_ip> \
  -s 4420 \
  --nr-io-queues 8

# ==============================
# Découverte NVMe-oF (TCP)
# ==============================
nvme discover -t tcp -a <storage_ip> -s 4420 -q <host_nqn>
nvme connect -t tcp \
  -n nqn.2020-01.com.storage:nvme-subsystem1 \
  -a <storage_ip> -s 4420

# ==============================
# Gestion des namespaces
# ==============================
nvme list                           # lister tous les devices NVMe
nvme list-subsys                    # lister subsystèmes et chemins
nvme id-ctrl /dev/nvme0             # identité contrôleur
nvme id-ns /dev/nvme0n1             # identité namespace

# Statut multipath NVMe (Native NVMe Multipath)
nvme list-subsys | grep -E "nvme|NQN|│"
cat /sys/class/nvme-subsystem/nvme-subsys0/iopolicy
# round-robin (défaut) ou numa ou queue-depth

# Changer la politique multipath NVMe
echo "round-robin" > /sys/class/nvme-subsystem/nvme-subsys0/iopolicy

# ==============================
# Performance NVMe-oF
# ==============================
nvme smart-log /dev/nvme0           # SMART et température
nvme get-feature /dev/nvme0 --feature-id=7  # queue depth

# Benchmark NVMe-oF (objectif < 100µs latence)
fio --filename=/dev/nvme0n1 \
  --direct=1 --rw=randread --bs=4k \
  --numjobs=16 --iodepth=32 \
  --runtime=60 --group_reporting \
  --name=nvmeof_test
# Objectif : > 500K IOPS à 4K, latence < 100µs

# ==============================
# NVMe-oF avec ASM Oracle
# ==============================
# Les devices NVMe-oF apparaissent comme /dev/nvmeXnY
# Pour Oracle ASM, utiliser udev rules pour créer des liens stables :
# /etc/udev/rules.d/99-oracle-nvme.rules :
# KERNEL=="nvme[0-9]*n[0-9]*", SUBSYSTEM=="block",
#   ATTRS{wwid}=="<wwid>",
#   SYMLINK+="oracleasm/nvme_disk01",
#   OWNER="grid", GROUP="asmadmin", MODE="0660"

udevadm control --reload-rules
udevadm trigger
```

### Comparaison des technologies de stockage Oracle

| Technologie | Latence typique | Bande passante | Usage Oracle |
|-------------|----------------|----------------|--------------|
| FC SAN 32G | 0.5–2 ms | ~6 GB/s | Standard HA/DR |
| iSCSI 25GbE | 0.5–3 ms | ~3 GB/s | Entrée de gamme HA |
| NVMe-oF RDMA | < 0.1 ms | > 20 GB/s | Exadata-like, OLTP critique |
| NVMe-oF TCP | 0.1–0.5 ms | > 10 GB/s | NVMe-oF économique |
| InfiniBand direct | < 0.05 ms | > 50 GB/s | Exadata interne |
| Local NVMe | < 0.05 ms | > 7 GB/s | Standby local, temp |

## 11.4 Oracle ASM et le stockage bas niveau

### udev Rules pour ASM (Linux)

```bash
# ==============================
# Méthode recommandée : udev rules (sans oracleasm kernel module)
# ==============================
# /etc/udev/rules.d/99-oracle-asm.rules

# Par WWID (multipath) — méthode la plus stable
KERNEL=="dm-[0-9]*", ENV{DM_WWN}=="0x6000097...", \
  SYMLINK+="oracleasm/disks/DATA01", \
  OWNER="grid", GROUP="asmadmin", MODE="0660"

# Par ID de partition
KERNEL=="sd[a-z][0-9]", SUBSYSTEM=="block", \
  ENV{ID_SERIAL}=="3600507680282862df000000000000f1", \
  SYMLINK+="oracleasm/disks/FRA01", \
  OWNER="grid", GROUP="asmadmin", MODE="0660"

# Recharger udev
udevadm control --reload-rules
udevadm trigger --type=devices --subsystem-match=block

# Vérifier
ls -la /dev/oracleasm/disks/
# Vérifier ASM voit les disques
export ORACLE_SID=+ASM1
asmcmd lsdsk --candidate        # disques candidats non utilisés
asmcmd lsdsk                    # disques utilisés

# ==============================
# OracleASM kernel module (méthode legacy)
# ==============================
yum install -y oracleasm-support kmod-oracleasm

# Configurer
oracleasm configure -i
# Répondre :
# Default user: grid
# Default group: asmadmin
# Scan at boot: y
# Enable ASM: y

# Créer disques ASM
oracleasm createdisk DATA01 /dev/sdb1
oracleasm createdisk DATA02 /dev/sdc1
oracleasm listdisks
oracleasm scandisks

# ==============================
# Diagnostic stockage Oracle
# ==============================
-- Vérifier I/O Oracle par fichier
SELECT f.name, s.phyrds, s.phywrts,
       s.readtim/NULLIF(s.phyrds,0) avg_read_ms,
       s.writetim/NULLIF(s.phywrts,0) avg_write_ms
FROM v$filestat s, v$datafile f
WHERE s.file# = f.file#
ORDER BY (s.readtim + s.writetim) DESC
FETCH FIRST 20 ROWS ONLY;

-- I/O par tablespace
SELECT t.name, s.phyrds + s.phywrts total_io,
       ROUND(s.readtim/NULLIF(s.phyrds,0)/100,2) avg_read_ms,
       ROUND(s.writetim/NULLIF(s.phywrts,0)/100,2) avg_write_ms
FROM v$tablespace t, v$filestat s, v$datafile d
WHERE t.ts# = d.ts# AND d.file# = s.file#
GROUP BY t.name, s.phyrds, s.phywrts, s.readtim, s.writetim
ORDER BY total_io DESC;

-- Segments chauds (hot segments)
SELECT owner, segment_name, segment_type,
       physical_reads, db_block_changes
FROM v$segment_statistics
WHERE statistic_name IN ('physical reads', 'db block changes')
AND value > 10000
ORDER BY value DESC
FETCH FIRST 20 ROWS ONLY;
```

## 11.5 Recommandations Layout Stockage Oracle HA

### Disk Groups ASM recommandés

```
+DATA   → Datafiles, Tempfiles, Control files (copie 1)
          Redondance : NORMAL (2 failure groups min) ou HIGH (3)
          
+REDO   → Online Redo Logs, Standby Redo Logs, Control files (copie 2)
          Redondance : HIGH recommandé (I/O critique)
          
+FRA    → Fast Recovery Area : archivelogs, backups RMAN, flashback logs
          Redondance : NORMAL minimum
          
+ACFS   → (optionnel) Binaires Oracle, scripts, si ACFS requis
          Redondance : NORMAL
```

### Calcul capacité ASM

```sql
-- Espace réel disponible (tenant compte de la redondance)
-- NORMAL : usable = free_mb / 2
-- HIGH   : usable = free_mb / 3
SELECT name, type,
       total_mb / 1024 total_gb,
       free_mb / 1024 free_gb,
       usable_file_mb / 1024 usable_gb,
       ROUND(100 - (usable_file_mb / (total_mb/
         CASE type WHEN 'NORMAL' THEN 2
                   WHEN 'HIGH' THEN 3
                   ELSE 1 END) * 100), 1) pct_used
FROM v$asm_diskgroup
ORDER BY name;
```

---

# SYSTEM PROMPT FINAL — À COLLER DANS LES INSTRUCTIONS DU PROJET CLAUDE

```
Tu es un expert Oracle Database 19c spécialisé en Haute Disponibilité et Disaster Recovery,
ainsi qu'en infrastructure système, réseau et stockage pour environnements Oracle critiques.

## RÈGLE ABSOLUE — RÉFÉRENCES OBLIGATOIRES

Chaque réponse doit obligatoirement se terminer par un bloc "📚 Références" structuré ainsi :

📚 Références
┌─────────────────────────────────────────────────────────────┐
│ [1] <Nom exact du PDF>  — <Chapitre/Section>  — p.<numéro> │
│ [2] <Nom exact du PDF>  — <Chapitre/Section>  — p.<numéro> │
│ [KB] Base de connaissance Oracle19c_HA — Partie X.Y         │
└─────────────────────────────────────────────────────────────┘

Règles impératives pour les références :
- TOUTE affirmation technique doit être numérotée [1], [2]... et référencée dans ce bloc.
- Si la réponse s'appuie sur un PDF du projet : nom du PDF + section ou chapitre + page.
- Si la réponse s'appuie sur la base de connaissance (fichier .md du projet) : indiquer [KB] + la partie concernée.
- Si AUCUN document du projet ne couvre le sujet : indiquer EXPLICITEMENT en début de réponse :
  "⚠️ Aucun document du projet ne couvre ce point. Réponse basée sur connaissance générale Oracle 19c."
  puis fournir quand même la réponse, avec [KB] si applicable.
- Ne jamais omettre le bloc Références, même pour une réponse courte.
- Les PDFs du projet ont priorité absolue sur tout autre source. Si un PDF contredit
  la base de connaissance, le PDF fait foi — le signaler explicitement.

Exemple de réponse correctement référencée :

  "Le mode MAXIMUM AVAILABILITY permet une perte de données quasi nulle [1].
  Le redo transport doit être configuré en SYNC avec AFFIRM [2].
  Les Standby Redo Logs sont obligatoires dans ce mode [1][KB]."

  📚 Références
  ┌──────────────────────────────────────────────────────────────────────┐
  │ [1]  Oracle Data Guard Concepts and Administration 19c — Chap. 6    │
  │      "Redo Transport Services" — p. 6-4                             │
  │ [2]  Oracle Data Guard Broker 19c — Chap. 4                         │
  │      "Configuring Protection Modes" — p. 4-12                       │
  │ [KB] Base de connaissance Oracle19c_HA — Partie 3.2 (paramètres DG) │
  └──────────────────────────────────────────────────────────────────────┘

## DOMAINES D'EXPERTISE

### Base de données Oracle 19c
- Oracle Grid Infrastructure 19c (Clusterware, ASM, ACFS)
- Oracle Real Application Clusters (RAC) 19c
- Oracle Data Guard 19c (Physical/Logical Standby, Active Data Guard, Broker)
- Architectures combinées RAC + Data Guard (Far Sync, cascaded standby)
- Oracle GoldenGate 19c (réplication logique, topologies, Microservices)

### Système
- Linux : RHEL / Oracle Linux 7/8/9 (tuning kernel, cgroups, NUMA, Huge Pages)
- Windows Server (services Oracle, WSFC, PowerShell)
- Prérequis OS, limits, optimisations pour Oracle HA

### Réseau
- Ethernet 10/25/40/100 GbE : bonding LACP, teaming, QoS, MTU jumbo
- InfiniBand : OFED, OpenSM, RDMA, RoCE
- VLAN : segmentation du trafic Oracle (public/interconnect/stockage/backup/DR)
- Architecture réseau inter-sites pour Data Guard et GoldenGate

### Stockage
- FC SAN : HBA, zoning, multipath (DM-Multipath), tuning
- iSCSI : Open-iSCSI, CHAP, multipath, optimisations
- NVMe-oF : RDMA, TCP, FC-NVMe, native multipath
- Oracle ASM : disk groups, udev rules, layout recommandé
- Benchmarking et diagnostic I/O

## COMPORTEMENT

1. Répondre avec la précision d'un DBA/architecte senior Oracle + infrastructure.
2. Fournir les commandes exactes avec leurs options et leur contexte d'exécution.
3. Indiquer les paramètres critiques avec leurs valeurs recommandées.
4. Signaler les pièges (gotchas), bugs Oracle connus et patches recommandés (avec MOS Note).
5. Structurer : Concept → Architecture → Commandes → Vérification → Troubleshooting.
6. Utiliser les versions exactes (19.3.0.0, RHEL 8.6, etc.) quand pertinent.
7. Pour tout switchover/failover/maintenance : fournir checklist pré/post opération.
8. En cas d'ambiguïté sur la topologie, poser des questions ciblées avant de répondre.
9. Ne jamais simplifier sur les sujets critiques : protection des données, quorum, I/O.
10. TOUJOURS terminer par le bloc 📚 Références — sans exception.
```
