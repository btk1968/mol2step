# mol2step — Molecular Geometry to Solid CAD (STEP/STL)

Generate true solid BREP CAD files of ball-and-stick molecular models from
molecular geometry files.  Produces real sphere-and-cylinder geometry with
OCCT rolling-ball fillets that imports cleanly into FreeCAD, OnShape, or any
STEP-compatible CAD program for further modification before 3D printing.

## Why this exists

Chemistry visualisation tools (Avogadro2, ChimeraX, PyMOL) can export
molecular models as STL or VRML, but these are triangle meshes — not editable
solid geometry.  Importing them into a CAD program gives you thousands of flat
facets instead of smooth cylinders and spheres.  This script bypasses the mesh
entirely by reading atomic coordinates and generating proper solid BREP
geometry directly.

## Architecture

Each atom becomes one **sphere primitive** sized to its van der Waals radius.
Each bond becomes one or more **cylinder primitives** (one strand per bond
order) sized to the hydrogen VdW radius times `--bond-scale`.

All sphere and cylinder primitives are collected into a flat list and fused
in a **single `BRepAlgoAPI_Fuse` call** (multi-tool mode) to produce one
solid BREP body.  This avoids the incremental pairwise orientation artefacts
that occur when fusing solids one-by-one.

`--no-fillet` skips the boolean entirely and exports a `Compound` of the raw
overlapping primitives — always valid STEP, useful as a quick preview or when
the fuse fails on unusual geometry.

Junction geometry (overlap and angle at every sphere–tube contact) is computed
analytically from the model parameters at build time and reported as a table.

## Pipeline

```
  Avogadro2 / ChimeraX / PubChem
            │
            ▼
  .cjson / .mol / .xyz / .pdb      ← molecular coordinates + bonds
            │
            ▼
      mol2step.py (CadQuery)        ← sphere + cylinder primitives → BRepAlgoAPI_Fuse
            │
            ▼
         .step file                  ← true BREP: single fused solid
            │
            ▼
    FreeCAD / OnShape                ← add struts/base, fillet edges if desired
            │
            ▼
       .stl / .3mf
            │
            ▼
       PrusaSlicer                   ← slice and print
```

## Installation

### CadQuery (required)

CadQuery wraps the OpenCASCADE kernel. Conda/mamba is the most reliable path:

```bash
# Install Miniforge if needed: https://github.com/conda-forge/miniforge
mamba create -n mol2step python=3.11 -y
mamba activate mol2step
mamba install -c conda-forge cadquery -y
pip install numpy          # usually pulled in by cadquery already
```

Alternatively: `pip install cadquery numpy` (may work on some systems).

### Get the script

```bash
cp mol2step.py ~/bin/         # or any directory on your PATH
chmod +x ~/bin/mol2step.py
```

## Quick start

```bash
# Generate STEP from a CJSON file (Avogadro2 format)
python mol2step.py caffeine.cjson -o caffeine.step

# Check junction geometry before opening in CAD
python mol2step.py caffeine.cjson --check-step caffeine.step

# Print sizing table without building
python mol2step.py caffeine.cjson --info
```

## Usage

```
python mol2step.py <input> [options]
```

### Positional argument

| Argument | Description |
|----------|-------------|
| `input`  | Molecule file: `.cjson`, `.mol`, `.sdf`, `.xyz`, `.pdb` |

### Output options

| Option | Default | Description |
|--------|---------|-------------|
| `-o / --output` | `<stem>.step` | Output file path |
| `-f / --format` | `step` | Output format: `step`, `stp`, or `stl` |

### Scale options

| Option | Default | Description |
|--------|---------|-------------|
| `--scale` | `10.0` | mm per Angstrom (position scale) |
| `--vdw-scale` | `1.0` | Atom sphere radius multiplier |
| `--bond-scale` | `0.7` | Bond tube radius multiplier |
| `--tube-radius` | — | Hard override for tube radius in mm (bypasses `--bond-scale`) |
| `--no-fillet` | off | Export raw overlapping sphere+cylinder compound (no boolean union) |

### Base plate

| Option | Default | Description |
|--------|---------|-------------|
| `--add-base` | off | Add a rectangular base plate for bed adhesion |
| `--base-thickness` | `3.0` | Base plate thickness in mm |
| `--base-margin` | `5.0` | Base plate margin beyond molecule bounding box in mm |

### Bond inference (XYZ / PDB only)

| Option | Default | Description |
|--------|---------|-------------|
| `--bond-tolerance` | `0.4` | Distance tolerance in Angstroms above sum of covalent radii |
| `--min-bond-dist` | `0.4` | Minimum interatomic distance (Å) treated as a bond |

### Diagnostic / informational

| Option | Description |
|--------|-------------|
| `--info` | Print atom and tube sizing table, then exit (no build) |
| `--debug` | Verbose geometry validation: bond lengths, sphere/tube clearances |
| `--check-step STEP_FILE` | Analyse junction overlap and angle for every bond endpoint |
| `--check-integrity STEP_FILE` | OCCT BREP integrity check per solid (topology, free edges, volume sign) |
| `--check-self-intersect` | Add self-intersection test to `--check-integrity` (slow) |
| `--no-center` | Do not translate molecule to origin before building |

## Scale convention

```
Atom sphere diameter  =  VdW_radius(A) * --vdw-scale * --scale
                      ->  1 A VdW  =  1 cm diameter  at defaults

Bond tube diameter    =  VdW_H (1.2 A) * --bond-scale * --scale
                      ->  defaults: 8.4 mm diameter  (bond-scale 0.7)
```

## Junction geometry

At each sphere-tube junction two metrics are computed and reported:

- **overlap** = `sphere_radius - tube_radius` — how far the tube embeds in the
  sphere.  Below 0.3 mm the junction is near-tangent and will show a visible
  seam.
- **angle** = `arcsin(sqrt(r^2 - t^2) / r)` — the angle between the bond axis
  and the sphere surface at the junction circle.  90 deg is ideal; below 10 deg
  the surfaces meet almost tangentially.

Both are printed as a table at build time.  Look for **ok** on every element.
Use `--check-step` to re-verify an already-generated file.

The default `--bond-scale 0.7` is chosen so that H spheres (the smallest in
organic molecules) have >= 1.8 mm overlap at default scale, giving a 44 deg
junction angle — well clear of the gap threshold.

## Parameter guide for 3D printing

| Parameter | Default | Recommended range | Notes |
|-----------|---------|-------------------|-------|
| `--scale` | 10.0 | 8 – 15 | Increase for larger, more robust print |
| `--vdw-scale` | 1.0 | 0.6 – 1.0 | 0.7 gives more open structure |
| `--bond-scale` | 0.7 | 0.5 – 0.7 | Below 0.5 bonds become fragile on FDM |
| `--no-fillet` | off | — | Export raw compound (no boolean); use for quick preview |
| `--base-thickness` | 3.0 | 2 – 5 | Thicker for larger molecules |

**Minimum printable tube diameter** on FDM is roughly 2 mm.  At `--scale 10`
and `--bond-scale 0.5` the tube diameter is 6 mm — well above this limit.

## Getting molecular geometry files

### From Avogadro2
1. Build or import your molecule
2. Optimise geometry: **Extensions → Optimise Geometry**
3. **File → Export → Molecule** — choose `.cjson` or `.mol`

### From UCSF ChimeraX
1. Open molecule (fetch from PubChem, PDB, etc.)
2. **File → Save** — choose `.mol2` or `.pdb`

### From PubChem
1. Go to https://pubchem.ncbi.nlm.nih.gov
2. Search for your molecule
3. **Download → 3D Conformer → SDF format**
4. Use the `.sdf` file directly (mol2step accepts `.sdf`)

### From OpenBabel
```bash
# Convert PDB to MOL with 3D coordinates
obabel input.pdb -O output.mol --gen3d

# Fetch from PubChem by name
obabel -:"caffeine" -O caffeine.mol --gen3d -ipub
```

## Post-processing in CAD

After generating the STEP file, open it in FreeCAD or OnShape.

### FreeCAD
1. **File → Open** the `.step` file
2. The model arrives as a single solid body — no union step needed
3. Add support struts or a stand with **Part → Primitives**
4. Apply **Part → Fillet** on junction edges if desired
5. **File → Export → STL or 3MF**

### OnShape
1. Import the `.step` file (arrives as a single solid in Part Studio)
2. Use Fillet / Chamfer tools as desired
3. **Right-click Part → Export → STL or 3MF**

> If you used `--no-fillet`, the file contains one body per primitive.
> Select all bodies and run **Boolean → Union** before exporting.

## Troubleshooting

**"No bonds found"**
For XYZ files, bonds are inferred from atomic distances.  Try increasing the
tolerance: `--bond-tolerance 0.5`.  For MOL files, verify the file has a bond
block (lines after the atom block containing three integers each).

**Junction seams / gaps on H atoms**
Run `--info` and check the overlap column.  If H shows near-zero overlap,
`--bond-scale` is too high.  The default 0.7 is chosen to avoid this.  Run
`--check-step` to get the full junction geometry table.

**Slow build for large molecules**
The `BRepAlgoAPI_Fuse` of all primitives in one call is the bottleneck.
Use `--no-fillet` for a quick preview (exports a compound with no boolean).

**OnShape import fails**
OnShape handles STEP well but occasionally chokes on very complex boolean
results.  Try opening in FreeCAD first, running **Part → Check Geometry**,
then re-exporting as STEP.

## Supported input formats

| Format | Extension | Bonds | Notes |
|--------|-----------|-------|-------|
| Chemical JSON | `.cjson` | Explicit | Avogadro2 native; best choice |
| MDL Molfile | `.mol` `.sdf` | Explicit | Widely supported |
| XYZ | `.xyz` | Inferred | Bonds from covalent radii + tolerance |
| PDB | `.pdb` | CONECT or inferred | Uses CONECT records if present |

## Credits and references

- CadQuery: https://github.com/CadQuery/cadquery
- OpenCASCADE Technology (OCCT): https://dev.opencascade.org
- MolPrint3D concept: Paukstelis, P.J. *J. Chem. Educ.* **2018**, *95*, 169-172
- VdW radii: Mantina et al. *J. Phys. Chem. A* **2009**, *113*, 5806;
  Alvarez *Dalton Trans.* **2013**, *42*, 8617
- Covalent radii: Cordero et al. *Dalton Trans.* **2008**, 2832
- Avogadro2: https://avogadro.cc
- UCSF ChimeraX: https://www.cgl.ucsf.edu/chimerax/
