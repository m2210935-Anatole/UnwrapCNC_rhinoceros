# UnwrapCNC

**Compact 2D cutting layouts from solid sheet-material parts in Rhino 9.**

UnwrapCNC identifies parts by material thickness, extracts planar face boundaries, and packs them onto rectangular sheets. Parallel heuristic search explores different part orders and rotations, with optional CUDA acceleration for collision and clearance checks.
<img width="1434" height="736" alt="image" src="https://github.com/user-attachments/assets/273cc699-48bf-4626-922b-a61ab7de4d59" />

The interface supports **English and Russian**, with a saved language preference.
<img width="494" height="422" alt="image" src="https://github.com/user-attachments/assets/1d33ebdf-6733-40e2-b7b3-2c80fd5db95c" />


## Features

- Detect parts matching a specified material thickness and tolerance.
- Extract outer and inner contours from a planar face or its opposite face.
- Search for compact layouts using multiple part orders and rotations.
- Place parts in concave free space around other parts.
- Use simplified proxy contours during search and Rhino geometry for final validation.
- Run concurrent CPU search with optional NVIDIA CUDA acceleration.
- Progressively reduce sheet width and height when all parts fit on one sheet.
- Save packing settings, including language, rotation step, and search time.
- Create separate layers for cutting contours and sheet frames.

## Requirements

| Component | Requirement |
|---|---|
| Application | Rhino 9 for Windows, running with .NET 10 |
| Input | Valid closed polysurfaces or solid extrusions with suitable opposing planar faces |
| Optional GPU | A compatible NVIDIA GPU and CUDA driver |
| Source build | .NET 10 SDK and an installed copy of Rhino 9 |

The CUDA kernel is embedded in the plugin. A CUDA Toolkit installation is required only when rebuilding the kernel. CPU packing is available when CUDA initialization is unavailable.

## Installation

1. Save `UnwrapCNC.rhp` in a permanent folder.
2. Open Rhino and run `_PlugInManager`.
3. Choose **Install** and select `UnwrapCNC.rhp`.
4. Run `UnwrapCNC`.

### Updating an existing installation

Close Rhino and replace the registered `.rhp` file, preserving its existing filename and location. Reopen Rhino and run `UnwrapCNC`.

Earlier versions may be registered as `CncNestRecovery.rhp`. In that case, rename the downloaded file to `CncNestRecovery.rhp` before replacing it. The command remains `UnwrapCNC`.

If Rhino reports that the plugin ID is already in use, locate the existing registration in `_PlugInManager` and update the file at that registered location.

## Quick start

1. Run `UnwrapCNC`.
2. Choose **Русский** or **English** from **Язык / Language**.
3. Enter the material thickness and tolerance.
4. Set the sheet dimensions, part spacing, and sheet margin.
5. Choose the rotation step, number of attempts, and search time.
6. Enable **Use GPU (CUDA)** if desired.
7. Click **Select parts**, select the solids in Rhino, and press **Enter**.
8. Inspect the generated contours on the **World XY** plane.

All dimensions use the current Rhino document units. Saved dimensional settings are converted when used in a document with different units. Object selection is requested for every run.

## Settings

| Setting | Purpose |
|---|---|
| Material thickness | Target thickness measured along a candidate face normal across the whole solid |
| Thickness tolerance ± | Allowed deviation from the target thickness, with Rhino's document tolerance also used during detection |
| Sheet width / Sheet height | Initial sheet dimensions and maximum dimensions for the layout |
| Part spacing | Requested minimum distance between neighboring outer contours |
| Sheet margin | Clearance from the sheet boundary |
| Packing attempts | Maximum attempts per tested sheet size, from 1 to 128 |
| Rotation step | Search increments of 5°, 15°, 30°, or 90° |
| Search time | Shared time budget for packing and sheet reduction |
| Use GPU (CUDA) | Enable GPU collision and clearance checks alongside CPU search |
| Use opposite face | Extract loops from the opposite face of the detected pair |
| Язык / Language | Switch the window, prompts, and command messages between Russian and English |

The default rotation step is **5°**. Larger steps consider fewer orientations and usually reduce computation time. When a straight boundary segment is available, the search starts from an orientation aligned with the longest such segment.

The default search budget is **10 seconds**. Set it to **0** to disable the time budget while retaining the attempt limit. Geometry preparation, CUDA initialization, and final Rhino validation take additional time. The first complete layout is allowed to finish even if the search budget expires.

Language changes apply immediately. Packing parameters are saved when **Select parts** is pressed.

## How packing works

1. **Detect suitable faces.** The plugin checks solids for opposing planar faces at the requested thickness. The largest suitable face is considered first.
2. **Extract contours.** The chosen face's outer and inner boundary loops are transformed onto World XY.
3. **Prepare proxies.** Simplified polygons and their rotated variants are prepared for the search.
4. **Explore layouts.** Workers vary part order and test positions. The search favors fewer sheets, then a smaller occupied bounding area.
5. **Reduce the sheet.** For a single-sheet result, empty outer strips are removed and width and height are reduced in steps of 5%, 2%, and 1%. Successful reductions retain a complete, validated layout.
6. **Validate and write.** Rhino checks the original curves and writes cutting contours and sheet frames.

CUDA checks polygon intersections, containment, and spacing in batches. Geometry and GPU buffers are reused throughout the search, including sheet reduction.

The search is heuristic. Packing quality and runtime depend on part shape, count, rotation step, spacing, and the search budget.

## Output

Each run creates layers with a unique prefix.

| Layer | Contents |
|---|---|
| `CNC_<id>_Cut` | Outer and inner cutting contours |
| `CNC_<id>_Sheets` | Rectangular sheet frames |

Cutting curves carry `SourceId` and `Sheet` user strings to identify their source object and destination sheet. Curve names distinguish `Outer` and `Hole`.

Select the cutting curves and use Rhino's **Export Selected** to export a format supported by your CAM workflow, such as DXF. Toolpaths, cutter compensation, machining order, and machine code are prepared in the downstream CAM application.

The command output reports skipped objects, search information, sheet dimensions, occupied area, and repairs made during exact validation.

## Current scope and limitations

- The input workflow targets solid sheet-material parts with opposing planar faces.
- Each extracted side is represented by one Rhino face. Sides split into multiple coplanar faces may need preparation before extraction.
- Pockets and chamfers are represented only through the boundary loops of the chosen face. Review these loops when defining machining operations.
- Inner loops are exported, while nesting uses the outer contour. Placing parts inside holes is outside the current search.
- Rotation settings apply globally. Grain direction and per-part orientation constraints require manual handling.
- Automatic sheet reduction runs only when the initial result uses one sheet.
- If the final exact check finds a conflicting placement, that part is moved to a separate sheet. The command reports the repair count.
- GPU performance varies by workload. Small or simple jobs can favor CPU execution.

## Validation status

The packing core has been exercised on synthetic rectangular and concave-part datasets. Checks cover part counts, sheet bounds, clearances, CPU/CUDA agreement for sampled placements, and retaining a complete result when the time budget expires.

The version 4.1 assembly passed the Rhino compatibility check. Its bilingual interface has not yet been verified interactively inside Rhino on a user model.

## Build from source

Place the source files, `UnwrapCNC.csproj`, and `nest.ptx` in the same directory. Build with the .NET 10 SDK.

```powershell
dotnet build UnwrapCNC.csproj -c Release
```

The resulting plugin is written to `bin/Release/net10.0/UnwrapCNC.rhp`.

The project uses Rhino assemblies from `C:\Program Files\Rhino 9\System` by default. Override the installation path when necessary.

```powershell
dotnet build UnwrapCNC.csproj -c Release -p:RhinoSystem="D:\Applications\Rhino 9\System"
```

### Source layout

| File | Responsibility |
|---|---|
| `UnwrapCNC.cs` | Rhino command, bilingual interface, extraction, exact validation, and output |
| `FastNest.cs` | Candidate positions, polygon collision checks, and packing |
| `HeuristicSearch.cs` | Parallel attempts, best result, time budget, and GPU session lifetime |
| `ShrinkCanvas.cs` | Progressive sheet-size reduction |
| `ProxyGeometry.cs` | Simplification of search contours |
| `CudaContours.cs` | CUDA driver integration and reusable GPU buffers |
| `nest.cu` | CUDA collision kernel source |
| `nest.ptx` | Compiled kernel embedded in the plugin |
| `UnwrapCNC.csproj` | Build configuration and Rhino references |

## Reporting an issue

Include the Rhino version, plugin version, document units and tolerance, packing settings, and relevant command output. For a geometry issue, attach a small reproducible `.3dm` example when possible. For a CUDA issue, also include the GPU model and driver version.
