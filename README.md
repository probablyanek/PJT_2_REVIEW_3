# ML-Based Asynchronous Sensor Fusion for Indoor Localization

This repository contains the technical documentation, presentation source, and architectural overview for an edge-deployed sensor fusion system. The project leverages 24 GHz FMCW Radar and 802.11mc Wi-Fi Fine Timing Measurement (FTM) to provide robust indoor positioning on low-power microcontrollers.

## Project Overview

Indoor localization often fails when relying on a single sensing modality. Radar provides high precision but suffers from Line-of-Sight (LoS) requirements, while Wi-Fi FTM can penetrate obstacles but remains significantly noisier due to multipath reflections. 

This system fuses both streams using a Multi-Layer Perceptron (MLP) deployed on an ESP32-S3. By employing a "Forward-Fill" feature engineering strategy and operating in polar coordinates $(r, \theta)$, the model maintains sub-20cm accuracy even during partial sensor dropouts or occlusion.

## Key Architectural Decisions

### 1. Polar Coordinate Fusion
To avoid circular dependencies, the system operates in Polar coordinates. Since Wi-Fi FTM provides range only, and Radar provides both range and angle, the $r$-axis serves as the common denominator for direct comparison and fusion without requiring premature projection into Cartesian $(x, y)$ space.

### 2. Dual-Core RTOS Isolation
High-speed UART data from the Radar (256k baud) is processed on Core 1, while Wi-Fi FTM bursts—which are timing-critical and often disable interrupts—are pinned to Core 0. This strict isolation prevents HW FIFO overflows and ensures zero dropped radar frames during active ranging.

### 3. Edge Inference (TFLite Micro)
The fusion model is a compact MLP (2,474 parameters) deployed using TensorFlow Lite Micro. The implementation utilizes Float32 precision rather than INT8 quantization to preserve regression accuracy, as the 11.6 KB memory footprint is negligible for the ESP32-S3’s internal SRAM.

### 4. Multipath Mitigation
The system forces 40 MHz (HT40) bandwidth to improve the Wi-Fi resolution floor and employs a Min-RTT filter across 16-frame bursts to isolate the true line-of-sight path from secondary reflections.

## Repository Structure

- `main.tex`: LaTeX/Beamer source for the technical presentation (Review-3).
- `PLAN.md`: Detailed content blueprint and data tables for every slide.
- `img/`: High-resolution results, training curves, and physical setup diagrams.
- `TikZDocs/`: Documentation for the system architecture diagrams generated in TikZ.

## Requirements and Compilation

### Presentation
The presentation is built using the Beamer class with the Madrid theme. To compile the source into a PDF:

1. Ensure a TeX distribution is installed (e.g., MiKTeX or TeX Live).
2. Run the following command twice to resolve TikZ positions and references:
   ```bash
   pdflatex main.tex
   ```

### Hardware Context
The implementation targets the following hardware:
- **Node:** ESP32-S3 (SRAM-based inference).
- **Radar:** HLK-LD2450 (UART interface).
- **Tag:** ESP32 (FTM Responder mode).
- **Reflector:** Trihedral corner reflector for RCS enhancement.

## Status: Review-3
Current results demonstrate a mean Euclidean error of 0.25m on the test set across 12 segments (8 clear, 4 occluded) in a dense hostel room environment. Remaining work focuses on extended range validation (up to 6m) and 3D-printed enclosure deployment.
