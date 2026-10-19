# Hemyrso local deviation review MVP

Status: Written specification awaiting user review. The conversational design was approved on 2026-10-19. Implementation starts after review of this specification and the implementation plan.

## Purpose and users

Build a local application for HEMY's BIM coordinator to compare an intended BIM model with a site point cloud, investigate suspected deviations, and export confirmed findings for site decisions. Imerso is a product reference. Its public descriptions do not establish its implementation or algorithms.

Success means that the coordinator can complete the workflow below with the supplied workshop data and distinguish measured evidence from uncertainty. This version assists human review. Its output is not a certification of construction quality.

## Agreed scope

- Use `workshop-scan-bim-viewer/workshop.ifc` as the initial design model.
- Use `workshop-scan-bim-viewer/workshop.e57` as the test point cloud.
- Keep `workshop-baseline.ifc` outside the initial comparison. Its intended relationship to the model has not been established.
- Process and store project data locally. Bind the application service to loopback.
- Support one active project and one comparison job at a time.
- Use assisted deviation detection with explicit alignment review and human confirmation.
- Export findings as CSV.
- Defer Microsoft Fabric connectivity, progress tracking, clash prediction, automatic BIM updates, cloud accounts, and external integrations.

## Input observations

Read-only inspection on 2026-10-19 established these facts:

| Input | Observation |
| --- | --- |
| `workshop.ifc` | 1,306,585 bytes. Header declares IFC4. Length unit is millimetres. |
| `workshop.e57` | 246,742,016 bytes. Header signature is `ASTM-E57`, version 1.0. Declared file length matches the filesystem length. |
| E57 compressed vectors | XML metadata declares two point sets, with 4,425,891 and 4,346,791 records. |
| E57 station poses | XML metadata contains a quaternion rotation and translation for each station. |

The E57 check read its header and embedded XML across page checksum boundaries. It did not verify page checksums or decode compressed point records. IFC geometry, scan validity, shared coordinates, useful coverage, and known real deviations remain unverified. These are implementation validation tasks, not assumed facts.

## User workflow

### 1. Create or reopen a local project

The coordinator selects an IFC and E57 through a local file workflow and chooses a project output directory. Source files stay unchanged. The app records file identity and stores derived data, alignment, run parameters, and reviewed findings in that directory.

The interface shows the current import stage, available progress, and errors. It must not display invented progress percentages. A failed import identifies the file and stage, preserves the source files, and allows retry.

### 2. Inspect the model and scan

Show the model and point cloud in a shared 3D view with independent visibility controls, orbit, pan, zoom, fit, and a selectable region. Selection of a model element shows its IFC GlobalId and type. A legend explains comparison colours and unassessed areas.

Use a bounded point-cloud preview rather than sending all raw point records to the browser. Show whether the preview is sampled. Keep preview sampling separate from the recorded analysis sampling policy.

### 3. Confirm alignment

Convert model lengths to a common internal unit of metres. Decode scan coordinates using the E57 reader's unit semantics. Apply each station's pose exactly once, accounting for whether the reader already applies it. Preserve the original coordinates in the source files.

Show the initial overlay and coordinate extents. Permit manual rigid alignment through at least three non-collinear corresponding points, and allow reset. Compute a proper rigid transform without scale changes or reflections. Reject degenerate control-point sets. Display residuals for the selected correspondences. Residuals are an alignment diagnostic, not a claim of independent survey accuracy.

Save the transform and require the coordinator to confirm the overlay before enabling comparison. The final viewer transform and the analysis transform must agree. Camera or renderer axis conventions must not alter the persisted geometry coordinates.

Automatic registration is outside this MVP. A later registration method may propose a transform, but manual review remains necessary until validated.

### 4. Run a bounded comparison

The coordinator selects a region and an adjustable distance threshold. Use 20 mm as an initial development threshold, visibly identified as provisional. Allow a positive threshold in millimetres. Record its exact value on each run.

Generate model triangles with a mapping to their IFC GlobalIds. Compute unsigned distances from valid, sampled scan points to model triangle surfaces in the selected region. Use a spatial index or equivalent bounded search. A distance to a nearest model vertex alone is not an acceptable substitute for distance to a triangle surface.

The comparison returns sample-level distances and the corresponding candidate model element. A point beyond the threshold is a candidate deviation, not proof that the associated element is misplaced. Clutter, unmodelled objects, occlusion, and incomplete models can cause large distances. State this interpretation in the review interface.

Exclude invalid and non-finite scan records. Record included point count, excluded point count when available, spatial sampling resolution, region, geometry settings, threshold, and alignment version. Any deterministic sampling must be reproducible and preserve a route from a displayed finding to its measured point coordinates.

Do not label an element missing because it has no nearby scan points. Unobserved model regions remain unassessed. This MVP does not claim complete element-level coverage, missing-element detection, signed deviation, or automatic construction progress.

### 5. Review and export findings

The coordinator creates a finding from a selected suspect point or region. Show the measured distance, candidate element, point location, and supporting local scan geometry. Allow notes and one of three review states: unreviewed, confirmed, or dismissed.

Use spatial aggregation to reduce visual clutter only if the underlying measured samples remain inspectable. Automatic grouping is optional for the first delivery. Manual selection must remain usable.

CSV export includes confirmed findings by default, with an explicit option to include all review states. Include finding ID, run ID, IFC GlobalId when available, element type, location in metres in the documented project coordinate frame, measured distance in millimetres, tolerance in millimetres, review state, notes, and creation time. Quote fields correctly and prevent spreadsheet formula execution from user-authored notes or labels.

### 6. Save and reopen

Persist alignment, comparison parameters, finding states, and notes. Reopening a project restores them. Replacing either source file invalidates derived geometry and comparison results. Editing alignment or comparison parameters creates a new run, with prior findings retained under the run that produced them.

## Local architecture

Use a browser interface and one local processing service. The service handles file access, IFC geometry conversion, E57 decoding, sampling, alignment, and comparison. The browser handles navigation, selection, controls, and result review. Large imports and comparisons run outside the request handler so progress and errors remain accessible.

Use established IFC and E57 readers rather than writing format decoders. Select and pin specific libraries during implementation planning after checking Windows support and their handling of transforms. Keep one processing process and one active job. Introduce a separate worker system only if an observed runtime limitation requires it.

Store projects in ordinary local directories with a versioned manifest and derived files. Use atomic manifest writes and a project write lock. Partial conversions must not be mistaken for ready outputs. Derived artifacts can be removed and rebuilt. A database and remote storage are unnecessary for this version.

The processing boundary accepts source paths and explicit parameters and returns local artifact references, summaries, and errors. This boundary allows a future Fabric connection without building that integration now.

## Minimal data model

- Project: ID, schema version, source identities, output directory, and display name.
- Geometry artifact: source identity, converter version, conversion settings, units, element mapping, and ready or failed state.
- Alignment: rigid 4-by-4 transform, control points, residuals, confirmation state, and revision.
- Comparison run: immutable source identities, alignment revision, region, threshold, sampling settings, measured samples, and processing status.
- Finding: ID, run ID, measured sample reference or explicit location, candidate element reference, review state, notes, and creation time.

The model owns the project coordinate frame. Store a cloud-to-model transform explicitly. Any renderer offset for numerical stability is derived and reversible.

## Errors and local access

Validate file types, readable paths, positive parameters, finite transforms, and output-directory writability at the service boundary. Restrict serving derived artifacts to the project output directory. Accept browser requests only from the app's own origin. Use local assets after dependency installation, with no project-data uploads or remote analytics.

Report unsupported IFC geometry and skipped entities with counts and identifiers. Fail a comparison that has no usable model surfaces or scan samples. Preserve failure details and allow retry. A restart may mark an interrupted job as interrupted and offer rerun; automatic resume of partial jobs is unnecessary.

## Acceptance criteria

1. The app imports usable geometry from the supplied IFC and decodes valid points from both declared E57 point sets. It reports unsupported geometry and rejected points.
2. The view overlays both inputs, supports selection and a bounded region, and displays their units and extents without claiming that their alignment is validated automatically.
3. A controlled alignment example recovers a known rigid transform. A degenerate correspondence set produces an actionable error. The viewer and processor use the same accepted transform.
4. A controlled plane example measures known 0 mm, 10 mm, and 40 mm offsets within 1 mm of numerical error. At a 20 mm threshold, the 40 mm sample is suspect and the 10 mm sample is within threshold. This verifies the computation, not the physical accuracy of the scanner.
5. A scan with no coverage of a model region does not create a missing-element finding. Distant clutter remains a candidate that a reviewer can dismiss.
6. At least one real workshop comparison completes for a selected region. Record its included samples, settings, duration, and observed resource limits. Real-world correctness remains provisional until HEMY supplies a verified real deviation or independent measurement.
7. Confirmed and dismissed findings, notes, alignment, and run settings survive project reopening. Stale results are not reused after source or alignment changes.
8. CSV export round-trips notes containing commas, quotes, and line breaks and preserves element identity and measurements. User-authored formula-like content opens as text in a spreadsheet.
9. The application can complete the workflow using local data and local runtime assets after installation. It reports failures without modifying source files.

## Verification and execution

Use focused behavioural checks for alignment, distance computation, invalidation, persistence, and export. Exercise the actual browser workflow with the supplied workshop data. Do not claim success from compilation or synthetic checks alone. Record limits when full-cloud processing is too costly and keep any region or sampling restriction visible to the user.

Use REA for evidence from public Imerso pages, public demos, accessible browser assets, and requested runtime behaviour. No authenticated Imerso access is available. Record observations, inferences, and unknowns separately. Ordinary source development and IFC or E57 processing use normal development tools.

Subagents inherit the active session model. The configured default is `gpt-6.1-sol`. Model selection, escalation, and review instructions in skills do not authorize a model switch. Astra requires explicit user approval for the specific scope. Include this restriction in isolated subagent briefs.

Before implementation, prepare a plan with concrete dependencies, file ownership, verification commands, and task dependencies. Assign separate subagent ownership for processing, interface, and review only after shared contracts are fixed. Keep the integration owner responsible for running the complete local workflow.

## Reference boundary

Imerso's public product description at https://www.imerso.com/product describes BIM and scan comparison, adjustable tolerances, review colours, progress tracking, and exports. Those descriptions informed the initial workflow. The distance method, local architecture, provisional threshold, and acceptance criteria above are Hemyrso design choices, not reverse-engineered claims about Imerso.
