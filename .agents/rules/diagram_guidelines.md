---
description: "Mandatory rules and guidelines for authoring, evaluating, and compiling technical architecture diagrams (.svg + .d2) across curriculum modules"
globs: "**/*.md, **/assets/**/*.svg, **/assets/**/*.d2"
---

# Architecture Diagram Rules & Guidelines

When authoring, auditing, or updating visual architecture diagrams in this repository, you MUST adhere to the following standards:

1. **Evaluate Concept Accuracy First:**
   - Do NOT blindly convert legacy code. Evaluate whether the existing diagram is technically sufficient.
   - If an existing diagram is incomplete, oversimplified, or missing critical semantics (feedback loops, cycle guards, accurate join cardinalities, buffer pool mechanics, or proper decision tree terminals), modify and enhance it to reflect true production internals.

2. **Standalone Vector Graphics (`.svg` + `.d2`):**
   - Save compiled vector graphics as `.svg` paired with companion `.d2` source files in `<folder>/assets/<document_name>/`.
   - Never leave raw Mermaid or D2 code blocks in markdown documentation.

3. **Typography & Alignment:**
   - Container, stage, and card headers/titles: `text-anchor="middle"` at horizontal center $x = \text{card\_x} + \frac{w}{2}$.
   - Body bullets, descriptions, and code: `text-anchor="start"` with uniform 16px internal padding.
   - Headers at $y \le 54$ must have at least 35–40px clearance; all diagram content starts at $y \ge 85\text{--}95$.

4. **Dynamic Heights & Runway Padding:**
   - Containers and cards must tightly hug their contents (no dead vertical voids).
   - Maintain 80–95px runway gaps between parallel columns so connector labels (`SQL Query`, `Physical Plan`, `date_id [FK]`) are never squeezed.

5. **Connector & Arrow Integrity (Curved vs. Orthogonal):**
   - Connector marker colors must match line strokes (`#arr-b` Blue, `#arr-g` Emerald, `#arr-a` Amber, `#arr-p` Purple, `#arr-r` Red, `#arr-s` Slate).
   - Use straight lines ONLY when purely horizontal (`dy = 0`) or purely vertical (`dx = 0`).
   - Whenever an edge connects points with different vertical and horizontal coordinates ($\Delta x \neq 0$ and $\Delta y \neq 0$), **MANDATORY**: use smooth curved Bézier arrows with straight terminal runways (`M x1 y1 L rx1 ry1 C ... rx2 ry2 L x2 y2` or rounded conduits). Never draw harsh slanted diagonal straight lines.
   - **Universal Straight Runway Rule (MANDATORY for Centered Arrowheads):** Because SVG markers have non-zero length ($5.4\text{--}8\text{px}$), curving directly into $(x_2, y_2)$ forces the line to enter the arrowhead marker at an angle with a $2\text{--}4\text{px}$ cross-track error, making the arrowhead look skewed or uncentered. Every connector MUST terminate with a **straight arrival runway** of length $L_{\text{in}} \ge 10\text{--}14\text{px}$ along the edge normal vector $\vec{v}_{\text{end}}$ ($P_{\text{runway}} = P_3 - L_{\text{in}} \cdot \vec{v}_{\text{end}}$ followed by `L x2 y2`), guaranteeing that the marker sits on a 100% straight line with $0.000\text{px}$ offset and zero angular tilt.
   - **Arrowhead Tangent & Perpendicular Alignment (CRITICAL):** Arrowheads must enter card boundaries flush and strictly perpendicular to the target edge. Never use spatial delta ratios to dictate curve tangents. The physical target edge dictates terminal control point $P_2 = (cx_2, cy_2)$:
     - **Entering TOP edge:** $cx_2 = x_2, cy_2 < y_2 \implies$ Arrowhead points **STRAIGHT DOWN** ($+90^\circ$).
     - **Entering BOTTOM edge:** $cx_2 = x_2, cy_2 > y_2 \implies$ Arrowhead points **STRAIGHT UP** ($-90^\circ$).
     - **Entering LEFT edge:** $cy_2 = y_2, cx_2 < x_2 \implies$ Arrowhead points **STRAIGHT RIGHT** ($0^\circ$).
     - **Entering RIGHT edge:** $cy_2 = y_2, cx_2 > x_2 \implies$ Arrowhead points **STRAIGHT LEFT** ($180^\circ$).
     Arrowheads must NEVER slide sideways along a card perimeter or point left/right when conceptually entering downwards.
   - **Zero Overlap:** Connectors must NEVER cross over or cut through an intermediate node or container. Reorganize layout or route curves through clear gutters.
   - **Bipartite Matrix Exception:** All-to-all crossbar matrix connections (e.g. wide shuffle matrix) may use straight rays for crossbar readability.
   - Zero broken stubs: every decision path must lead to a labeled terminal card; iterative processes must have closed feedback loopback arrows.

6. **Visual Cues & Cognitive Anchors:**
   - Use subtle visual representations (Venn sets, memory buffers, timeline nodes, state pills, warning/success banners) to make diagrams understandable at a glance without clutter.

7. **Markdown Formatting:**
   - Always enforce an explicit blank line between `![img](...)` and `*D2 Source:*`.
   - Validate all SVGs for valid XML (`&amp;`, `&lt;`, `&gt;`).

