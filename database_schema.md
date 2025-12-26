# F1 Data Management - Schema Database SQL

## Panoramica

| Tabella | Descrizione | Records (stimati) |
|---------|-------------|-------------------|
| `seasons` | Anni di stagione F1 | 9 |
| `teams` | Team per stagione con WCC e ATR | ~90 |
| `drivers` | Piloti con WDC, punti e vittorie | ~180 |
| `events` | GP con pole position | ~190 |
| `regulations` | Regolamenti FIA (da OpenAI) | ~37 |
| `circuit_performance` | Confronto Pre/Post 2022 (9 circuiti stabili) | 9 |

---

## Diagramma ER

```mermaid
erDiagram
    seasons ||--o{ teams : contains
    teams ||--o{ drivers : contains
    seasons ||--o{ events : contains
    regulations ||--|| seasons : references
    circuit_performance ||--|| events : "derived from"

    seasons {
        int year PK
    }
    
    teams {
        int id PK
        int year FK
        string name
        bool is_wcc
        float atr_percentage
        int wt_hours_annual
    }
    
    drivers {
        int id PK
        int team_id FK
        string name
        string code
        bool is_wdc
        float points
        int wins
        int races_count
        int first_round
        int last_round
    }
    
    events {
        int id PK
        int year FK
        int round
        string gp_name
        string circuit
        string circuit_id
        string pole_time
        float pole_time_seconds
        string pole_driver
        string pole_team
    }
    
    regulations {
        int id PK
        int year FK
        string type
        string description
        string impact
        string source
        string era
    }
    
    circuit_performance {
        string circuit FK
        string circuitID PK
        float pre_best
        int pre_year
        float post_best
        int post_year
        float delta
        float pct
        string era_diff
    }
```

---

## Dettaglio Tabelle

### `seasons`
Anno della stagione F1.

| Colonna | Tipo | Note |
|---------|------|------|
| `year` | INTEGER | PK, 2017-2025 |

### `teams`
Team per ogni stagione con info WCC e ATR.

| Colonna | Tipo | Note |
|---------|------|------|
| `year` | INTEGER | FK → seasons |
| `name` | TEXT | Nome team |
| `is_wcc` | INTEGER | 1 = vincitore WCC |
| `atr_percentage` | REAL | % ATR (solo 2022+) |
| `wt_hours_annual` | INTEGER | Ore wind tunnel annuali |

### `drivers`
Piloti con statistiche stagionali.

| Colonna | Tipo | Note |
|---------|------|------|
| `team_id` | INTEGER | FK → teams.rowid |
| `name` | TEXT | Nome completo |
| `code` | TEXT | Codice 3 lettere (VER, HAM...) |
| `is_wdc` | INTEGER | 1 = vincitore WDC |
| `points` | REAL | Punti totali stagione |
| `wins` | INTEGER | Vittorie |
| `races_count` | INTEGER | Gare disputate |
| `first_round` | INTEGER | Prima gara |
| `last_round` | INTEGER | Ultima gara |

### `events`
Gran Premi con dati pole position.

| Colonna | Tipo | Note |
|---------|------|------|
| `year` | INTEGER | FK → seasons |
| `round` | INTEGER | Numero gara |
| `gp_name` | TEXT | Nome GP |
| `circuit` | TEXT | Nome circuito |
| `circuit_id` | TEXT | ID stabile per entity matching |
| `pole_time` | TEXT | Tempo pole (mm:ss.xxx) |
| `pole_time_seconds` | REAL | Tempo in secondi |
| `pole_driver` | TEXT | Poleman |
| `pole_team` | TEXT | Team poleman |

### `regulations`
Regolamenti FIA estratti via OpenAI API.

| Colonna | Tipo | Note |
|---------|------|------|
| `year` | INTEGER | FK → seasons |
| `type` | TEXT | Aero/Financial/Safety/Engine |
| `description` | TEXT | Descrizione dettagliata |
| `impact` | TEXT | Major/Minor/Revolutionary |
| `source` | TEXT | Fonte (FIA + anno) |
| `era` | TEXT | Pre-2022 / Post-2022 |

### `circuit_performance`
Tabella derivata: confronto giro più veloce Pre vs Post 2022 su 9 circuiti stabili (layout invariato).

| Colonna | Tipo | Note |
|---------|------|------|
| `Circuit` | TEXT | FK - Nome circuito |
| `CircuitID` | TEXT | PK - Nome circuito |
| `Pre_Best` | REAL | Miglior tempo pre-2022 (secondi) |
| `Pre_Year` | INTEGER | Anno del record pre-2022 |
| `Post_Best` | REAL | Miglior tempo post-2022 (secondi) |
| `Post_Year` | INTEGER | Anno del record post-2022 |
| `Delta` | REAL | Differenza (+ = più lento, - = più veloce) |
| `Pct` | REAL | Variazione percentuale |
| `Era_Diff` | TEXT | FASTER / SLOWER |

---

## Query Utili

### WDC/WCC Winners
```sql
SELECT t.year, d.name AS wdc, t.name AS team, 
       (SELECT name FROM teams WHERE year=t.year AND is_wcc=1) AS wcc
FROM teams t JOIN drivers d ON t.rowid = d.team_id
WHERE d.is_wdc = 1 ORDER BY t.year;
```

### Regulations by Year
```sql
SELECT year, type, description, impact
FROM regulations ORDER BY year, type;
```

### Pole Positions per Driver
```sql
SELECT pole_driver, COUNT(*) as poles 
FROM events GROUP BY pole_driver 
ORDER BY poles DESC LIMIT 10;
```

### Circuit Performance: Pre vs Post 2022
```sql
SELECT Circuit, Delta, Pct, Era_Diff
FROM circuit_performance
ORDER BY Delta;
```

