# Schéma Topologie OSPF

#networking #ospf #gns3 #jedha

---

## Topologie Triangle

```mermaid
graph TB
    subgraph "OSPF Area 0"
        R1["🏢 R1 - Paris<br/>Router ID: 192.168.31.1"]
        R2["🏢 R2 - Berlin<br/>Router ID: 192.168.23.1"]
        R3["🏢 R3 - Rome<br/>Router ID: 192.168.31.2"]
    end

    R1 <-->|"192.168.12.0/24<br/>Fa0/0 ↔ Fa0/0<br/>.1 ↔ .2"| R2
    R2 <-->|"192.168.23.0/24<br/>Fa0/1 ↔ Fa0/0<br/>.1 ↔ .2"| R3
    R3 <-->|"192.168.31.0/24<br/>Fa0/1 ↔ Fa0/1<br/>.2 ↔ .1"| R1

    style R1 fill:#e1f5fe
    style R2 fill:#fff3e0
    style R3 fill:#f3e5f5
```

---

## Schéma ASCII (alternative)

```
                    ┌─────────────────────┐
                    │   R1 - PARIS        │
                    │   Fa0/0: 192.168.12.1│
                    │   Fa0/1: 192.168.31.1│
                    └─────────┬───────────┘
                             │
              Fa0/0          │          Fa0/1
        192.168.12.0/24      │     192.168.31.0/24
                             │
           ┌─────────────────┴─────────────────┐
           │                                   │
           ▼                                   ▼
┌──────────────────────┐           ┌──────────────────────┐
│   R2 - BERLIN        │           │   R3 - ROME          │
│   Fa0/0: 192.168.12.2│           │   Fa0/0: 192.168.23.2│
│   Fa0/1: 192.168.23.1│           │   Fa0/1: 192.168.31.2│
└──────────┬───────────┘           └───────────┬──────────┘
           │                                   │
           │         192.168.23.0/24           │
           │           Fa0/1 ↔ Fa0/0           │
           └───────────────────────────────────┘
```

---

## Tableau d'adressage IP

| Routeur | Site | Interface | Adresse IP | Sous-réseau | Voisin |
|---------|------|-----------|------------|-------------|--------|
| R1 | Paris | Fa0/0 | 192.168.12.1/24 | 192.168.12.0/24 | R2 |
| R1 | Paris | Fa0/1 | 192.168.31.1/24 | 192.168.31.0/24 | R3 |
| R2 | Berlin | Fa0/0 | 192.168.12.2/24 | 192.168.12.0/24 | R1 |
| R2 | Berlin | Fa0/1 | 192.168.23.1/24 | 192.168.23.0/24 | R3 |
| R3 | Rome | Fa0/0 | 192.168.23.2/24 | 192.168.23.0/24 | R2 |
| R3 | Rome | Fa0/1 | 192.168.31.2/24 | 192.168.31.0/24 | R1 |

---

## Sous-réseaux

| Sous-réseau | Lien | Description |
|-------------|------|-------------|
| 192.168.12.0/24 | R1 ↔ R2 | Paris - Berlin |
| 192.168.23.0/24 | R2 ↔ R3 | Berlin - Rome |
| 192.168.31.0/24 | R3 ↔ R1 | Rome - Paris |

---

## Chemins OSPF (Load Balancing ECMP)

```mermaid
graph LR
    subgraph "Depuis R1 : Route vers 192.168.23.0/24"
        R1["R1<br/>Paris"] 
        R1 -->|"Chemin 1<br/>via 192.168.12.2<br/>Cost: 20"| R2["R2<br/>Berlin"]
        R1 -->|"Chemin 2<br/>via 192.168.31.2<br/>Cost: 20"| R3["R3<br/>Rome"]
    end
```

### Tableau des chemins depuis R1

| Destination | Chemin | Via (Next-Hop) | Interface | Coût |
|-------------|--------|----------------|-----------|------|
| 192.168.23.0/24 | 1 | 192.168.12.2 (R2) | Fa0/0 | 20 |
| 192.168.23.0/24 | 2 | 192.168.31.2 (R3) | Fa0/1 | 20 |

→ **ECMP** (Equal-Cost Multi-Path) : Les 2 chemins ont le même coût, donc les deux sont utilisés !

---

## Redondance - Scénario de panne

```mermaid
graph TB
    subgraph "Lien R1-R2 DOWN"
        R1_down["R1<br/>Paris"] 
        R2_down["R2<br/>Berlin"]
        R3_down["R3<br/>Rome"]
        
        R1_down -.->|"❌ Fa0/0 DOWN"| R2_down
        R1_down -->|"✅ Fa0/1<br/>Seul chemin actif"| R3_down
        R3_down -->|"✅ Fa0/0"| R2_down
    end
```

### Avant la panne (2 chemins)
```
O  192.168.23.0/24 [110/20] via 192.168.12.2, Fa0/0
                   [110/20] via 192.168.31.2, Fa0/1
```

### Après la panne (1 chemin)
```
O  192.168.23.0/24 [110/20] via 192.168.31.2, Fa0/1
```

→ OSPF reconverge automatiquement !

---

## Relations de voisinage OSPF

```mermaid
graph LR
    R1((R1)) <-->|"FULL"| R2((R2))
    R2 <-->|"FULL"| R3((R3))
    R3 <-->|"FULL"| R1
```

| Routeur | Voisin 1 | Voisin 2 |
|---------|----------|----------|
| R1 | R2 (via Fa0/0) | R3 (via Fa0/1) |
| R2 | R1 (via Fa0/0) | R3 (via Fa0/1) |
| R3 | R2 (via Fa0/0) | R1 (via Fa0/1) |

---

*Dernière mise à jour : Février 2026*
