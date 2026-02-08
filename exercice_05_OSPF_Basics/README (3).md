# Exercice 05 - OSPF Basics

#networking #ospf #ansible #gns3 #jedha

---

##  Description

Configuration OSPF entre 3 routeurs Cisco c2691 formant une topologie triangle redondante.

**Scénario** : TechBridge Solutions - 3 agences (Paris, Berlin, Rome) interconnectées par des liaisons WAN dédiées.

---

##  Objectifs pédagogiques

- [x] Comprendre le fonctionnement d'OSPF (protocole à état de liens)
- [x] Établir des relations de voisinage OSPF
- [x] Configurer les adresses IP sur chaque interface
- [x] Annoncer les réseaux dans OSPF Area 0
- [x] Vérifier la propagation dynamique des routes
- [x] Observer le load balancing ECMP
- [x] Tester la redondance (failover et reconvergence)

---

##  Topologie

```
            R1 (Paris)
           /          \
       Fa0/0          Fa0/1
    .1    |            |    .1
          |            |
   192.168.12.0   192.168.31.0
          |            |
    .2    |            |    .2
       Fa0/0          Fa0/1
           \          /
     R2 (Berlin)----R3 (Rome)
            Fa0/1  Fa0/0
              .1    .2
           192.168.23.0
```

---

##  Plan d'adressage

| Lien | Sous-réseau | R1 | R2 | R3 |
|------|-------------|----|----|-----|
| R1↔R2 | 192.168.12.0/24 | .1 (Fa0/0) | .2 (Fa0/0) | - |
| R2↔R3 | 192.168.23.0/24 | - | .1 (Fa0/1) | .2 (Fa0/0) |
| R3↔R1 | 192.168.31.0/24 | .1 (Fa0/1) | - | .2 (Fa0/1) |

---

##  Utilisation

### Prérequis

- GNS3 Server accessible (http://192.168.138.198:80)
- Template `c2691` disponible
- Ansible installé
- Package `expect` installé

### Installation

```bash
# Cloner le repo
git clone <repo_url>
cd exercice_05_OSPF_Basics

# Vérifier la connexion GNS3
curl -s "http://192.168.138.198:80/v2/version"
```

### Exécution

```bash
# 1. Créer la topologie (3 routeurs + liens)
ansible-playbook playbooks/01_create_topology.yml

# 2. Configurer OSPF sur tous les routeurs
ansible-playbook playbooks/02_configure_ospf.yml
```

### Vérification manuelle

```bash
# Récupérer les ports console
curl -s "http://192.168.138.198:80/v2/projects/<project_id>/nodes" | jq '.[] | {name, console}'

# Se connecter à R1
telnet 192.168.138.198 <port_console_R1>
```

```cisco
! Vérifier les voisins OSPF
show ip ospf neighbor

! Vérifier les routes apprises
show ip route ospf

! Vérifier les interfaces OSPF
show ip ospf interface brief

! Vérifier la base de données OSPF
show ip ospf database
```

---

##  Structure du projet

```
exercice_05_OSPF_Basics/
├── README.md                    # Ce fichier
├── SCHEMA_OSPF.md              # Schémas Mermaid de la topologie
├── RESUME_EXERCICE.md          # Guide pour refaire l'exercice
├── ansible.cfg                  # Configuration Ansible
├── inventory.yml                # Inventaire (localhost)
├── vars/
│   └── network.yml              # Variables (IPs, liens, routeurs)
└── playbooks/
    ├── 01_create_topology.yml   # Création topologie GNS3
    └── 02_configure_ospf.yml    # Configuration OSPF
```

---

##  Résultats attendus

### 1. Voisins OSPF
```cisco
R1#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
192.168.31.2      1   FULL/DR         00:00:36    192.168.31.2    FastEthernet0/1
192.168.23.1      1   FULL/BDR        00:00:32    192.168.12.2    FastEthernet0/0
```

### 2. Routes OSPF (Load Balancing)
```cisco
R1#show ip route ospf
O    192.168.23.0/24 [110/20] via 192.168.31.2, FastEthernet0/1
                     [110/20] via 192.168.12.2, FastEthernet0/0
```

### 3. Redondance
Après `shutdown` de Fa0/0 sur R1 :
```cisco
R1#show ip route ospf
O    192.168.23.0/24 [110/20] via 192.168.31.2, FastEthernet0/1
```
→ Le trafic passe par l'autre chemin automatiquement !

---

##  Commandes utiles

| Commande | Description |
|----------|-------------|
| `show ip ospf neighbor` | Liste des voisins OSPF |
| `show ip route ospf` | Routes apprises via OSPF |
| `show ip ospf interface` | État des interfaces OSPF |
| `show ip ospf database` | Base de données LSDB |
| `show ip protocols` | Protocoles de routage actifs |
| `debug ip ospf adj` | Debug des adjacences (attention !) |

---

##  Concepts clés

| Concept | Explication |
|---------|-------------|
| **Area 0** | Zone backbone obligatoire |
| **Wildcard mask** | Inverse du masque (255.255.255.0 → 0.0.0.255) |
| **Process ID** | Local au routeur (peut différer entre routeurs) |
| **Router ID** | Identifiant unique (plus haute IP loopback ou interface) |
| **Coût OSPF** | 100 Mbps / Bande passante (par défaut) |
| **ECMP** | Equal-Cost Multi-Path (load balancing) |
| **LSA Type 1** | Router LSA (généré par chaque routeur) |

---

##  Tests de validation

### Test 1 : Voisinage
```cisco
show ip ospf neighbor
```
 Tous les voisins en état FULL

### Test 2 : Routes
```cisco
show ip route ospf
```
 Routes vers tous les sous-réseaux distants

### Test 3 : Connectivité
```cisco
ping 192.168.23.2
```
 Ping réussi vers R3 depuis R1

### Test 4 : Redondance
```cisco
interface Fa0/0
shutdown
!
show ip route ospf
```
 Route alternative utilisée

---

##  Ressources


- [RFC 2328 - OSPFv2](https://tools.ietf.org/html/rfc2328)
- [Cisco OSPF Configuration Guide](https://www.cisco.com/c/en/us/support/docs/ip/open-shortest-path-first-ospf/)

---

##  Auteur

**Matthieu** - Formation Cybersécurité Jedha

---

*Dernière mise à jour : Février 2026*
