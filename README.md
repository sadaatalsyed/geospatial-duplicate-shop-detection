# Geospatial Duplicate Shop Detection

An entity-resolution engine that identifies duplicate retail-shop records in a field-sales database using a combination of **spatial indexing**, **text normalisation**, **graph construction**, and **DFS-based connected-component detection**.

---

## The Problem

A secondary-sales platform onboards tens of thousands of retail shops via field agents who manually enter shop names and GPS coordinates. Over time the same physical shop accumulates multiple database records under slightly different names:

| Record A | Record B |
|---|---|
| `Ali General Store` | `Ali Gen Stor` |
| `Ahmed Cosmetics` | `Ahmed Cosmetic Shop` |
| `Noor Mart` | `Noor Mart & Bakery` |

These duplicates inflate shop counts, split sales history across records, and break downstream credit-limit logic that depends on a unique shop identity.

**Goal:** Flag every group of records that likely refers to the same physical shop, so the operations team can review and merge them.

---

## Algorithm Overview

Brute-force pairwise comparison across 40,000+ shops is O(n²) and takes hours. This engine reduces it to seconds with a 3-layer approach:

### Layer 1 — Spatial Indexing (500m grid)

The city is divided into a grid of 500m × 500m cells. Each shop is assigned to a cell based on its GPS coordinates. A shop can only be a duplicate of another shop in its own cell or one of the 8 directly adjacent cells (9-cell Moore neighbourhood). This reduces the comparison universe from the full dataset to ~15–30 shops per neighbourhood — a ~99% reduction in comparisons.

```
┌───┬───┬───┐
│ x │ x │ x │
├───┼───┼───┤   Each shop only compared against
│ x │ ● │ x │   shops in its 9-cell neighbourhood
├───┼───┼───┤
│ x │ x │ x │
└───┴───┴───┘
```

### Layer 2 — Text Normalisation + Prefix Matching

Before comparing names:
1. Lowercase and strip punctuation
2. Remove high-frequency generic words (`store`, `mart`, `pharmacy`, `general`, etc.)
3. Extract the "core name" — the distinctive owner/brand identifier

Two shops are **name-similar** if their first 4 core-name characters match. This O(1) check has a low false-negative rate for shops in the same 500m cell: if two shops on the same street share the first 4 characters of their distinctive name, they are almost certainly the same shop.

> **Design note:** A weighted fuzzy-score approach (`fuzz.token_set_ratio` + `partial_ratio` + `WRatio`) was implemented and benchmarked. Prefix-matching was chosen for production because it runs 10x faster with comparable accuracy at this spatial scale. The fuzzy-scoring code is retained (commented out) in `shop_deduplication_engine.py` for reference.

### Layer 3 — Similarity Graph + DFS Connected Components

Shops that pass **both** the proximity check (≤200m Haversine distance) and the name-similarity check are connected by an edge in an undirected graph.

```
  Shop_A ──── Shop_B
     │
  Shop_C          Shop_D ──── Shop_E
```

**Depth-First Search** then traverses the graph to find all connected components. Each component with more than one node is a duplicate group. All shops in a group receive the same `SimilarGroupTag`.

---

## Output

Each input shop record gets two new columns:

| Column | Description |
|---|---|
| `SimilarGroupTag` | `"Unique"` if no duplicates found; otherwise a shared tag identifying the duplicate cluster |
| `ClosestSimilarShop` | ShopCode of the nearest duplicate within the group |
| `DistanceToClosestShop_m` | Ground distance (metres) to the closest duplicate |

Example:

| VizShopCode | VizShopName | SimilarGroupTag | ClosestSimilarShop | DistanceToClosestShop_m |
|---|---|---|---|---|
| S001 | Ali General Store | S001_S002 | S002 | 47.3 |
| S002 | Ali Gen Stor | S001_S002 | S001 | 47.3 |
| S003 | Noor Pharmacy | Unique | null | null |

---

## Repository Structure

```
geospatial-duplicate-shop-detection/
│
├── shop_deduplication_engine.py   # Full algorithm: spatial index → graph → DFS → output
│
├── utils/
│   └── logging_setup.py           # Structured logging shared utility
│
├── data/                          # NOT committed (see .gitignore)
│   ├── shops.xlsx                 # Input: VizShopCode, VizShopName, Lat, Long
│   └── shops_with_duplicate_groups.csv   # Output
│
├── requirements.txt
├── .gitignore
└── README.md
```

---

## How to Run

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Place your input file at data/shops.xlsx
#    Required columns: VizShopCode, VizShopName, Lat, Long

# 3. Run
python shop_deduplication_engine.py
```

Output is written to `data/shops_with_duplicate_groups.csv`.

---

## Complexity Analysis

| Step | Time Complexity | Notes |
|---|---|---|
| Text normalisation | O(n) | One pass per shop |
| Sector assignment | O(n) | O(1) per shop |
| Graph construction | O(n × k²) | k = avg. shops per 9-cell neighbourhood (~15–30) |
| DFS connected components | O(V + E) | V = shops, E = duplicate edges |
| Closest-shop annotation | O(g × m²) | g = groups, m = avg. group size (~2–5) |
| **Total** | **~O(n)** practical | vs O(n²) brute-force |

---

## Configuration

All tunable parameters are defined as module-level constants at the top of `shop_deduplication_engine.py`:

| Constant | Default | Description |
|---|---|---|
| `SECTOR_SIZE_METERS` | 500 | Grid cell edge length |
| `PROXIMITY_THRESHOLD_METERS` | 200 | Max distance to be duplicate candidates |
| `FUZZY_MATCH_THRESHOLD` | 85 | Min score if fuzzy matching is enabled |
| `GENERIC_WORDS` | `{"store", "mart", ...}` | Words stripped before name comparison |

---

## Tech Stack

| Tool | Role |
|---|---|
| Python 3.11+ | Core language |
| pandas | Data loading and output |
| Haversine (math stdlib) | GPS distance calculation |
| fuzzywuzzy | Fuzzy name matching (available, not active by default) |
| DFS / graph (collections.defaultdict) | Connected-component detection |

---

## Author

**Syed Basit Hussain Shah** — Data Engineer / Analytics Engineer  
[GitHub](https://github.com/sadaatalsyed) · [LinkedIn](https://www.linkedin.com/in/basithussain0793/)
