# LunarSync

**Cross-Sensor Lunar Image Correspondence & Change Detection**
Chandrayaan-2 OHRC ↔ TMC-2 Registration Pipeline

---

## 🛰️ Pipeline at a Glance

```mermaid
flowchart TD
    A["📥 INPUT\nOHRC / TMC + PDS4 Metadata + DEM"] --> B

    subgraph S1["1️⃣ PHYSICS-AWARE PREPROCESSING"]
        B["SPICE Observation Geometry\nSun / View / Acquisition angles"] --> C
        C["Orthorectification\nSPICE + DEM → Geometry + Scale correction"] --> D
        D["Illumination Normalization\nHapke / Ratio → removes sun-angle effect"] --> E
        E["Shadow Mask\nDEM + Sun Geometry → reliable vs shadow regions"]
    end

    E --> F

    subgraph S2["2️⃣ MODEL RECOMMENDATION SYSTEM"]
        F["SuperPoint + LightGlue\n(diagnostic probe)"] --> G{Difficulty?}
        G -- Easy --> H1["SuperPoint + LightGlue"]
        G -- Medium --> H2["LoFTR"]
        G -- Hard --> H3["RoMa v2"]
    end

    H1 --> I
    H2 --> I
    H3 --> I

    subgraph S3["3️⃣ CORRESPONDENCE MATCHING"]
        I["Easy → Sparse matches + confidence\nMedium → Semi-dense matches\nHard → Dense matches + confidence"]
    end

    I --> J

    subgraph S4["4️⃣ ROBUST GEOMETRIC VERIFICATION"]
        J["MAGSAC++\nInlier/Outlier detection + robust transform estimate"]
    end

    J --> K

    subgraph S5["5️⃣ SPATIALLY UNIFORM MATCH SELECTION"]
        K["Confidence-Weighted Hash Grid\n→ reliable, evenly-spread inliers"]
    end

    K --> L

    subgraph S6["6️⃣ CONFIDENCE-WEIGHTED TRANSFORMATION REFINEMENT"]
        L["Minimize Σ wᵢ · residualᵢ²\n→ Refined Transformation"]
    end

    L --> M

    subgraph S7["7️⃣ FULL-RESOLUTION WARPING"]
        M["Apply transformation to ORIGINAL full-res image"] --> N["📦 Registered Product"]
    end

    N --> O

    subgraph S8["8️⃣ VALIDATION & QUALITY CONTROL"]
        O["Inlier Ratio · RMSE · Registration Error\nSpatial Coverage · SUCCESS/FAIL"]
    end

    O --> P

    subgraph S9["9️⃣ CHANGE DETECTION"]
        P["Registered Image + Global Reference Map"] --> Q["Same-Location Comparison"]
        Q --> R["SSIM"]
        R --> S["🎯 True Change + Lat/Lon"]
    end
```

---

## 🧩 The 9 Stages — Visual Summary

| # | Stage | Input → Output |
|---|---|---|
| 1 | **Physics-Aware Preprocessing** | Raw OHRC/TMC + DEM → geometry-corrected, illumination-normalized, shadow-mapped images |
| 2 | **Model Recommendation System** | Image pair → routed to Easy / Medium / Hard tier |
| 3 | **Correspondence Matching** | Routed pair → sparse / semi-dense / dense matches, each with confidence |
| 4 | **Robust Geometric Verification** | Raw matches → inliers only, via MAGSAC++ |
| 5 | **Spatially Uniform Match Selection** | Inliers → evenly-distributed, confidence-weighted subset |
| 6 | **Confidence-Weighted Transformation Refinement** | Selected matches → final transformation matrix |
| 7 | **Full-Resolution Warping** | Transformation + original full-res image → registered product |
| 8 | **Validation & QC** | Registered product → pass/fail metrics |
| 9 | **Change Detection** | Registered image + reference map → confirmed changes with lat/lon |

---

## 🧭 Stage 1 — Physics-Aware Preprocessing

```mermaid
flowchart LR
    A[SPICE Geometry] --> B[Orthorectification]
    B --> C[Illumination Normalization]
    C --> D[Shadow Mask]
```

| Block | What it fixes |
|---|---|
| SPICE Observation Geometry | Where the sun and spacecraft actually were at capture time |
| Orthorectification | Scale mismatch + viewing-angle distortion (SPICE + DEM) |
| Illumination Normalization | Shadow/brightness mismatch caused by different sun angles (Hapke / Ratio) |
| Shadow Mask | Flags which regions are reliable vs. shadow-corrupted |

---

## 🧭 Stage 2 — Model Recommendation System

```mermaid
flowchart TD
    A["SuperPoint + LightGlue\n(diagnostic probe)"] --> B{Quality Gate}
    B -- Easy --> C["SuperPoint + LightGlue"]
    B -- Medium --> D["LoFTR"]
    B -- Hard --> E["RoMa v2"]
```

> Only **one** correspondence model ever runs per pair for the actual matching — the probe decides which one, so no redundant heavy-model passes.

---

## 🧭 Stage 3–6 — Matching → Verification → Selection → Refinement

```mermaid
flowchart LR
    A[Matches + Confidence] --> B["MAGSAC++\nInlier/Outlier"]
    B --> C["Confidence-Weighted\nHash Grid"]
    C --> D["Weighted Transformation Fit\nΣ wᵢ·residualᵢ²"]
```

- **High-confidence matches** → strongly influence the final transformation
- **Low-confidence matches** (shadow-boundary, low-texture) → automatically down-weighted, never discarded outright

---

## 🧭 Stage 7–8 — Warping & Validation

```mermaid
flowchart LR
    A[Refined Transformation] --> B["Apply to ORIGINAL\nfull-resolution image"]
    B --> C[Registered Product]
    C --> D{QC Metrics}
    D -- Pass --> E[✅ Registration SUCCESS]
    D -- Fail --> F[❌ Registration FAIL]
```

| QC Metric | Checks |
|---|---|
| Total Correspondences | How many matches were found overall |
| Inlier Match Count / Ratio | How many survived geometric verification |
| Reprojection RMSE | Average pixel error after alignment |
| Registration Error | Overall alignment accuracy |
| Spatial Match Coverage | Whether matches span the whole image, not just one corner |

---

## 🧭 Stage 9 — Change Detection

```mermaid
flowchart LR
    A[Registered Image] --> C[Same-Location Comparison]
    B[Global Reference Map] --> C
    C --> D[SSIM]
    D --> E["🎯 True Change + Lat/Lon"]
```

---

## 🗂️ Models Used

| Model | Role |
|---|---|
| **SuperPoint + LightGlue** | Fast diagnostic probe + final matcher for easy pairs |
| **LoFTR** | Medium-difficulty semi-dense matcher |
| **RoMa v2** | Dense matcher for hard pairs (low-texture, heavy-shadow, large scale-gap) |
| **MAGSAC++** | Robust geometric outlier rejection |

---

## 📌 One-Line Summary

**Correct for physics first → pick the cheapest matcher that will work → verify robustly → weight everything by confidence → warp the original full-resolution image → check it actually passed → compare against the global map to flag real surface change.**
