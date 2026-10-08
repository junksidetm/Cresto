# Project Version History - Cresto

## [2026-10-03 18:13] - Fork Lineage & Liquid Glass Extraction Case Study
- **Action:** Forked Cresto from `https://github.com/Nevodev/Cresto.git` to `junksidetm/Cresto`.
- **Case Study & Learning Extraction:** Created `liquid_glass/` folder containing the complete implementation modules and architecture case study for replicating Liquid Glass in Android Jetpack Compose.
- **Files Included in liquid_glass/:**
  - `Shader.kt`: Contains AGSL runtime shaders (`LENS_SHADER`, `DEFAULT_HIGHLIGHT_SHADER`, `AGSL_CODE`).
  - `MultiLayerGlassEffect.kt`: Dual-branch refraction and blur compositor (`RenderEffect.createBlendModeEffect`).
  - `GlassUtility.kt`: `Modifier.glass`, `Modifier.glassDecorations`, `drawGlassRim`, and `GlassStyle`.
  - `GlassVisual.kt`: Jetpack Compose `GraphicsLayer` and `DrawModifierNode` inner ambient shadows and volume glow.
  - `MaterialRecipes.kt`: Predefined material recipes (`thin`, `ultraThin`, `regular`, `thick`, `appBar`, `floatingBar`).
  - `MaterialRecipeRenderEffect.kt`: RuntimeShader color mapping effect.
  - `glass_top_rim_light.png`: Pre-rendered high-dynamic-range optical rim light texture.
  - `LIQUID_GLASS_CASE_STUDY.md`: Exhaustive architecture documentation and replication guide.
- **Libraries & Tools:**
  - Android Gradle Plugin: `9.3.1`
  - Kotlin: `2.4.10`
  - Backdrop (`io.github.kyant0:backdrop`): `2.0.0`
  - Shapes (`io.github.kyant0:shapes`): `1.2.0`
  - AGSL / RuntimeShader: Android 13+ (API 33+)
- **Status:** 100% (Fork completed, case study documented, and extraction module established).

## [2026-10-08 18:07:50 IST] - Tri-Platform Source Mirrors Integration
- **Action**: Added GitHub (Main), Codeberg (Mirror), and GitLab (Mirror) repository badges and dedicated Source Mirrors section in README.md.
- **Files Modified**:
  - `README.md`
  - `Version.md`
- **Status**: 100% (Completed & Synced)
