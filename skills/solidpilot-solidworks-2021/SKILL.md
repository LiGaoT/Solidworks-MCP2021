---
name: solidpilot-solidworks-2021
description: Use a SolidWorks MCP server to create, edit, and verify CAD parts, assemblies, and drawings in SolidWorks 2021. Apply when a user asks for SolidWorks MCP modeling or a 2021 compatibility check; requires a running MCP adapter and SolidWorks 2021.
---

# SolidWorks MCP modeling on 2021

Use the available SolidWorks MCP tools to build editable feature trees. Confirm that the execution server and SolidWorks 2021 are connected before changing documents. Identify the installed SolidWorks revision, language, and template paths because these affect COM calls and standard-view names.

## Interpret the drawing first

- Read every supplied view and dimension. Record dimensions that are explicit, derived, and unresolved separately. Do not turn an unresolved manufacturing dimension into an unmarked assumption.
- Choose a global coordinate system and map each orthographic view to it before creating a sketch. Check hole axes, front/top/right orientation, handedness, and projection standard.
- Keep separate manufactured components as separate parts when the drawing describes a welded or assembled structure. Preserve hole groups, slots, counterbores, countersinks, fillets, chamfers, and welds as distinct features.
- Use the tool contract and the active `recipe://usage/*` resources (or `get_recipe` when the client cannot read resources) for parameters and supported feature vocabulary.

## Build and verify

1. Call `ensure_ready`, then `open_new_part` or `open_new_assembly`. Use the correct part, assembly, and drawing templates for the local installation.
2. Build the main solid with `create_sketch`, `add_sketch_entity`, and `extrude_feature`. All length inputs to the SolidPilot execution layer are in **meters**; convert drawing dimensions in mm explicitly.
3. Create holes and slots from their true center coordinates. For repeated groups, derive each location from the dimension chain and check the resulting count and spacing with `analyze_model`.
4. Add edge features after the underlying body is stable. Query `analyze_model(analysis_type='edges')` immediately before `add_edge_feature`; select by `edge_indices` and re-query after topology changes. A C10 chamfer is distinct from an R10 fillet or a weld size a10.
5. Use `analyze_model` (`geometry`, `mass_properties`, `features`, and targeted `faces`/`edges`) and `verify_state` to check each meaningful stage. A successful tool response alone is not proof that the intended geometry was created.
6. Save to `.sldprt` or `.sldasm`, confirm that the file exists and is nonempty, close it, reopen it, and read geometry again. For an assembly, inspect components and their full transforms with `analyze_assembly`.
7. When needed, create drawing views from a saved model, export a PDF, and inspect the actual views. Compare them to the source drawing before declaring the model complete.

The execution REST API uses a unique `operationId` and current `stateVersion` for each operation. A caller using REST directly should read `GET /api/tool/state` before `POST /api/tool/execute`. An MCP client should use the adapter's tools and let it manage this state.

## SolidWorks 2021 compatibility

- Build the C# execution layer against **2021** `SolidWorks.Interop.*` assemblies from that installation's `api/redist` directory. Do not run a binary compiled against a different SolidWorks major version without rebuilding and testing.
- SolidWorks 2021 interop does not expose `IBaseFlangeFeatureData.SymmetricThickness`. If symmetric base-flange thickness is requested, report it as unsupported unless the running API demonstrates an equivalent operation; do not silently omit it.
- `CreateDrawViewFromModelView3` can return null for English standard-view names on a Chinese installation. Try the localized standard-view name; for front view, `*前视` was confirmed on SolidWorks 2021 SP0 Chinese. Validate each other orientation on the actual installation.
- Treat advanced sheet-metal, mates, drafting, and editable topology references as pilot until verified on the target machine. Record exact errors and stop the dependent feature chain when an operation fails.

## Evidence and handoff

Report the created files, feature count, body count, relevant dimensions, mass or volume, and any unresolved assumptions. Keep the original drawing available for comparison. Manufacturing-ready claims require checking hole tolerances, threads, welds, material, and geometric tolerances independently of the solid model.
