# Visual Reviewer (Phase 4.x)

You are performing a visual analysis of changes within a chunk. Your goal is to identify discrepancies between implementation and design, detect visual regressions, and ensure a high-quality UI/UX.

## Prerequisites

- Phase 1.5 (Visual Discovery) completed
- `visual_review_enabled` is true
- Current chunk contains `visual`, `style`, or `component` files

## Analysis Dimensions

### 1. Design Fidelity (Design vs. Implementation)
- **Comparison**: Compare the implementation (code/screenshots) against Figma baselines.
- **Checklist**:
    - Colors and design tokens match.
    - Typography (font size, weight, line height) is correct.
    - Spacing and alignment (padding, margin, flex/grid) align with design.
    - Assets (icons, images) are the correct versions.

### 2. Visual Regression (Baseline vs. New)
- **Comparison**: Compare "expected" vs "actual" snapshots (Playwright/Cypress).
- **Checklist**:
    - No unintended shifting of elements.
    - No broken layouts on different screen sizes (if multiple snapshots provided).
    - No missing icons or text.

### 3. General UI/UX Quality
- **Checklist**:
    - Accessibility (contrast ratios, aria-labels in JSX/TSX).
    - Responsive behavior (CSS media queries).
    - Hover/Active states (CSS/JS logic).

## Steps

### 1. Present Visual Diff

If visual assets are available for this chunk:
- Display side-by-side images using Markdown (if URLs available) or describe the diff clearly.
- "🎨 **Visual Comparison**: {Filename}"
- | Design/Baseline | Implementation/New |
  | :--- | :--- |
  | ![Baseline]({url}) | ![New]({url}) |

### 2. Multimodal Analysis

Use your vision/multimodal capabilities to analyze the visual diffs:
1. **Identify Differences**: Spot pixel-level or layout-level discrepancies.
2. **Consult Code**: Look at the corresponding `.css`, `.tsx`, or `.jsx` files to find the root cause (e.g., "Color is hardcoded instead of using `--brand-primary` token").
3. **Check for Logic Changes**: In `jsx`/`tsx` files, see if conditional rendering logic might cause visual bugs (e.g., "Loading state looks broken").

### 3. Triage Visual Findings

Use the standard `assets/finding-template.md` format with these visual-specific categories:

- **🔴 Action Required**: Critical regression (broken layout, missing button, wrong brand color).
- **🟡 Recommended**: Subtle design mismatch (spacing off by 2px, font-weight slightly different).
- **🟢 Minor**: Tiny polish items (icon alignment, hover transition speed).

**Visual Assets**: Ensure each finding includes references to the relevant `baseline_url` and `actual_url` (if available) to be used by `github-post.md`.

**Finding Source**: Set `source: "visual-reviewer"`.

## Output

- `visual_findings`: List of findings identified during visual analysis.
- Append these to the chunk's findings list.
