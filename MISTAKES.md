# Mistakes log (Blender 5.1/5.2 · Cycles Metal · M4 Max)

This is the file Claude appends to each time something goes wrong. Each entry follows the same pattern: what it looked like, what the cause turned out to be, and what fixed it. Translated from Vietnamese and lightly condensed.

## Performance / workflow

- **One iteration took 20–50 min and only one CPU core was busy.** `sample <pid>` showed the time going into `BKE_mesh_wrapper_ensure_subdivision`, which is OpenSubdiv building its StencilTable. The cause was a SUBSURF modifier on Rock Master meshes. Fix: remove SUBSURF. For smoothing, use Geometry Nodes instead: Subdivide Mesh (linear), then Blur Attribute on position ×4, then Set Position with 2–4 cm noise along the normal. This runs multithreaded and finishes in seconds.
- **Fast iterations.** The builder only runs the steps being inspected, at 25% resolution and 32–64 spp. For small areas, use border renders.
- **Region render and stitching.** Use `border` with `use_crop_to_border=False`, the same .blend and the same seed. Blend the seams over 48 px and measure the difference along the seam; it should stay under 3/255.
- **Installing an extension wiped every enabled add-on.** `blender -b --factory-startup --command extension install-file` saves the factory session's userpref.blend. Drop `--factory-startup` and back up userpref.blend before installing.
- **Waiting for a background render.** Wait for the output file to appear, not for "Error" in the log, because add-on errors pollute the log. Use `kill -0 <pid>`, not `pgrep -f <string>`, which matches the wait loop's own command line.

## Materials / geometry

- **Rocks looked waxy, plaster-white and reflected the blue sky.** Rock Master ships with roughness 0.16. Fix: 0.85, highlight 0.25.
- **Rocks "melted like wax" with dark round pits.** Smooth shading plus Catmull-Clark on concave meshes. Don't round rocks off; pick scanned assets that already have grain.
- **Distant cliffs looked ghostly pale.** Same 0.16 roughness, plus haze. Objects beyond ~700 m behind thick haze can be removed entirely.
- **Scanned rock was paper-thin at the edges and black underneath.** Some scans are open shells (one had about 800 boundary edges). Check `bmesh e.is_boundary` first and only use closed meshes in the foreground.
- **Crystal-like vertical spikes on rocks.** Two causes: scale.z was stretched to bury the base, and a 0.2 m snow displacement ran on an unsubdivided mesh. Fix: scale uniformly (≤ ×1.6) and colour the snow by geometric normal only.
- **Adaptive subdivision on hundreds of rocks used 74 GB of RAM and hung.** Each object is diced separately. Only use adaptive subdivision on a few objects under 15 m.
- **"Nothing real has perfectly cut edges."** Straight boolean cuts, flat facets and ruler-straight cliff tops all read as CG. Bevel and break every edge.

## Light / colour

- **The sun ended up behind the camera.** Sky Texture `sun_rotation` is measured clockwise from +Y.
- **Orange fire rendered pink or cauliflower-white.** AgX desaturates and shifts hue in highlights. For fire scenes, use Filmic with High Contrast.
- **Alpenglow pink looked like dull salmon under AgX Punchy.** Use Filmic with Medium High Contrast.
- **Exposure and white balance changes did nothing** (exposure −2 and +2 gave identical renders). The Photographer add-on overrides them at render time. Set exposure through the add-on (`ev = 9.416 − exposure`) and do white balance as channel gains in the compositor.
- **A low backlit sun barely lights the ground** (sun at 4.4° in front of the camera puts only about 8% direct light on the ground). Blocking the sun changes almost nothing; reduce sky fill instead.
- **CG crops were too saturated compared with real drone photos** (0.53 vs 0.32) and had compressed highlights. Fix: AgX High Contrast plus a compositor desaturation.

## Volumes / atmosphere

- **VDB was 2–3× the wrong size.** The bbox comes from the .blend until the grid is loaded. Set the filepath, call `grids.load()`, then `view_layer.update()`, and only then measure.
- **Stair-step banding in fire at 4K.** VDB voxels were magnified 2–3×. Fix: `volume_interpolation = 'CUBIC'`.
- **Haze tinted the whole sky.** Use roughly 0.00015·exp(−z/600) with a low ceiling (~800 m). Valley fog works at σ 0.003–0.045 m⁻¹ with a height falloff and noise holes.
- **A 260 km fog box made a bright horizontal band above the horizon.** Limit it to about 40 km.
- **EEVEE world volume made the frame pitch black.** Use a bounded volume box around the scene and camera.

## Scattering (Geometry Nodes)

- **Blender 5.2 Random Value has a single socket.** 5.1 code that uses `inputs[2]` or `outputs[2]` throws IndexError.
- **Debris scattered 1.5 km away looked like flies.** At that distance 3–8 m chunks are 5–10 px; remove them.
- **Scattering rocks and bushes on distant real-terrain mountains made them worse.** Anything under 1 px at 3+ km turns into dot clusters.
- **Scattering on an object that already has a tree-scatter modifier turned the leaves lime-yellow.** Scatter on a separate object.

## Terrain from real data

- **Norway (fjord).** AWS Terrarium DEM gave rounded mountains because the source is about 30 m interpolated. Kartverket's 5 m NHM DTM (WCS) fixed it.
- **Distant peaks 3–5 km away looked waxy and crumpled-paper.** On steep cliffs the lidar is flat TIN faces, and the 45 m scanned texture tile repeats every 22 px at 4.5 km. Fix: generate a 2 m geology map in numpy (erosion gullies along the slope, snow by slope and curvature) and use it beyond 600 m.
- **Snow stuck to vertical cliffs like paint.** Slope was computed before the vertical exaggeration was applied.
- **Snow patches looked like cow hide or camouflage.** Large-scale noise decided the boundary. Expose rock along the ribs between gullies instead.
- **Grand Canyon.** USGS 3DEP 1 m lidar. When downsampling, interpolate with a validity-mask weight, otherwise the −999999 nodata value creates 40 km holes.
- **The camera ended up standing on the cliff face** (the 1 m DEM rises 15 m in 4 m). Search for a standing spot where the 5×5 max slope is under 18° and everything 40 m ahead is lower.
- **Distant canyon walls looked smooth at 0.3–5 km.** Colour changes, aerial imagery and scattering did not help. What worked was real displacement with Blender 5.2 adaptive subdivision, using a scanned cliff displacement map with box projection, a 10 m tile near and 45 m far, and amplitude growing with distance.
- **Height-based sine bump turned into contour lines.** Avoid any periodic procedural pattern.
- **An erosion add-on (A.N.T. Eroder/Strata) did nothing on real lidar**, since real terrain is already eroded. Strata produced jagged contour steps.
- **NAIP aerial imagery has baked-in morning shadows.** Use only its low-frequency colour.

## Water / boats (Mekong delta)

- **Boolean-cutting boat hulls out of an open water plane failed.** EXACT deleted most of the water and FLOAT cut almost nothing. Also, a newly created object's `matrix_world` is identity until `view_layer.update()`. Robust fix: rebuild the water grid and triangulate the cells containing boats with holes, using `tessellate_polygon`.
- **Boat wakes made from two flat strips looked like straight searchlight beams.** Fix: a physical Kelvin wake field turned into a normal map and projected in world space inside the water shader.
- **Distant water looked "smooth like a shader".** Perlin noise reads as procedural. Fix: FFT waves with a Phillips spectrum plus slow flow streaks that follow distance-to-bank. A structured sky matters too.
- **Procedural coconut palms were all normalised to 1 m and then scaled**, which turned young palms into giant ferns. Keep real sizes. Fronds UV-mapped onto a scanned palm-leaf atlas.

## Self-checks

- **Linter** (`kiem_canh.py`): finds floating objects, constant roughness and sRGB normal maps. Later versions also check that floating objects have water under them and that people and props touch a support surface (raycast including instances). It found 87 problems in the rainforest and caught houses overhanging the canal, boats placed on the bank, and people floating mid-air or sunk 40 cm into the floor.
- **Blind crop test.** A separate model guesses whether a crop is a real photo or a render. Results: rainforest 0/18 fooled, then 60 crops at 256/512/1024 px all 100% correct; Mekong delta 15 rounds plus a second judge, 0% fooled, with confidence stuck at 0.82–0.90. Real photos were identified by "signs of life" (power lines, vehicles, rust, tarps, foam on wakes, lens flare) and by messiness. Renders were caught by repeated houses, palms and boats, clean surfaces, flat field patches and gradient fog. Small parameter tweaks never moved the score; only changes of *kind* did.
- **Film grain, bloom and chromatic aberration did nothing** while the scene was still far from real.

## Making these clay renders

- **The first clay pass used Workbench and froze the whole Mac three times** (GPU hang, WindowServer killed by the watchdog, only a reboot fixed it). The valley scene has about 2.2 million scattered instances, roughly 57 billion triangles if every instance is drawn. Cycles handles that through BVH instancing; Workbench rasterises every triangle, and with 8× anti-aliasing that is one enormous GPU job. Fix: clay in Cycles with a grey material override (`view_layer.material_override`), 32 samples plus denoising.
- **Clay renders came out blown white or pitch black.** The Photographer add-on re-applies the camera's EV as scene exposure on every depsgraph update, and the new pulled-back camera got the add-on's default EV. Fix: turn off its exposure on every camera and reset exposure to 0.
- **Pulled-back camera rendered black in the fjord.** Stepping back 150 m put it 51 m inside a cliff. Fix: raycast the terrain under the new position and lift the camera above it.
