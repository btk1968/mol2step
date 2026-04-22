# Contributing to mol2step

This project is the starting point for a graduate synthesis class.  These
notes are aimed at students contributing extensions or improvements.

## Setting up your environment

```bash
# 1. Fork the repository on GitHub, then clone your fork
git clone https://github.com/<your-username>/mol2step.git
cd mol2step

# 2. Create the conda environment (recommended — cadquery needs OCCT)
mamba env create -f environment.yml
mamba activate mol2step

# 3. Verify the installation
python mol2step.py --help
python mol2step.py examples/benzene.mol --info
```

## Running the tool on a molecule

```bash
# Generate a STEP file
python mol2step.py examples/benzene.mol -o benzene.step

# Check junction geometry before opening in CAD
python mol2step.py examples/benzene.mol --check-step benzene.step

# Smaller atoms and thinner bonds (recommended starting point)
python mol2step.py molecule.cjson --vdw-scale 0.7 --bond-scale 0.5
```

## Project ideas for class extensions

Below are suggested directions.  Pick one, open an issue to claim it, then
submit a pull request when ready.

### Geometry and model quality
- **Colour coding** — export atom spheres with element colours (CPK scheme)
  as separate STEP bodies or as an OBJ/VRML with vertex colours
- **Space-filling (CPK) model** — use full VdW spheres with no tubes; just
  a compound of overlapping spheres per bond connection
- **Ribbon/cartoon backbone** — for peptides/proteins, a ribbon along the
  backbone instead of all-atom ball-and-stick
- **Printability analysis** — detect overhanging bonds (angle from vertical
  > threshold) and suggest orientations or flag support-needing regions

### Input format support
- **MOL2 / Tripos format** (`.mol2`) — widely used by ChimeraX
- **mmCIF / PDBx** — standard for macromolecular structures from the PDB
- **Direct PubChem fetch** — given a compound name or CID, fetch the 3D
  conformer via the PubChem REST API without a separate download step

### Output and workflow
- **Multi-material STL / 3MF** — one body per element, each tagged with a
  colour/material for multi-filament printers (Bambu AMS, Prusa MMU)
- **Assembly instructions** — for split-fragment models, generate a PDF with
  labelled part diagrams showing how pieces connect
- **FreeCAD macro** — a `.FCMacro` that automatically unions all bodies and
  applies standard fillets after STEP import

### Analysis
- **Steric clash detection** — report atom pairs closer than sum of VdW radii
  (useful for checking optimisation quality before printing)
- **Print-time estimate** — given slicer settings (layer height, speed),
  estimate filament use and print time from bounding box + volume

## Code style

- Python 3.11+, no external dependencies beyond `cadquery` and `numpy`
- All geometry in millimetres internally; Ångstroms only at the input boundary
- New functions should include a docstring explaining units and expected types
- Keep the CLI backwards-compatible (add flags, do not rename existing ones)

## Pull request checklist

- [ ] `python mol2step.py examples/benzene.mol -o /tmp/test.step` succeeds
- [ ] `python mol2step.py examples/benzene.mol --check-step /tmp/test.step`
      reports no GAP or MARGINAL junctions
- [ ] `python mol2step.py --help` renders without error
- [ ] No new files larger than ~1 MB committed (STEP/STL outputs go in
      `.gitignore`)
