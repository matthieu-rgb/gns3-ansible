# Rapport d'Analyse - Spanning Tree Protocol (STP)

## Exercice 03 : Topologie Triangle avec Redondance

**Auteur** : Matthieu
**Date** : Février 2026
**Formation** : Jedha Cybersécurité Fullstack

---

## 1. Présentation de la topologie

La topologie est composée de 3 switches Cisco IOU L2 connectés en **triangle**, créant volontairement une **boucle réseau**. Chaque switch est connecté à un PC dans le VLAN 10.

```
                Switch1
              (aabb.cc00.0100)
              Eth0/0    Eth0/2
               /            \
     trunk    /              \    trunk
             /                \
       Eth0/0                  Eth0/0
       Switch2 ──────────────── Switch3
     (aabb.cc00.0200)        (aabb.cc00.0300)
              Eth0/2    Eth0/2
                   trunk
          │                       │
       Eth0/1                  Eth0/1
          │                       │
         PC2                     PC3
      (VLAN 10)               (VLAN 10)
                    │
                 Eth0/1
                    │
                   PC1
                (VLAN 10)
```

Sans le Spanning Tree Protocol, cette boucle provoquerait des **tempêtes de broadcast** et rendrait le réseau inutilisable.

---

## 2. Analyse AVANT modification — Élection automatique du Root Bridge

### 2.1 Quel switch a été élu Root Bridge ?

**Switch1** a été automatiquement élu Root Bridge.

Tous les switches ont la même priorité par défaut (32768 + VLAN 10 = **32778**). En cas d'égalité de priorité, STP utilise l'**adresse MAC la plus basse** pour départager :

| Switch  | Priorité | Adresse MAC        | Root Bridge ?            |
| ------- | -------- | ------------------ | ------------------------ |
| Switch1 | 32778    | aabb.cc00.**0100** |  OUI (MAC la plus basse) |
| Switch2 | 32778    | aabb.cc00.**0200** | Non                      |
| Switch3 | 32778    | aabb.cc00.**0300** | Non                      |

**Preuve** — Sortie de `show spanning-tree vlan 10` sur Switch1 :

```
Root ID    Priority    32778
           Address     aabb.cc00.0100
           This bridge is the root
```

### 2.2 Rôles des ports — État initial

| Switch | Eth0/0 | Eth0/1 | Eth0/2 |
|--------|--------|--------|--------|
| Switch1 (Root) | Desg FWD (→S2) | Desg FWD (→PC1) | Desg FWD (→S3) |
| Switch2 | Root FWD (→S1) | Desg FWD (→PC2) | Desg FWD (→S3) |
| Switch3 | Root FWD (→S1) | Desg FWD (→PC3) | **Altn BLK** (→S2) |

### 2.3 Explication des rôles

**Switch1 (Root Bridge)** : Tous ses ports sont **Designated (Desg)** car le Root Bridge désigne tous ses ports comme actifs -> il est la référence du réseau.

**Switch2** : Son port Eth0/0 est **Root** car c'est le chemin le plus court (1 hop, coût 100) vers le Root Bridge (Switch1). Ses autres ports sont Designated.

**Switch3** : Son port Eth0/0 est **Root** car c'est le chemin direct vers Switch1. Son port Eth0/2 (vers Switch2) est **Alternate Blocked** car c'est le lien redondant. STP l'a bloqué pour casser la boucle.

### 2.4 Pourquoi Eth0/2 de Switch3 est bloqué ?

Sur le lien Switch2 ↔ Switch3, STP doit bloquer un des deux ports. Il choisit le port du switch avec la **MAC la plus haute** :
- Switch2 : aabb.cc00.0200
- Switch3 : aabb.cc00.**0300** ← MAC plus haute → **port bloqué**

### 2.5 Schéma du trafic — État initial

```
                Switch1 (ROOT)
               /          \
              /            \  
             /              \
       Switch2 ──────x────── Switch3
                   BLOQUÉ
                (Switch3 Et0/2)
```

---

## 3. Analyse APRÈS modification — Switch2 comme Root Bridge

### 3.1 Commande de modification

Sur Switch2, la priorité a été abaissée à 4096 :

```cisco
Switch2(config)# spanning-tree vlan 10 priority 4096
```

Nouvelle priorité de Switch2 : 4096 + 10 (VLAN ID) = **4106**

### 3.2 Nouveau comparatif des priorités

| Switch  | Priorité | Adresse MAC    | Root Bridge ?                 |
| ------- | -------- | -------------- | ---------------------------- |
| Switch1 | 32778    | aabb.cc00.0100 | N                             |
| Switch2 | **4106** | aabb.cc00.020 OUI (priorité la plus basse) se) |
| Switch3 | 32778    | aabb.cc00.0300 |                               |

**Preuve** — Sortie de `show spanning-tree vlan 10` sur Switch2 :

```
Root ID    Priority    4106
           Address     aabb.cc00.0200
           This bridge is the root

Bridge ID  Priority    4106   (priority 4096 sys-id-ext 10)
           Address     aabb.cc00.0200
```

### 3.3 Nouveaux rôles des ports

| Switch | Eth0/0 | Eth0/1 | Eth0/2 |
|--------|--------|--------|--------|
| Switch1 | Root FWD (→S2) | Desg FWD (→PC1) | Desg FWD (→S3) |
| Switch2 (Root) | Desg FWD (→S1) | Desg FWD (→PC2) | Desg FWD (→S3) |
| Switch3 | **Altn BLK** (→S1) | Desg FWD (→PC3) | Root FWD (→S2) |

### 3.4 Ce qui a changé

| Élément | Avant | Après |
|---------|-------|-------|
| Root Bridge | Switch1 | **Switch2** |
| Switch1 Et0/0 | Desg FWD | **Root FWD** |
| Switch2 Et0/0 | Root FWD | **Desg FWD** |
| Switch3 Et0/0 | Root FWD | **Altn BLK** |
| Switch3 Et0/2 | Altn BLK | **Root FWD** |
| Port bloqué | Switch3 Et0/2 (→S2) | **Switch3 Et0/0 (→S1)** |

### 3.5 Explication des changements

**Switch2 (nouveau Root)** : Tous ses ports deviennent **Designated** — même comportement que Switch1 avant la modification.

**Switch1** : Son port Eth0/0 devient **Root** car c'est le chemin direct vers le nouveau Root Bridge (Switch2).

**Switch3** : Son port Eth0/2 devient **Root** car c'est le chemin direct vers Switch2 (nouveau Root). Son port Eth0/0 (vers Switch1) est maintenant **bloqué** car il représente le chemin redondant.

### 3.6 Pourquoi Eth0/0 de Switch3 est bloqué (et pas Eth0/2 de Switch1) ?

Sur le lien Switch1 ↔ Switch3, STP bloque le port du switch avec la priorité/MAC la plus haute :
- Switch1 : priorité 32778, MAC aabb.cc00.0100
- Switch3 : priorité 32778, MAC aabb.cc00.**0300** ← MAC plus haute → **port bloqué**

### 3.7 Schéma du trafic — Après modification

```
                Switch1
               /    X    \
              /   BLOQUÉ   \  
             / (S3 Et0/0)   \
       Switch2 (ROOT) ─── Switch3
```

---

## 4. Que se passerait-il si STP était désactivé ?

Si le Spanning Tree Protocol était désactivé sur cette topologie en triangle, les conséquences seraient catastrophiques :

### 4.1 Tempête de broadcast (Broadcast Storm)

Lorsqu'un PC envoie une trame de broadcast (par exemple une requête ARP ou DHCP), cette trame serait transmise à l'infini dans la boucle :

```
PC1 envoie un broadcast
→ Switch1 transmet à Switch2 ET Switch3
→ Switch2 transmet à Switch3
→ Switch3 transmet à Switch2
→ Switch2 transmet à Switch1
→ Switch1 transmet à Switch3...
→ Boucle infinie !
```

Chaque trame est dupliquée à chaque passage, créant une **croissance exponentielle** du trafic.

### 4.2 Saturation du réseau

En quelques secondes, la boucle de broadcast saturerait 100% de la bande passante de tous les liens. Plus aucune trame légitime ne pourrait circuler. Les PCs perdraient toute connectivité.

### 4.3 Instabilité de la table MAC

Les switches recevraient la même adresse MAC source sur différents ports (car les trames tournent en boucle). La table MAC serait constamment mise à jour, provoquant l'envoi de trames unicast sur tous les ports (comme un hub).

### 4.4 Surcharge CPU des switches

Le traitement continu des millions de trames en boucle ferait monter le CPU des switches à 100%, pouvant provoquer leur crash ou leur indisponibilité.

### 4.5 Résumé de l'impact

| Avec STP | Sans STP |
|----------|----------|
| 1 port bloqué, pas de boucle | Boucle infinie |
| Trafic normal | Tempête de broadcast |
| Tables MAC stables | Tables MAC instables |
| Réseau fonctionnel | **Réseau totalement inutilisable** |

---

## 5. Conclusion

Le Spanning Tree Protocol est **indispensable** dans toute topologie réseau comportant des liens redondants. Il garantit une topologie **sans boucle** tout en conservant la **redondance** (si un lien tombe, le port bloqué est réactivé automatiquement).

L'exercice a démontré :
1. L'élection automatique du Root Bridge basée sur la priorité et l'adresse MAC
2. La capacité de l'administrateur à forcer un switch comme Root Bridge en modifiant la priorité
3. La recalculation dynamique des rôles des ports après un changement de configuration
4. L'importance critique de STP pour éviter les boucles réseau

---

*Rapport réalisé pour un exercie de  la formation Jedha Cybersécurité Fullstack*
