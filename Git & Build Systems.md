---
type: concept
domain: SWE Foundations
status: drafted
last-reviewed: 2026-05-27
tags: [git, cmake, build-systems, colcon, ament, devtools]
---

# Git & Build Systems


---

## Core idea

Git stores a **directed acyclic graph of snapshots** (not diffs); every commit is a SHA-1 pointer to a full tree. Build systems (CMake, Make, colcon) translate a high-level description of *what to compile and link* into a dependency graph, then execute only the stale subgraph.

---

## Key facts & formulas

### Git internals

| Object | What it stores |
|--------|---------------|
| **blob** | file contents (no name) |
| **tree** | directory listing → blob SHAs |
| **commit** | tree SHA + parent SHA(s) + metadata |
| **tag** | pointer to a commit + annotation |

- `HEAD` → branch ref → commit SHA. Detached HEAD = HEAD points to a commit directly.
- **Rebase** rewrites commits onto a new base (linear history, clean `git log`). **Merge** creates a merge commit (preserves topology). Use rebase for personal branches before PR; use merge for integrating mainline into a long-lived branch.
- `git reflog` is your undo button — it logs every HEAD move for 90 days.

### Key commands under pressure

```bash
# Undo last commit but keep changes staged
git reset --soft HEAD~1

# Undo last commit and unstage
git reset --mixed HEAD~1   # (default)

# Nuclear undo — discard working tree too
git reset --hard HEAD~1

# Cherry-pick one commit onto current branch
git cherry-pick <sha>

# Interactive rebase: squash/reword last 3 commits
git rebase -i HEAD~3

# Find which commit introduced a bug (binary search)
git bisect start
git bisect bad HEAD
git bisect good <known-good-sha>

# Stash including untracked files
git stash push -u -m "wip: sensor fusion refactor"

# See what's in a stash without applying
git stash show -p stash@{0}
```

### Daily git workflow

```bash
# See what's changed
git status
git diff                        # unstaged changes
git diff --staged               # staged changes

# Stage and commit
git add path/to/file.cpp        # stage specific files (safer than git add .)
git commit -m "Add feature X"

# Push / pull
git push
git pull --rebase               # rebase local commits on top of upstream (cleaner than merge)

# Check history
git log --oneline -10
git log --oneline --graph --all  # visualize branches

# Undo staged file (keep changes in working tree)
git restore --staged file.cpp

# See what a commit changed
git show <sha>
```

### .gitignore quick-ref

```gitignore
# Compiled ELF binaries (no extension on Linux)
my_binary
build/

# Object files and archives
*.o
*.a
*.so

# CMake build artifacts
CMakeCache.txt
CMakeFiles/

# IDE / editor
.vscode/
*.swp

# Python bytecode
__pycache__/
*.pyc
```

**Tip:** List binaries by exact name when they live alongside source. Use `*.ext` patterns for artifact classes. Check what's being ignored with `git check-ignore -v <file>`.

### Submodules vs. subtrees
- **Submodule**: repo embeds a *pointer* to another repo at a fixed SHA. Clone needs `--recurse-submodules`. Used in ROS2 workspaces that pin vendor libs.
- **Subtree**: copies the sub-repo's history in-tree. No extra clone step but history is messier.

### CMake essentials

```cmake
cmake_minimum_required(VERSION 3.22)
project(my_robot CXX)

# Find dependencies
find_package(rclcpp REQUIRED)
find_package(Eigen3 REQUIRED)

# Build target
add_executable(controller src/controller.cpp)

# Link (modern target-based style — no include_directories globally)
target_link_libraries(controller
  PRIVATE rclcpp::rclcpp Eigen3::Eigen)

# Install for colcon
install(TARGETS controller DESTINATION lib/${PROJECT_NAME})
```

CMake two-step:
1. **Configure**: reads `CMakeLists.txt`, generates Makefiles or Ninja files.
2. **Build**: `cmake --build build/` runs the generated build system.

`BUILD_SHARED_LIBS=ON` → `.so`; off → `.a`. Set per-target with `add_library(foo STATIC ...)`.

### colcon / ament (ROS2)

```bash
# Build only changed packages + their dependents
colcon build --symlink-install --packages-up-to my_pkg

# Run tests for one package
colcon test --packages-select my_pkg
colcon test-result --verbose

# Source the overlay (always do this after build)
source install/setup.bash
```

- `ament_cmake` wraps CMake; adds `ament_target_dependencies(target rclcpp sensor_msgs)` macro which sets includes + links in one call.
- `symlink-install` means edits to Python/launch files take effect without rebuilding — critical for fast iteration.
- Build order is resolved from `package.xml` `<depend>` tags.

### Linking quick-ref

| Flag | Meaning |
|------|---------|
| `-L/path` | add library search directory |
| `-lfoo` | link `libfoo.so` or `libfoo.a` |
| `-I/path` | add include search directory |
| `-rpath` | embed runtime search path in binary |
| `ldd binary` | show runtime shared lib dependencies |

### Snap package manager

Snap is Ubuntu's containerized package system — apps ship with all dependencies bundled, isolated from the host system. Common on Ubuntu 20.04+.

```bash
# Install / remove
sudo snap install <package>
sudo snap remove <package>

# List installed snaps
snap list

# Update all snaps
sudo snap refresh

# Update a specific snap
sudo snap refresh <package>

# See available versions / channels
snap info <package>

# Install from a specific channel (e.g., edge, beta, stable)
sudo snap install <package> --channel=edge

# Connect a plug (give a snap access to a resource)
sudo snap connect <snap>:<plug> <snap>:<slot>

# See what interfaces a snap uses
snap connections <package>

# Run a snap app directly
snap run <package>
```

**Snap vs apt:**
| | apt | snap |
|---|---|---|
| Packaging | `.deb`, host libs | Self-contained bundle |
| Updates | `apt upgrade` | Auto-updated in background |
| Isolation | Shares host libs | Confined (can't see host by default) |
| ROS2 | `apt install ros-humble-*` | `snap install ros2-humble` (less common) |

**Common gotcha:** Snaps run in a confined sandbox. If a snap can't access `/home` or a USB device, it needs the right interface connected (`home`, `hardware-observe`, `raw-usb`, etc.).

```bash
# If a snap can't access files in your home directory
sudo snap connect <snap>:home :home

# List all available interfaces
snap interfaces
```

---


---


---


---

## Links
- Related: [[colcon ament & launch]], [[Docker for ROS]], [[Modern C++ for Robotics]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

