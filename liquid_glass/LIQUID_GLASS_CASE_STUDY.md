# Liquid Glass Architecture Case Study: Cresto & Glasense UI

A comprehensive technical case study and replication guide examining how the creator of **Cresto** ([Nevodev/Cresto](https://github.com/Nevodev/Cresto)) implemented **Liquid Glass** in Android using Jetpack Compose, Android Graphics Shading Language (AGSL), and hardware-accelerated `RenderEffect`.

---

## 1. Executive Summary & Core Philosophy

Standard Android "frosted glass" (Glassmorphism) typically stops at a single `RenderEffect.createBlurEffect(...)` with a translucent color overlay.

**Cresto's Liquid Glass** operates on a fundamentally different optical model:
1. **Real Physical Refraction (Snell's Law Simulation):** It does not just blur the background; it bends and distorts light rays around the edges of the container using custom AGSL shaders driven by 2D Signed Distance Fields (SDF).
2. **Dual-Branch Multi-Layer Compositing:** It executes two independent optical branches with different blur radii and refraction depths, then blends them via `BlendMode.SRC_OVER`.
3. **Photorealistic Specular Highlight Rim:** It computes normal angles in real-time on the GPU to draw specular edge light, overlaid with an inverted optical rim light texture (`glass_top_rim_light.png`) via `BlendMode.Plus`.
4. **Volume Ambient Shadows & Inner Depth:** It nests internal ambient shadows and body highlights using Jetpack Compose `GraphicsLayer` and `DrawModifierNode`.

---

## 2. The Optical Rendering Stack

```
┌────────────────────────────────────────────────────────────────────────┐
│                   Foreground Content (Text, Icons, Buttons)            │  <-- 100% Sharp
├────────────────────────────────────────────────────────────────────────┤
│                   Specular Highlights & Optical Rim Light              │  <-- AGSL Highlight + glass_top_rim_light.png
├────────────────────────────────────────────────────────────────────────┤
│                   Inner Ambient Shadow & Body Volume Glow              │  <-- GlassVisual.kt (BlendMode.Plus)
├────────────────────────────────────────────────────────────────────────┤
│                   Dual-Branch Refraction & Blur Compositor             │  <-- MultiLayerGlassEffect.kt (BlendMode.SRC_OVER)
│  ┌──────────────────────────────────┐┌───────────────────────────────┐ │
│  │ Branch 1: Micro-Blur + Deep Lens ││ Branch 2: Wide-Blur + Rim Lens│ │
│  │ Blur 2dp, Refraction 48dp        ││ Blur 16dp, Refraction 48dp    │ │
│  └──────────────────────────────────┘└───────────────────────────────┘ │
├────────────────────────────────────────────────────────────────────────┤
│                   Background Canvas / Underneath View Hierarchy        │  <-- Captured by Kyant Backdrop
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Deep Dive into the AGSL Shaders

All shaders are defined in [`Shader.kt`](./Shader.kt). They run natively on the device GPU via `android.graphics.RuntimeShader` (Android 13+ / API 33+).

### 3.1 The Refractive Lens Shader (`LENS_SHADER`)

The heart of the liquid glass effect is the optical lens shader. Instead of simply blurring pixels, it calculates the **geometric surface normal** of the rounded rectangle and displaces the texture coordinates:

```glsl
float sdRoundedRect(float2 coord, float2 halfSize, float radius) {
    float2 cornerCoord = abs(coord) - (halfSize - float2(radius));
    float outside = length(max(cornerCoord, 0.0)) - radius;
    float inside = min(max(cornerCoord.x, cornerCoord.y), 0.0);
    return outside + inside;
}

float2 gradSdRoundedRect(float2 coord, float2 halfSize, float radius) {
    float2 cornerCoord = abs(coord) - (halfSize - float2(radius));
    if (cornerCoord.x >= 0.0 || cornerCoord.y >= 0.0) {
        return sign(coord) * normalize(max(cornerCoord, 0.0));
    } else {
        float gradX = step(cornerCoord.y, cornerCoord.x);
        return sign(coord) * float2(gradX, 1.0 - gradX);
    }
}

float circleMap(float x) {
    return 1.0 - sqrt(1.0 - x * x);
}

uniform shader content;
uniform float2 size;
uniform float4 cornerRadii;
uniform float refractionHeight;
uniform float refractionAmount;
uniform float depthEffect;

half4 main(float2 coord) {
    float2 halfSize = size * 0.5;
    float2 centeredCoord = coord - halfSize;
    float radius = radiusAt(coord, cornerRadii);
    float sd = sdRoundedRect(centeredCoord, halfSize, radius);

    // If pixel is deep inside the container body (beyond refractionHeight), render un-refracted
    if (-sd >= refractionHeight) {
        return content.eval(coord);
    }
    sd = min(sd, 0.0);

    // Spherical curve mapping for glass meniscus
    float d = circleMap(1.0 - -sd / refractionHeight) * refractionAmount;
    float gradRadius = min(radius * 1.5, min(halfSize.x, halfSize.y));
    
    // Calculate normal vector pointing inward/outward along glass curvature
    float2 grad = normalize(
        gradSdRoundedRect(centeredCoord, halfSize, gradRadius)
            + depthEffect * normalize(centeredCoord)
    );
    
    // Sample backdrop texture displaced by normal gradient * displacement magnitude
    return content.eval(coord + d * grad);
}
```

#### Why This Creates "Liquid" Glass:
- **Signed Distance Field (SDF):** `sdRoundedRect` returns the exact mathematical distance from any pixel to the boundary of the squircle/rounded rectangle.
- **Surface Normal Gradient:** `gradSdRoundedRect` computes the partial derivatives $\nabla \text{SDF}$, yielding the exact normal vector of the curved glass edge.
- **Spherical Meniscus:** `circleMap(x) = 1.0 - sqrt(1.0 - x*x)` mathematically models the convex curve of a droplet or beveled glass edge.
- When background content moves behind this glass, the background appears to refract through a real physical lens.

---

### 3.2 The Specular Highlight Shader (`DEFAULT_HIGHLIGHT_SHADER`)

To simulate directional ambient lighting reflecting off the glass rim:

```glsl
uniform float2 size;
uniform float4 cornerRadii;
layout(color) uniform half4 color;
uniform float angle;
uniform float falloff;

half4 main(float2 coord) {
    float2 halfSize = size * 0.5;
    float2 centeredCoord = coord - halfSize;
    float radius = radiusAt(coord, cornerRadii);
    float gradRadius = min(radius * 1.5, min(halfSize.x, halfSize.y));
    float2 grad = gradSdRoundedRect(centeredCoord, halfSize, gradRadius);
    
    // Dot product between surface normal and incident light angle
    float intensity = pow(abs(dot(grad, float2(cos(angle), sin(angle)))), falloff);
    return color * intensity;
}
```

- Calculates incident light angle $\theta$ (default $90^\circ$ for top-down light).
- Computes the dot product $\mathbf{N} \cdot \mathbf{L}$ raised to `falloff` exponent, creating intense gleams on the top and bottom edges while feathering the sides.

---

### 3.3 Bezier Tone-Mapping & Dithering (`AGSL_CODE`)

Used in `MaterialRecipeRenderEffect.kt` to mimic Apple VisionOS/iOS tone curve:

```glsl
float bezierMap(float x) {
    float invX = 1.0 - x;
    return (invX * invX * invX) * p0
        + 3.0 * (invX * invX) * x * p1
        + 3.0 * invX * (x * x) * p2
        + (x * x * x) * p3;
}
```

- Modifies the luminance curve through a cubic Bézier polynomial to preserve contrast in bright/dark regions.
- Adds **Interleaved Gradient Noise** dithering (`interleavedGradientNoise(fragCoord)`) to eliminate color banding artifacts in 8-bit displays when heavy blurs are applied.

---

## 4. Multi-Layer Compositing Engine (`MultiLayerGlassEffect.kt`)

Rather than applying a single blur, Cresto constructs a dual-pass composite:

```kotlin
@RequiresApi(Build.VERSION_CODES.TIRAMISU)
fun multiLayerGlassEffect(
    width: Float,
    height: Float,
    cornerRadii: FloatArray,
    firstBlurRadius: Float,      // e.g. 2.dp (sharp edge detail)
    firstOpacity: Float,         // e.g. 1.0f
    firstRefractionHeight: Float, // e.g. 16.dp
    firstRefractionAmount: Float, // e.g. 48.dp
    secondBlurRadius: Float,     // e.g. 16.dp (deep background wash)
    secondOpacity: Float,        // e.g. 0.8f
    secondRefractionHeight: Float,
    secondRefractionAmount: Float
): ComposeRenderEffect {
    val first = glassBranch(width, height, cornerRadii, firstBlurRadius, firstOpacity, firstRefractionHeight, firstRefractionAmount)
    val second = glassBranch(width, height, cornerRadii, secondBlurRadius, secondOpacity, secondRefractionHeight, secondRefractionAmount)

    return RenderEffect.createBlendModeEffect(
        first,
        second,
        BlendMode.SRC_OVER
    ).asComposeRenderEffect()
}
```

### The `glassBranch` Pipeline:
Each branch connects three hardware stages:
1. `RenderEffect.createBlurEffect(blurRadius, blurRadius, Shader.TileMode.DECAL)`
2. `RenderEffect.createRuntimeShaderEffect(lensShader, "content")`
3. `RenderEffect.createChainEffect(lens, blur)`: Feeds the blurred output directly into the refraction lens!
4. `RenderEffect.createColorFilterEffect(ColorMatrixColorFilter(...))`: Applies opacity.

---

## 5. Optical Rim Light Overlay (`GlassUtility.kt`)

In addition to AGSL specular highlights, Cresto overlays a pre-rendered high-dynamic-range rim light bitmap:

```kotlin
fun DrawScope.drawGlassRim(
    image: ImageBitmap,
    height: Dp,
    alpha: Float = 0.75f,
    blendMode: BlendMode = BlendMode.Plus
) {
    val rimHeightPx = height.toPx()
    val sourceSize = IntSize(image.width, image.height)
    val destinationSize = IntSize(size.width.toInt(), rimHeightPx.toInt())

    // 1. Top rim highlight
    drawImage(
        image = image,
        srcSize = sourceSize,
        dstSize = destinationSize,
        blendMode = blendMode,
        alpha = alpha
    )
    
    // 2. Inverted bottom rim highlight
    withTransform({
        scale(scaleX = 1f, scaleY = -1f)
    }) {
        drawImage(
            image = image,
            srcSize = sourceSize,
            dstSize = destinationSize,
            blendMode = blendMode,
            alpha = alpha
        )
    }
}
```

Using `BlendMode.Plus` (additive blending) causes the rim light to physically saturate and brighten whatever colored content lies beneath the glass.

---

## 6. Inner Shadows & Body Depth (`GlassVisual.kt`)

For components where full AGSL shaders might be disabled or paired with additional depth:

```kotlin
fun Modifier.glassVisual(shape: () -> Shape): Modifier {
    val tirEdge = Shadow(radius = 0.25.dp, color = Color.Black.copy(alpha = 0.3f))
    val glassShadow = InnerShadow(radius = 16.dp, color = Color.Black.copy(alpha = 0.1f), offset = DpOffset(0.dp, 8.dp))
    val glassLight = InnerShadow(radius = 1.dp, color = Color.White.copy(alpha = 0.2f), blendMode = BlendMode.Plus)
    val glassBody = InnerShadow(radius = 8.dp, color = Color.White.copy(alpha = 0.1f), blendMode = BlendMode.Plus)

    return this
        .then(ShadowElement(shapeProvider, shadow = { tirEdge.copy(offset = DpOffset(0.75.dp, 0.dp)) }))
        .then(ShadowElement(shapeProvider, shadow = { tirEdge.copy(offset = DpOffset((-0.75).dp, 0.dp)) }))
        .then(InnerShadowElement(shapeProvider, shadow = { glassShadow }))
        .then(InnerShadowElement(shapeProvider, shadow = { glassLight.copy(offset = DpOffset(0.dp, 1.dp)) }))
        .then(InnerShadowElement(shapeProvider, shadow = { glassLight.copy(offset = DpOffset(0.dp, (-1).dp)) }))
        .then(InnerShadowElement(shapeProvider, shadow = { glassBody.copy(offset = DpOffset(0.dp, 4.dp)) }))
        .then(InnerShadowElement(shapeProvider, shadow = { glassBody.copy(offset = DpOffset(0.dp, (-4).dp)) }))
}
```

---

## 7. How to Replicate Cresto's Liquid Glass in Any App

### Step 1: Add Dependencies
In your `libs.versions.toml`:
```toml
[versions]
backdrop = "2.0.0"
shapes = "1.2.0"

[libraries]
backdrop = { module = "io.github.kyant0:backdrop", version.ref = "backdrop" }
shapes = { module = "io.github.kyant0:shapes", version.ref = "shapes" }
```

In your module `build.gradle.kts`:
```kotlin
dependencies {
    implementation(libs.backdrop)
    implementation(libs.shapes)
}
```

### Step 2: Copy the Liquid Glass Files
Copy the files directly from this `liquid_glass/` directory into your project:
- [`Shader.kt`](./Shader.kt)
- [`MultiLayerGlassEffect.kt`](./MultiLayerGlassEffect.kt)
- [`GlassUtility.kt`](./GlassUtility.kt)
- [`GlassVisual.kt`](./GlassVisual.kt)
- [`MaterialRecipes.kt`](./MaterialRecipes.kt)
- [`MaterialRecipeRenderEffect.kt`](./MaterialRecipeRenderEffect.kt)
- [`glass_top_rim_light.png`](./glass_top_rim_light.png) (place in `res/drawable/`)

### Step 3: Wrap Background in a Backdrop
In your root layout or screen:
```kotlin
import com.kyant.backdrop.rememberBackdrop
import com.kyant.backdrop.backdrop

val backdrop = rememberBackdrop()

Box(modifier = Modifier.fillMaxSize()) {
    // 1. Content underneath the glass
    LazyColumn(modifier = Modifier.fillMaxSize().backdrop(backdrop)) {
        items(50) { ItemCard(it) }
    }

    // 2. Liquid Glass Floating Bar / Card
    Box(
        modifier = Modifier
            .align(Alignment.BottomCenter)
            .padding(16.dp)
            .glass(
                backdrop = backdrop,
                shape = RoundedRectangularShape(28.dp),
                style = GlassStyle(
                    firstBlurRadius = 2.dp,
                    secondBlurRadius = 16.dp,
                    firstRefractionAmount = 48.dp
                )
            )
            .padding(horizontal = 24.dp, vertical = 12.dp)
    ) {
        Text("Liquid Glass Floating Capsule", color = Color.White)
    }
}
```

---

## 8. Performance & Hardware Constraints

Cresto explicitly displays a warning in `AppearanceScreen.kt`:
> *"Enabling 'Liquid Glass' can significantly impact performance."*

### Why It's Heavy:
1. **Multiple GPU Render Targets:** For each frame, `backdrop` captures the underlying layer into an offscreen buffer.
2. **Dual-Pass SDF Ray Displacements:** The GPU executes two separate blur shaders and two runtime AGSL shaders per pixel before compositing them with `SRC_OVER`.
3. **Minimum OS Requirement:** Full AGSL refraction requires **Android 13+ (API 33+)** because `RuntimeShader` with `createRuntimeShaderEffect` is not available on older versions.

### Recommended Optimizations for Production:
1. Provide a user toggle (like Cresto's `KEY_LIQUID_GLASS`).
2. Fallback gracefully to single-pass `RenderEffect.createBlurEffect(24dp, 24dp)` on API 31–32 and elevated containers on API < 31.
3. Keep the refracted surface area constrained (capsules, top app bars, floating action buttons) rather than full-screen glass surfaces during 120 FPS list scrolling.
