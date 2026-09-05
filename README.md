# VLSI Course — Complete Self-Study Materials

> From EC background → Job-ready chip design & manufacturing expert, with NPTEL certification

## 📁 Repository Structure

```
VLSI_Practise/
├── docs/                          # GitHub Pages hosted course website
│   ├── index.html                 # Main landing page
│   ├── vlsi_01_electronics_cmos.html
│   ├── vlsi_02_memory_arithmetic.html
│   ├── vlsi_03_verilog_rtl.html
│   ├── vlsi_04_physical_design.html
│   ├── vlsi_05_fabrication.html
│   ├── vlsi_06_dft_lowpower.html
│   ├── vlsi_07_08_projects_nptel.html
│   └── vlsi_style.css
├── Assignments/                   # Course assignments and projects
│   └── Digital Assignment I.pdf
├── VIVADO_VLSI_CHEATSHEET.md      # Vivado & RTL Design Foundational Guide
└── .gitignore
```

## 🚀 Quick Start

### View the Course Website
This repository hosts a complete VLSI course website via GitHub Pages. Access it at:
- **GitHub Pages**: [View](https://utsav-pixel.github.io/VLSI_Practise/)
<!-- 
### Course Phases
The course is split into 7 phases, each with dedicated HTML files:

1. **Phase 1** — Electronics & CMOS Refresh (6–8 weeks)
2. **Phase 2** — Memory, Sequential & Arithmetic Circuits (6–8 weeks)
3. **Phase 3** — RTL Design & Verification (8–10 weeks)
4. **Phase 4** — Physical Design & STA (8 weeks)
5. **Phase 5** — Semiconductor Fabrication (4–5 weeks)
6. **Phase 6** — DFT, Low Power & Custom Layout (8 weeks)
7. **Phase 7 & 8** — Projects, NPTEL & Job Prep (10–12 weeks) -->

## 🛠️ Vivado & RTL Design Guide

For detailed Vivado workflow, SystemVerilog templates, testbench patterns, and best practices, see:
- **[VIVADO_VLSI_CHEATSHEET.md](./VIVADO_VLSI_CHEATSHEET.md)** — Complete reference for:
  - Project creation (GUI & Tcl scripted)
  - SystemVerilog vs Verilog
  - Three abstraction levels (gate-level, dataflow, behavioral)
  - FSM design patterns
  - Testbench patterns
  - Constraints (XDC)
  - Debugging in Vivado
  - Common pitfalls
<!-- 
## 📚 Tools Covered

| Tool | Purpose | Phase |
|------|---------|-------|
| LTSpice | SPICE simulation of MOSFETs & CMOS circuits | Phase 1 |
| iverilog + GTKWave | Verilog simulation & waveform viewing | Phase 3 |
| Yosys | Logic synthesis (RTL → gates) | Phase 3 |
| OpenSTA | Static timing analysis | Phase 4 |
| OpenROAD | Full physical design flow | Phase 4 |
| Magic VLSI + Sky130 PDK | Custom transistor-level layout | Phase 6 |
| KLayout | GDSII viewing | Phase 4/6 | -->

## 📝 Assignments

Course assignments are stored in the `Assignments/` folder. Each assignment should follow the structure outlined in the Vivado guide.

## 🔧 Development

### Local Preview
To preview the website locally:
1. Open `docs/index.html` in your browser
2. Navigate between phases using the top navigation bar

### Git Workflow
- Commit only source files (`.sv`, `.v`, `.xdc`, `.tcl`, `README.md`)
- Never commit Vivado-generated cache files (see `.gitignore`)
- One commit per meaningful step for readable history

## 📖 Philosophy

- **Understand every abstraction level** — gate → dataflow → behavioral
- **Never trust a waveform you haven't self-checked**
- **Document everything in text** so it's diffable on GitHub
- **Reproducible builds** using Tcl scripts

## 🎯 Roadmap

This repository follows a structured path from fundamentals to job-ready skills:

1. Combinational + sequential fundamentals
2. Standard reusable blocks (FIFO, UART, SPI, ALU)
3. Verification discipline (self-checking testbenches, SVA)
4. Small CPU core (RISC-V)
5. Timing closure & physical awareness

---

**Note**: This repository is designed for self-study with NPTEL certification preparation. The course materials are structured to be both a learning resource and a portfolio for job applications.
