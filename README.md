# 🧭 CppND Route Planning Project

## Implementation of A* Search Algorithm using OpenStreetMap Data

This project builds a **route planner** application in C++ that finds the shortest path between two points on a real-world map using the **A\* (A-Star) pathfinding algorithm**. It reads map data from [OpenStreetMap](https://www.openstreetmap.org/), computes the optimal route, and renders the result in an interactive graphical window.

Think of it as a **mini Google Maps** — you give it a start and end location, and it calculates and draws the shortest route on a real street map.

---

## 📖 Table of Contents

- [What Does This Program Do?](#-what-does-this-program-do)
- [How Does the Code Work?](#-how-does-the-code-work-step-by-step)
- [Project Structure](#-project-structure)
- [Prerequisites](#-prerequisites)
- [How to Run (Step-by-Step on Windows 11 using WSL)](#-how-to-run-step-by-step-on-windows-11-using-wsl)
- [Input & Output](#-input--output)
- [Common Errors & Fixes](#-common-errors--fixes)
- [Running Unit Tests](#-running-unit-tests)

---

## 🎯 What Does This Program Do?

When you run this program, it performs the following actions:

1. **Reads a real-world map** from an OpenStreetMap XML file (`map.osm`) containing roads, buildings, water bodies, parks, and railways.
2. **Asks the user** for a starting point `(x, y)` and a destination point `(x, y)` as coordinates between `0` and `100`.
3. **Runs the A\* Search Algorithm** to find the shortest path along actual roads between those two points.
4. **Prints the total distance** of the route in meters to the terminal.
5. **Opens a graphical window** (400×400 pixels) that renders the full map and highlights the computed route in **orange**, with a **green marker** at the start and a **red marker** at the destination.

### What You See on the Map Window

| Element | Color | Description |
|---|---|---|
| Background | Light beige `(238, 235, 227)` | Empty map area |
| Roads | White / Yellow / Pink | Streets of different types (motorway, residential, etc.) |
| Buildings | Light brown `(208, 197, 190)` | Building outlines and fills |
| Water | Blue `(155, 201, 215)` | Rivers, lakes, ponds |
| Parks / Leisure | Green `(189, 252, 193)` | Parks and recreational areas |
| Railways | Grey with dashes | Train tracks |
| **Your Route** | **Orange** | **The shortest path found by A\*** |
| **Start Point** | **Green square** | **Where the route begins** |
| **End Point** | **Red square** | **Where the route ends** |

---

## ⚙️ How Does the Code Work? (Step-by-Step)

The program executes in **4 stages**:

```
[ map.osm File ] → [ RouteModel (Graph) ] → [ RoutePlanner (A* Search) ] → [ Render (IO2D GUI Window) ]
```

### Stage 1: Reading the Map — `main.cpp` + `model.cpp` + `pugixml`

- The file `map.osm` is an XML file exported from OpenStreetMap. It contains raw geographic data: latitude/longitude coordinates of points (called **nodes**), and sequences of connected points forming roads and building outlines (called **ways**).
- The `pugixml` library (located in `thirdparty/pugixml/`) parses this XML and extracts all nodes and ways into C++ data structures.
- `model.cpp` organizes these into categorized collections: `Roads`, `Buildings`, `Waters`, `Railways`, `Leisures`, and `Landuses`.

### Stage 2: Building the Road Network — `route_model.cpp`

- `RouteModel` extends the base `Model` class and converts raw map nodes into searchable **graph nodes** (`RouteModel::Node`).
- Each node stores:
  - `x, y` — normalized coordinates (0.0 to 1.0)
  - `parent` — pointer to the previous node in the path (used for backtracking)
  - `g_value` — actual distance traveled from the start node
  - `h_value` — estimated distance to the destination (heuristic)
  - `visited` — whether A\* has already evaluated this node
  - `neighbors` — list of directly connected nodes via roads
- `CreateNodeToRoadHashmap()` builds a lookup table mapping each node to the roads passing through it, enabling fast neighbor discovery.
- `FindClosestNode(x, y)` takes user-input coordinates and finds the nearest actual road intersection on the map.

### Stage 3: A\* Pathfinding Algorithm — `route_planner.cpp`

This is the **core algorithm** of the project. A\* evaluates nodes using a cost function:

```
f(n) = g(n) + h(n)
```

Where:
- **g(n)** = actual distance traveled from start to node `n`
- **h(n)** = estimated straight-line (Euclidean) distance from node `n` to the destination
- **f(n)** = total estimated cost of the cheapest path passing through `n`

#### How the algorithm works:

1. **`RoutePlanner()` constructor** — Converts user input coordinates (0–100) to percentages (0.0–1.0), then finds the closest map nodes to the start and end positions.

2. **`AStarSearch()`** — The main loop:
   - Mark the start node as visited and add it to the `open_list`.
   - **While** the current node is not the destination:
     - Call `AddNeighbors()` to discover all connected road nodes.
     - Call `NextNode()` to pick the most promising node (lowest `f` value).
   - Once the destination is reached, call `ConstructFinalPath()` to trace the route.

3. **`AddNeighbors(current_node)`** — For the current intersection:
   - Find all neighboring intersections connected by roads.
   - For each unvisited neighbor: set its `parent` to the current node, calculate its `g_value` (distance so far + distance to neighbor) and `h_value` (straight-line distance to destination), and add it to the `open_list`.

4. **`NextNode()`** — Sorts the `open_list` by `f = g + h` in descending order, pops and returns the node with the **lowest** `f` value (most promising candidate).

5. **`ConstructFinalPath(destination)`** — Traces back from the destination node to the start by following `parent` pointers, accumulating the total distance. Reverses the path so it reads start → destination. Multiplies by `MetricScale()` to convert the normalized distance into **real-world meters**.

### Stage 4: Rendering the Map — `render.cpp` + IO2D Library

- `Render` takes the completed model (with the `path` vector populated by A\*) and draws it layer by layer onto an `io2d::output_surface`.
- Drawing order (back to front): Background → Land uses → Leisure areas → Water → Railways → Roads → Buildings → **Route path (orange)** → **Start marker (green)** → **End marker (red)**.
- The IO2D library creates a 400×400 pixel window at 30 FPS with auto-resize support.

---

## 📁 Project Structure

```
cppND_Route_planning/
├── CMakeLists.txt              # Build configuration (CMake)
├── map.osm                     # OpenStreetMap XML data file (the map)
├── map.png                     # Reference screenshot of the map
│
├── src/                        # Source code
│   ├── main.cpp                # Entry point: reads map file, takes user input, runs A*
│   ├── model.cpp / model.h     # Parses OSM XML into Roads, Buildings, Water, etc.
│   ├── route_model.cpp/.h      # Extends Model with graph nodes for pathfinding
│   ├── route_planner.cpp/.h    # A* search algorithm implementation
│   └── render.cpp / render.h   # Draws the map and route using IO2D graphics
│
├── test/                       # Unit tests
│   └── utest_rp_a_star_search.cpp  # Google Test cases for A* components
│
├── thirdparty/                 # Third-party libraries (included)
│   ├── pugixml/                # XML parser for reading map.osm
│   └── googletest/             # Google Test framework for unit testing
│
├── cmake/                      # Custom CMake find modules
├── lib/                        # Library output directory
└── build/                      # Build output directory (created during compilation)
```

---

## 📋 Prerequisites

### Why Each Prerequisite is Needed

| Tool / Library | Version Required | Why It's Needed |
|---|---|---|
| **CMake** | >= 3.11.3 | Automates the build process — finds libraries, links files, generates Makefiles |
| **GCC (g++)** | >= 7.4.0 | Compiles C++17 code (`std::optional`, `std::byte`, `std::string_view`) |
| **IO2D** (`P0267_RefImpl`) | Latest | Provides 2D graphics functions to open a window and draw the map |
| **Cairo** | Any recent | Low-level 2D vector graphics engine used by IO2D to draw lines and shapes |
| **GraphicsMagick** | Any recent | Image processing library for surface and texture handling |
| **libpng** | Any recent | PNG image support for Cairo |
| **pugixml** | Included | XML parser for reading `map.osm` (already in `thirdparty/`) |
| **Google Test** | Included | Unit testing framework (already in `thirdparty/`) |

---

## 🚀 How to Run (Step-by-Step on Windows 11 using WSL)

Since this project depends on the IO2D graphics library (which requires Cairo and X11), the easiest way to build and run on Windows 11 is using **WSL (Windows Subsystem for Linux)** with Ubuntu.

> **Note:** Windows 11 supports WSLg, which means Linux GUI windows (like the IO2D map) will automatically appear on your Windows desktop!

### Step 1: Install Ubuntu in WSL

Open **Command Prompt (`cmd`)** or **PowerShell** and run:

```cmd
wsl --install -d Ubuntu
```

- This downloads and installs Ubuntu inside WSL.
- When prompted, create a **username** (e.g., `surabhi`) and a **password**.
- After setup, you will see a green prompt like: `surabhi@PC-Name:~$`

### Step 2: Open Ubuntu Terminal

From now on, whenever you want to use Ubuntu:
- Search for **"Ubuntu"** in the Windows Start Menu and click it, **OR**
- In Command Prompt, type: `wsl -d Ubuntu`

### Step 3: Install Build Tools & Dependencies

Copy and paste this into your Ubuntu terminal:

```bash
sudo apt update && sudo apt install -y build-essential cmake g++ libgraphicsmagick1-dev libpng-dev libcairo2-dev git
```

Enter your Ubuntu password when prompted.

### Step 4: Download & Install the IO2D Library

```bash
cd ~
git clone --recurse-submodules https://github.com/cpp-io2d/P0267_RefImpl.git
cd P0267_RefImpl
mkdir build && cd build
cmake -DCMAKE_POLICY_VERSION_MINIMUM=3.5 -DIO2D_WITHOUT_SAMPLES=1 -DIO2D_WITHOUT_TESTS=1 ..
```

> **⚠️ If you get a `cstdint` or `uint16_t` compile error** (happens with GCC 13+), run this fix before `make`:
> ```bash
> sed -i '1i#include <cstdint>' ~/P0267_RefImpl/P0267_RefImpl/P0267_RefImpl/xinterchangebuffer.cpp
> ```

Then compile and install:

```bash
make -j$(nproc)
sudo make install
sudo ldconfig
```

### Step 5: Copy the Project to Linux Home Directory

Building directly on the Windows drive (`/mnt/c/`) causes permission errors. Copy the project into your Linux home directory:

```bash
cp -r "/mnt/c/Users/Surabhi M R/Downloads/cppND_Route_planning" ~/cppND_Route_planning
```

### Step 6: Compile the Route Planner

```bash
cd ~/cppND_Route_planning
rm -rf build && mkdir build && cd build
cmake -DCMAKE_POLICY_VERSION_MINIMUM=3.5 ..
make -j$(nproc)
```

If successful, you will see:
```
[100%] Linking CXX executable OSM_A_star_search
[100%] Built target OSM_A_star_search
```

### Step 7: Run the Program! 🎉

```bash
./OSM_A_star_search
```

---

## 📍 Input & Output

### Input

The program prompts you for **two pairs of coordinates** (x, y), each between **0** and **100**:

```
Enter Start Cordinate [ x y ]
10 10
Enter End Cordinate[ x y ]
90 90
```

| Value | Map Position |
|---|---|
| `0 0` | Bottom-left corner |
| `50 50` | Center of the map |
| `100 100` | Top-right corner |

### Example Inputs to Try

| Example | Start (x y) | End (x y) | Description |
|---|---|---|---|
| Long diagonal route | `10 10` | `90 90` | Bottom-left to top-right |
| Medium route | `25 25` | `75 75` | Quarter to three-quarter |
| Short route | `40 40` | `60 60` | Near the center |
| Horizontal route | `10 50` | `90 50` | Left to right along the middle |
| Vertical route | `50 10` | `50 90` | Bottom to top along the center |

### Output

#### 1. Terminal Output
The program prints the total path distance in meters:
```
Reading OpenStreetMap data from the following file: ../map.osm
Distance: 3124.52 meters.
```

#### 2. Graphical Window (IO2D)
A 400×400 pixel window opens on your desktop showing:
- The full street map rendered from `map.osm`
- Roads drawn in white/yellow/pink depending on road type (motorway, residential, etc.)
- Buildings in light brown, water in blue, parks in green
- **Your calculated route highlighted in orange**
- **Green square** at the start position
- **Red square** at the end position

#### 3. Re-running with Different Coordinates
1. Close the map window (click the ❌ button).
2. Run the program again:
   ```bash
   ./OSM_A_star_search
   ```
3. Enter new coordinates.

#### 4. Using a Custom Map File
```bash
./OSM_A_star_search -f ../your_custom_map.osm
```

---

## 🔧 Common Errors & Fixes

### Error 1: `'cmake' is not recognized`
**Cause:** CMake is not installed or not in your system PATH.
**Fix:**
```bash
sudo apt install -y cmake
```

### Error 2: `Could NOT find io2d`
**Cause:** The IO2D library has not been installed yet.
**Fix:** Follow [Step 4](#step-4-download--install-the-io2d-library) to clone, build, and install IO2D.

### Error 3: `Operation not permitted` (during cmake on `/mnt/c/`)
**Cause:** Building directly on the Windows file system (`/mnt/c/`) from WSL causes permission issues because Linux and Windows handle file permissions differently.
**Fix:** Copy the project to your Linux home directory first:
```bash
cp -r "/mnt/c/Users/YOUR_USERNAME/Downloads/cppND_Route_planning" ~/cppND_Route_planning
cd ~/cppND_Route_planning
```

### Error 4: `error: 'uint16_t' was not declared` (IO2D build)
**Cause:** GCC 13 and newer removed implicit `<cstdint>` includes. The IO2D source code is older and doesn't include it explicitly.
**Fix:** Add the missing include:
```bash
sed -i '1i#include <cstdint>' ~/P0267_RefImpl/P0267_RefImpl/P0267_RefImpl/xinterchangebuffer.cpp
```
Then re-run `make -j$(nproc)`.

### Error 5: `Compatibility with CMake < 3.5 has been removed`
**Cause:** Newer versions of CMake (4.x) require a minimum version policy flag for older projects.
**Fix:** Add the policy flag when running cmake:
```bash
cmake -DCMAKE_POLICY_VERSION_MINIMUM=3.5 ..
```

### Error 6: `'apt' not found` in WSL
**Cause:** Your default WSL distribution is not Ubuntu (likely Docker Desktop's minimal system).
**Fix:** Install Ubuntu explicitly:
```cmd
wsl --install -d Ubuntu
```
Then open Ubuntu from the Start Menu or run `wsl -d Ubuntu`.

### Error 7: `Failed to read.` (at runtime)
**Cause:** The program cannot find the map file. By default, it looks for `../map.osm` relative to the `build/` directory.
**Fix:** Make sure you run the program from inside the `build/` directory:
```bash
cd ~/cppND_Route_planning/build
./OSM_A_star_search
```

### Error 8: No GUI window appears
**Cause:** X11 display server is not available in WSL.
**Fix (Windows 11):** WSLg should work automatically. Try restarting WSL:
```cmd
wsl --shutdown
wsl -d Ubuntu
```
**Fix (Windows 10):** Install an X server like [VcXsrv](https://sourceforge.net/projects/vcxsrv/), launch it, then in WSL run:
```bash
export DISPLAY=:0
./OSM_A_star_search
```

---

## 🧪 Running Unit Tests

The project includes unit tests using Google Test to verify individual A\* components.

```bash
cd ~/cppND_Route_planning/build
./test
```

Expected output (all tests pass):
```
[==========] Running X tests from Y test cases.
[----------] Global test environment set-up.
...
[  PASSED  ] X tests.
```

---

## 📚 References

- [A\* Search Algorithm — Wikipedia](https://en.wikipedia.org/wiki/A*_search_algorithm)
- [OpenStreetMap](https://www.openstreetmap.org/)
- [IO2D Graphics Library (P0267_RefImpl)](https://github.com/cpp-io2d/P0267_RefImpl)
- [Udacity C++ Nanodegree](https://www.udacity.com/course/c-plus-plus-nanodegree--nd213)
