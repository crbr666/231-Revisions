# Résumé
## À quoi ça sert ?

Le RAID (**Redundant Array of Independent Disks**) permet de combiner plusieurs disques physiques en un seul volume logique, avec deux objectifs principaux :

- **Tolérance aux pannes** : survivre à la défaillance d'un ou plusieurs disques
- **Performance** : augmenter les débits en lecture/écriture par parallélisation

> ⚠️ Le RAID **ne remplace pas la sauvegarde**. Il protège contre la panne matérielle, mais pas contre la suppression accidentelle, la corruption, le ransomware ou un sinistre physique. Règle **3-2-1** : 3 copies, sur 2 supports différents, dont 1 hors site.

---

## Les RAID de base

|Niveau|Nom|Disques min.|Tolérance|Performance lecture|Performance écriture|Capacité utile|
|---|---|---|---|---|---|---|
|RAID 0|Striping|2|❌ aucune|⬆⬆ (parallèle)|⬆⬆ (parallèle)|100%|
|RAID 1|Mirroring|2|✅ 1 disque|⬆ (multi-source)|➡ (double écriture)|50%|
|RAID 5|Striping + parité|3|✅ 1 disque|⬆ (parallèle)|⬇ (calcul parité)|( n−1 ) / n|
|RAID 6|Striping + double parité|4|✅ 2 disques|⬆ (parallèle)|⬇⬇ (2× parité)|( n−2 ) / n|

---

### RAID 0 — Striping

Les données sont découpées en blocs et répartis sur tous les disques en parallèle. Aucune redondance : si un disque tombe, tout est perdu. Utilisé pour la performance pure.

```mermaid
graph LR
    D[Données A1 A2 A3 A4] --> D1[D1 A1 · A3]
    D --> D2[D2 A2 · A4]
```

---

### RAID 1 — Mirroring

Chaque donnée est écrite simultanément sur deux disques. Survit à la perte d'un disque. Capacité réduite de moitié.

```mermaid
graph LR
    D[Données A] --> D1[D1 A]
    D --> D2[D2 A - copie exacte]
```

---

### RAID 5 — Striping + parité XOR tournante

#### Bloc et stripe

- **Un bloc** = la plus petite unité de données écrite sur un disque (ex. 64 Ko)
- **Un stripe** = une rangée de blocs répartis simultanément sur tous les disques

```
           D1      D2      D3
Stripe 1 [ A1  ][ A2  ][ P   ]   ← une rangée = 1 stripe
Stripe 2 [ B1  ][ P   ][ B2  ]
Stripe 3 [ P   ][ C1  ][ C2  ]
```

Chaque disque physique contient donc une pile verticale de blocs, mélange de données et de parité :

```
D1 (disque physique)
├── bloc A1   (données, stripe 1)
├── bloc B1   (données, stripe 2)
└── bloc P    (parité, stripe 3)
```

#### Rotation de la parité

La parité ne tombe pas toujours sur le même disque — elle tourne à chaque stripe :

```
Stripe 1 → P est sur D3
Stripe 2 → P est sur D2
Stripe 3 → P est sur D1
```

Cela distribue la charge d'écriture équitablement entre tous les disques.

> C'est ce qui distingue le RAID 5 du **RAID 4**, où P est toujours sur le même disque — créant un goulot d'étranglement à chaque écriture :

```
RAID 4 :
Stripe 1 → P est sur D3   ← toujours D3
Stripe 2 → P est sur D3   ← toujours D3
Stripe 3 → P est sur D3   ← D3 surchargé
```

#### Vue par stripe (RAID 5)

```mermaid
graph TD
    S1[Stripe 1] --> A1[D1: A1]
    S1 --> A2[D2: A2]
    S1 --> A3[D3: A1⊕A2]
    S2[Stripe 2] --> B1[D1: B1]
    S2 --> B2[D2: B1⊕B2]
    S2 --> B3[D3: B2]
    S3[Stripe 3] --> C1[D1: C1⊕C2]
    S3 --> C2[D2: C1]
    S3 --> C3[D3: C2]
    style A3 fill:#ffaa00,color:#000
    style B2 fill:#ffaa00,color:#000
    style C1 fill:#ffaa00,color:#000
```

🟠 Blocs oranges = blocs de parité — physiquement stockés sur les disques, répartis par rotation.

---

### RAID 6 — Striping + double parité tournante

Comme le RAID 5 mais avec **deux blocs de parité indépendants (P et Q)** qui tournent également entre les disques. Survit à la perte de 2 disques simultanément.

- **P** = XOR simple → reconstruit 1 disque perdu
- **Q** = Reed-Solomon → reconstruit même si P est aussi perdu

```
           D1       D2       D3       D4
Stripe 1 [ A1   ][ A2   ][ P    ][ Q    ]
Stripe 2 [ B1   ][ P    ][ Q    ][ B2   ]
Stripe 3 [ P    ][ Q    ][ C1   ][ C2   ]
Stripe 4 [ Q    ][ D1   ][ D2   ][ P    ]
```

Chaque disque physique contient donc un mix des quatre types de blocs :

```
D1 (disque physique)
├── A1   (données, stripe 1)
├── B1   (données, stripe 2)
├── P    (parité XOR, stripe 3)
└── Q    (Reed-Solomon, stripe 4)
```

La logique est la même qu'en RAID 5 — distribuer la charge sur tous les disques pour éviter qu'un seul soit surchargé. Ici on distribue **deux** blocs de parité au lieu d'un, mais le principe de rotation reste identique.

```mermaid
graph TD
    S1[Stripe 1] --> A1[D1: A1]
    S1 --> A2[D2: A2]
    S1 --> A3[D3: A1⊕A2]
    S1 --> A4[D4: RS·A1·A2]
    S2[Stripe 2] --> B1[D1: B1]
    S2 --> B2[D2: B1⊕B2]
    S2 --> B3[D3: RS·B1·B2]
    S2 --> B4[D4: B2]
    style A3 fill:#ffaa00,color:#000
    style A4 fill:#4488ff,color:#fff
    style B2 fill:#ffaa00,color:#000
    style B3 fill:#4488ff,color:#fff
```

🟠 Orange = parité P (XOR) — 🔵 Bleu = parité Q (Reed-Solomon)

#### Scénarios de reconstruction RAID 6

```mermaid
graph TD
    P1[Panne 1 disque] --> R1[P suffit XOR inverse]
    P2[Panne 2 disques] --> R2[P + Q combinés]
    P3[P perdu + 1 disque] --> R3[Q prend le relais]
    style P1 fill:#ffaa00,color:#000
    style P2 fill:#ff4444,color:#fff
    style P3 fill:#ff4444,color:#fff
    style R1 fill:#44bb44,color:#fff
    style R2 fill:#44bb44,color:#fff
    style R3 fill:#44bb44,color:#fff
```

---

## Pourquoi XOR ne suffit pas pour 2 pannes

Avec 3 disques + P (XOR) :

```
D1=3   D2=5   D3=6   P = 3⊕5⊕6 = 0
```

Si D1 et D2 tombent simultanément :

```
D1=?   D2=?   D3=6   P=0

On sait que : D1⊕D2⊕6 = 0
Donc        : D1⊕D2   = 6

→ infinité de solutions : (1,7) (2,4) (3,5)...
  impossible de déterminer D1 et D2 séparément
```

XOR donne une **somme sans identité** — impossible de résoudre deux inconnues avec une seule équation.

---

## Reed-Solomon — principe

Reed-Solomon traite les données comme des **points sur une courbe polynomiale** :

```
Polynôme :  f(x) = A1·x² + A2·x + A3

D1 = f(1)
D2 = f(2)
D3 = f(3)
P  = f(4)   ← parité XOR
Q  = f(5)   ← point supplémentaire sur la courbe
```

Avec 2 points connus, on peut toujours reconstruire un polynôme de degré 2 :

```
2 disques perdus = 2 inconnues
P + Q            = 2 équations indépendantes
→ système soluble, solution unique
```

### Pourquoi ne pas utiliser Reed-Solomon partout ?

||XOR (P)|Reed-Solomon (Q)|
|---|---|---|
|Calcul|Ultra rapide|Coûteux en CPU|
|Reconstruction|Simple|Complexe|
|Tolère|1 panne|2 pannes|

C'est un **compromis coût/bénéfice** : RS n'est utilisé que quand on a besoin de tolérer 2 pannes simultanées, car il alourdit chaque opération d'écriture.

---

## Les RAID composés (nested RAID)

### RAID 10 — Mirror + Stripe

On crée des paires miroir (RAID 1), puis on stripe (RAID 0) entre elles.

```mermaid
graph LR
    D[Données] --> S[RAID 0]
    S --> G1[Paire 1 RAID 1]
    S --> G2[Paire 2 RAID 1]
    G1 --> A[D1]
    G1 --> B[D2 miroir]
    G2 --> C[D3]
    G2 --> E[D4 miroir]
```

- ✅ Failover par paire + load balancing entre paires
- ❌ Capacité utile = 50%
- 📌 Idéal pour les bases de données à forte charge

---

### RAID 01 — Stripe + Mirror

On stripe d'abord, puis on duplique le groupe entier. Moins résilient que le RAID 10.

```mermaid
graph LR
    D[Données] --> M[RAID 1]
    M --> G1[Groupe 1 RAID 0]
    M --> G2[Groupe 2 RAID 0 copie exacte]
    G1 --> A[D1]
    G1 --> B[D2]
    G2 --> C[D3]
    G2 --> E[D4]
```

---

### RAID 50 — RAID 5 + Stripe

Plusieurs groupes RAID 5 sont striés ensemble. Dans chaque groupe, la parité tourne entre les disques physiques.

```mermaid
graph LR
    D[Données] --> S[RAID 0]
    S --> G1[Groupe 1 RAID 5]
    S --> G2[Groupe 2 RAID 5]
    G1 --> A[D1 données+P]
    G1 --> B[D2 données+P]
    G1 --> C[D3 données+P]
    G2 --> E[D4 données+P]
    G2 --> F[D5 données+P]
    G2 --> G[D6 données+P]
```

> Chaque disque contient un mix données/parité selon le stripe — P tourne entre D1, D2, D3 dans le groupe 1.

---

### RAID 60 — RAID 6 + Stripe

Comme le RAID 50 mais avec double parité (P+Q) tournante par groupe. Tolère 2 disques défaillants par groupe.

```mermaid
graph LR
    D[Données] --> S[RAID 0]
    S --> G1[Groupe 1 RAID 6]
    S --> G2[Groupe 2 RAID 6]
    G1 --> A[D1 données+P+Q]
    G1 --> B[D2 données+P+Q]
    G1 --> C[D3 données+P+Q]
    G1 --> Cq[D4 données+P+Q]
    G2 --> E[D5 données+P+Q]
    G2 --> F[D6 données+P+Q]
    G2 --> G[D7 données+P+Q]
    G2 --> Gq[D8 données+P+Q]
```

> P (XOR) et Q (Reed-Solomon) tournent tous les deux entre les disques de chaque groupe.

---

## Failover et Load Balancing

### Failover — RAID 1 (état normal)

```mermaid
graph LR
    C[Client] --> D1[D1 actif]
    D1 -.miroir.-> D2[D2 en attente]
```

### Failover — RAID 1 (après panne)

```mermaid
graph LR
    C[Client] --> D1[D1 ✗ mort]
    C --> D2[D2 ✓ prend le relais]
    style D1 fill:#ff4444,color:#fff
    style D2 fill:#44bb44,color:#fff
```

### Failover — RAID 5 (reconstruction par parité tournante)

Quand un disque tombe, le système recalcule les données manquantes via XOR à partir des blocs survivants — peu importe que le bloc perdu soit une donnée ou un bloc de parité.

```mermaid
graph TD
    C[Client]
    C --> D1[D1 A1 · B1 · C1⊕C2]
    C --> D2[D2 ✗ mort]
    C --> D3[D3 A1⊕A2 · B2 · C2]
    D1 & D3 --> R[Reconstruction XOR A2 = A1⊕A1⊕A2 B1⊕B2 recalculé]
    style D2 fill:#ff4444,color:#fff
    style R fill:#ffaa00,color:#fff
```

### Load Balancing — RAID 0

```mermaid
graph LR
    C[Requête A1 A2 A3 A4] --> D1[D1 A1 · A3]
    C --> D2[D2 A2 · A4]
    D1 & D2 --> R[Résultat A1 A2 A3 A4]
```

### Failover + Load Balancing — RAID 10 (après panne)

```mermaid
graph LR
    C[Client] --> S[RAID 0 LB maintenu]
    S --> G1[Paire 1 RAID 1]
    S --> G2[Paire 2 RAID 1]
    G1 --> D1[D1 ✗ mort]
    G1 --> D2[D2 ✓ failover]
    G2 --> D3[D3]
    G2 --> D4[D4]
    style D1 fill:#ff4444,color:#fff
    style D2 fill:#44bb44,color:#fff
```

---

## Résumé comparatif

|RAID|Redondance|Perf. lecture|Perf. écriture|Capacité|Usage typique|
|---|---|---|---|---|---|
|0|❌|⬆⬆|⬆⬆|100%|Cache, vidéo|
|1|✅ 1 disque|⬆|➡|50%|OS, petits serveurs|
|5|✅ 1 disque|⬆|➡|(n-1)/n|Serveurs fichiers|
|6|✅ 2 disques|⬆|⬇|(n-2)/n|Archivage|
|10|✅ 1/paire|⬆⬆|⬆|50%|BDD, prod critique|
|50|✅ 1/groupe|⬆⬆|⬆|bonne|NAS, SAN|
|60|✅ 2/groupe|⬆⬆|➡|correcte|Datacenter|
