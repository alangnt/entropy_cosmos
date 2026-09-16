# Entropy Cosmos

A 3D N-body gravitational simulation of an accretion disk: 1,000 planets orbiting a
supermassive central body, integrated with the **Barnes-Hut algorithm** for O(N log N)
force computation, parallelized with OpenMP, and replayed as a real-time 3D animation.

## How It Works

The universe starts as a flat, rotating disk. A central "black hole" of 8.54×10³⁶ kg sits
at the origin, surrounded by 1,000 planets scattered randomly inside a 400-million-metre
cube. Each planet's mass is drawn from a log-normal distribution around one Earth mass,
and each is given the exact tangential velocity needed for a circular orbit
(`v = sqrt(G·M / r)`), so the system settles into a disk rather than collapsing immediately.

Each time step:

1. Build an octree over all active bodies, accumulating total mass and centre of mass per node.
2. Compute gravitational forces by Barnes-Hut traversal — a node farther away than
   `s/d < θ` is collapsed into a single point mass (parallelized with OpenMP).
3. Update accelerations, velocities and positions with Euler integration.
4. Run an accretion pass: any two bodies whose physical radii overlap merge into one,
   conserving mass and momentum. The absorbed body is deactivated.
5. Every 3,600 steps, append every body's position and mass to `orbit.csv`.

Physical radii come from each body's density (rocky, gas giant or stellar, chosen by mass),
so heavier bodies sweep up a larger collision cross-section as they grow. A softening
parameter (`ε = 10`) keeps close-range forces finite.

## Building & Running

### Prerequisites

- **C++ compiler** with C++17 support (e.g. `g++` or `clang++`)
- **CMake** 3.16+
- **OpenMP** — on macOS, install via `brew install libomp`
- **Python 3** with the packages listed in `requirements.txt`

### Build

```bash
./build.sh
```

This wipes `build/`, configures a Release build with CMake and produces `build/universe`.
CMake locates Homebrew's `libomp` automatically on macOS.

To compile by hand instead:

```bash
# macOS
g++ -O3 -std=c++17 -Xpreprocessor -fopenmp \
    -I/opt/homebrew/opt/libomp/include \
    -L/opt/homebrew/opt/libomp/lib -lomp \
    main.cpp -o universe

# Linux
g++ -O3 -std=c++17 -fopenmp main.cpp -o universe
```

### Run the simulation

```bash
./build/universe
```

`orbit.csv` is written to the current working directory, so run it from the project root —
`visualize.py` expects to find it there. The file has one row per body per recorded frame:
`Step,PlanetID,X,Y,Z,Mass`.

### Visualize

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python visualize.py
```

Opens an interactive 3D animation of the disk. Marker size and colour both track each
body's mass (log-scaled), so mergers are visible as growing, reddening points.

## Project Structure

```
├── main.cpp          # Simulation engine (octree, Barnes-Hut, accretion, OpenMP)
├── visualize.py      # 3D scatter animation of orbit.csv
├── CMakeLists.txt    # Build configuration
├── build.sh          # Clean Release build helper
└── requirements.txt  # Python dependencies (PyQt6, pandas, matplotlib)
```

## Key Parameters

All defined at the top of [main.cpp](main.cpp), except the body counts and step count,
which live in `main()`.

| Parameter | Value | Description |
|-----------|-------|-------------|
| `GRAVITATIONAL_CONSTANT` | 6.6743×10⁻¹¹ | Gravitational constant |
| `DT` | 86,400 s | Integration time step (1 day) |
| `THETA` | 0.5 | Barnes-Hut opening angle criterion |
| `EPSILON` | 10 | Softening parameter |
| Central mass | 8.54×10³⁶ kg | Mass of the central body |
| Planets | 1,000 | Orbiting bodies, plus the central one |
| Steps | 360,000 | Total iterations |
| Snapshot interval | 3,600 steps | Rows written to `orbit.csv` (100 frames) |
