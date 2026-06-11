---
type: concept
domain: SWE Foundations
status: drafted
last-reviewed: 2026-05-27
tags: [git, cmake, build-systems, colcon, ament, devtools]
---

# Git & Build Systems

> [!question] Explain it cold
> *Before reading anything below, say or write the answer from memory. This is the rep that matters — the reference section is just the answer key.*
>
> - What is Git's storage model, in two sentences?
> - When would you rebase vs. merge?
> - What does CMake actually output, and what runs it?

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

## Where I've used it

- **Articulus Surgical**: multi-package ROS2 workspace with `colcon`; pinned vendor kinematics lib as a git submodule so all devs built the same SHA. Had to debug a `find_package` failure caused by an out-of-order `ament_target_dependencies` call.
- **WALL-E V3**: CMake cross-compile toolchain file targeting ARM Cortex-A; used `CMAKE_TOOLCHAIN_FILE` and `CMAKE_SYSROOT` to avoid host/target library mixup.
- **FENCE-BOT**: used `git bisect` to find the commit that broke encoder interrupt timing after a refactor of the AVR peripheral layer.

---

## Interview follow-ups

- **Q: What's the difference between `git merge` and `git rebase` — when would you use each?**
  - **A:** Merge preserves branch topology and adds a merge commit; rebase replays commits linearly. I rebase feature branches before opening a PR (cleaner history, easier review), but never rebase commits already pushed to a shared branch — it rewrites SHAs and forces everyone else to `reset --hard`.

- **Q: How does CMake's `target_link_libraries` with `PRIVATE/PUBLIC/INTERFACE` work?**
  - **A:** `PRIVATE` = dep used only in this target's `.cpp`, not exposed in headers. `PUBLIC` = used in both implementation *and* headers (callers inherit the dep). `INTERFACE` = only in headers, not in `.cpp` (header-only libs). Getting this wrong causes either missing includes for downstream targets or unnecessary recompilation.

- **Q: Your ROS2 node compiles but crashes at runtime with "cannot open shared object file". Why?**
  - **A:** The installed `.so` isn't on `LD_LIBRARY_PATH` — usually means you forgot to `source install/setup.bash`, or the library was installed to a non-standard prefix not known to the dynamic linker. Fix: source the overlay, or add an `-rpath` to the CMake target, or add the path to `/etc/ld.so.conf` and run `ldconfig`.

- **Q: How do you handle a conflict during `git rebase`?**
  - **A:** Git pauses at each conflicting commit. Resolve the conflict in the file, `git add` the resolved file, then `git rebase --continue`. If it's a mess, `git rebase --abort` to return to pre-rebase state and think again.

- **Q: What's `cmake --build` vs. just running `make`?**
  - **A:** `cmake --build` is generator-agnostic — it works whether the underlying system is Make, Ninja, or MSVC. Running `make` directly couples you to the Makefile generator. CI scripts should always use `cmake --build`.

- **Q: How does `git cherry-pick` differ from `git rebase`?**
  - **A:** Cherry-pick copies a single specific commit onto HEAD. Rebase moves a *range* of commits onto a new base, rewriting the entire chain. Cherry-pick is surgical; rebase is structural.

---

## Gotchas / what trips me up

- Forgetting `source install/setup.bash` after a `colcon build` — the old overlay is stale and you get wrong library versions silently.
- `ament_target_dependencies` vs `target_link_libraries` — they're not equivalent. `ament_target_dependencies` also handles `ament_index` exports; missing it breaks downstream `find_package`.
- `git rebase` on a shared branch — rewrites SHAs, causes diverged history for teammates. Only rebase unpublished commits.
- CMake cache poisoning: old `CMakeCache.txt` after changing a toolchain. Fix: delete the build directory entirely, don't just re-run cmake.
- `git stash pop` on a dirty tree can silently merge stash changes with current edits. Prefer `git stash apply` + manual verify before `git stash drop`.

---

## Links
- Related: [[colcon ament & launch]], [[Docker for ROS]], [[Modern C++ for Robotics]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What are the four Git object types?
?
blob (file contents), tree (directory), commit (tree + parent + metadata), tag (annotated pointer to commit).

What does `git reset --soft HEAD~1` do vs `--hard`?
?
`--soft` undoes the commit but keeps changes staged. `--hard` discards both the commit and all working-tree changes.

When should you NOT rebase?
?
Never rebase commits already pushed to a shared/remote branch — it rewrites SHAs and forces teammates to reset.

What is the CMake configure step vs the build step?
?
Configure: reads CMakeLists.txt and generates Makefiles/Ninja files. Build: executes the generated system to compile and link.

What does `target_link_libraries(foo PUBLIC bar)` mean vs PRIVATE?
?
PUBLIC: bar is needed in both foo's implementation and its headers — callers inherit bar. PRIVATE: bar is only in foo's .cpp, not exposed to callers.

What does `colcon build --symlink-install` do?
?
Installs Python scripts and launch files as symlinks into install/ so edits take effect immediately without rebuilding.

How do you find which commit introduced a bug in git?
?
`git bisect start`, mark HEAD as bad, mark a known-good commit as good — git binary-searches through commits for you.

What causes "cannot open shared object file" at ROS2 runtime?
?
The install overlay wasn't sourced (`source install/setup.bash`), so LD_LIBRARY_PATH doesn't include the package's lib directory.

How do you stage specific files safely (instead of `git add .`)?
?
`git add path/to/file.cpp` — staging by name prevents accidentally committing secrets, binaries, or unrelated changes. Use `git status` first to see what's dirty.

How do you undo a staged file without losing your changes?
?
`git restore --staged file.cpp` — removes it from the staging area but keeps the edits in the working tree.

What's the difference between `git pull` and `git pull --rebase`?
?
`git pull` merges upstream into your branch (adds a merge commit). `git pull --rebase` replays your local commits on top of upstream — cleaner history, no merge commit noise.

Snap vs apt — key difference?
?
apt installs `.deb` packages that share host system libraries. Snap bundles all dependencies and runs in a sandbox. Snaps auto-update; apt requires `apt upgrade`.

Why might a snap fail to read files in your home directory?
?
Snaps are sandboxed and need the `home` interface connected: `sudo snap connect <snap>:home :home`. Without it, the snap can't see `/home` by default.
