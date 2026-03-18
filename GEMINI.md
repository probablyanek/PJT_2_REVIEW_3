# Project: ML-Based Asynchronous Sensor Fusion for Indoor Localization

This directory contains the LaTeX source code, content planning, and assets for a technical presentation (Review-3) regarding an edge-deployed neural fusion system on ESP32-S3 nodes.

## Directory Overview

The project focuses on fusing Radar (HLK-LD2450) and Wi-Fi FTM (802.11mc) data using a multi-layer perceptron (MLP) deployed on the edge (ESP32-S3) to achieve precise indoor localization, even under occlusion.

- **Purpose:** Technical presentation for "Review-3" (March 2026).
- **Core Technologies:** LaTeX (Beamer), TikZ (for diagrams), ESP32-S3, TensorFlow Lite Micro.
- **Key Concept:** Asynchronous sensor fusion using a "Forward-Fill" feature engineering strategy in polar coordinates $(r, \theta)$.

## Key Files

- `main.tex`: The primary LaTeX source file. It uses the `beamer` class and the `Madrid` theme. It contains 17 slides covering the entire project lifecycle from problem statement to results and remaining work.
- `PLAN.md`: A comprehensive markdown document that serves as the content blueprint for the presentation. It contains the text, table data, and logic descriptions for every slide.
- `img/`: Contains image assets:
    - `logo.png`: The institutional logo displayed in the header.
    - `Placeholder.png`: Temporary placeholders for results graphs (training loss, scatter plots, etc.).
- `TikZDocs/`: A subdirectory dedicated to TikZ diagram documentation or snippets.
- `.gitignore`: Configured to ignore common LaTeX auxiliary files (`.aux`, `.log`, `.nav`, `.out`, `.snm`, `.toc`, etc.).

## Usage

### Compiling the Presentation
To generate the PDF presentation, compile `main.tex` using a LaTeX distribution (like TeX Live or MiKTeX):

```bash
pdflatex main.tex
# or
xelatex main.tex
```

Note: You may need to run the command twice to resolve references and TikZ positions (especially for the header logo placement).

### Content Updates
- **Textual Changes:** Update `PLAN.md` first to maintain a clear record of the presentation logic, then apply the changes to the corresponding frames in `main.tex`.
- **Diagrams:** The diagrams are written in TikZ directly within `main.tex`. Styles are defined in the preamble (`sysblock`, `dataflow`, `coreblock`, etc.).
- **Images:** Replace `Placeholder.png` with actual result plots in the `img/` directory once data collection is finalized.

## Technical Details (Context for AI)
- **RTOS Architecture:** The project emphasizes core isolation on the ESP32-S3 (Wi-Fi on Core 0, UART/ML on Core 1) to prevent data corruption.
- **ML Model:** A small MLP (2,474 parameters, 11.6 KB) using Float32 precision (no quantization) to maintain regression accuracy for distance and angle.
- **Coordinate System:** Uses Polar $(r, \theta)$ to avoid circular dependencies when fusing 1D Wi-Fi range with 2D Radar data.
