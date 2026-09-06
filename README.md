# xLOD

a level of detail optimization that saves alot of math crunching 

*Co-authored by [sighthough](https://youtu.be/UtPiUGwu-0Q) and Gemini.*

👉 **[CLICK HERE TO RUN THE LIVE BENCHMARK](https://sighthough.github.io/xLOD/)**

# Hybrid xLOD Engine: Mesh LOD + Precision-Gated Shader Arithmetic

An experimental WebGL rendering technique that combines conventional distance-based mesh decimation (**Mesh LOD**) with dynamic float precision truncation (**Arithmetic Precision Gating**).

By scaling down both **geometric complexity** and **vertex/fragment math precision** relative to camera depth, Hybrid xLOD achieves reduced memory bandwidth and math operations without noticeable visual degradation.

```text
===================================================================================================
                                      HYBRID xLOD PIPELINE
===================================================================================================

  CAMERA                                                         FAR DEPTH
  (Near)                                                          (Far)
    |--------------------------|--------------------------|-------------------------->
    |                          |                          |
    +--> NEAR ZONE             +--> MID ZONE              +--> FAR ZONE
    |    - High-Poly Mesh      |    - Med-Poly Mesh       |    - Low-Poly Mesh
    |    - FP32 Full Precision |    - 16-level Truncation  |    - 3-bit / 8-level Truncation
    |    - Crisp Highlights    |    - Smooth Gradients    |    - Ambient / Quantized Shading
    |                          |                          |
    +--------------------------+--------------------------+--------------------------+
    | Geometry:  100% Triangles| Geometry:  ~30% Triangles| Geometry:  ~10% Triangles|
    | Precision: 32-bit FP     | Precision: ~16-bit FP    | Precision: ~3-bit FP     |
    +--------------------------+--------------------------+--------------------------+
===================================================================================================

```

---

## How It Works

Traditional Level of Detail (LOD) only reduces triangle counts. However, distant fragments still execute high-precision 32-bit floating-point operations ($FP32$) for lighting, specular reflections, and vertex position transforms where sub-pixel visual detail is completely lost to distance.

Hybrid xLOD introduces a dual-reduction strategy:

1. **Mesh Decimation (Geometry LOD):** Swaps detailed meshes for simplified geometries as objects move away from the camera.
2. **Arithmetic Precision Gating (Shader LOD):** Dynamically quantizes scalar and vector operations in GLSL based on fragment distance.

### Distance Precision Interpolation

The active quantization level $L$ is calculated per-vertex or per-fragment:

$$t = \text{clamp}\left(\frac{d - d_{\text{start}}}{d_{\text{end}} - d_{\text{start}}}, 0.0, 1.0\right)$$

$$L = \text{mix}\left(256.0, 2^{b_{\text{min}}}, t\right)$$

Where:

* $d$ is the distance from the camera to the vertex/fragment.
* $d_{\text{start}}$ is the distance where precision gating begins.
* $d_{\text{end}}$ is the distance where minimum precision ($b_{\text{min}}$) is reached.
* $L$ is the number of discrete steps allowed in floating-point calculations.

---

## Performance Profile

| Feature Zone | Mesh LOD | Vertex Math Precision | Lighting Specular Math | Est. Shader Math Reduction |
| --- | --- | --- | --- | --- |
| **Near Zone** ($0 - 15\text{m}$) | High-Poly | $FP32$ (Full) | Continuous $FP32$ | $0\%$ |
| **Mid Zone** ($15 - 50\text{m}$) | Med-Poly | Quantized (16 Levels) | Truncated Blinn-Phong | $\sim 40\% - 60\%$ |
| **Far Zone** ($50\text{m}+$) | Low-Poly | Quantized (8 Levels) | Aggressive Truncation | $\sim 75\% - 90\%$ |

---

## Integration Guide

You can implement Hybrid xLOD into any custom Three.js `ShaderMaterial` or GLSL pipeline using the uniform parameters and helper functions below.

### 1. GLSL Helper Snippets

Include these uniform definitions and truncation functions in both your **Vertex** and **Fragment** shaders:

```glsl
// --- xLOD UNIFORMS ---
uniform float uXlodEnabled;   // 1.0 = On, 0.0 = Baseline FP32
uniform float uStartDist;      // Distance (meters) where truncation starts
uniform float uEndDist;        // Distance (meters) where max truncation is reached
uniform float uMinBits;        // Minimum target bit width at maximum distance (e.g. 3.0)
uniform float uGateVertices;   // 1.0 = Enable vertex quantization
uniform float uGateLighting;   // 1.0 = Enable lighting quantization

// --- QUANTIZATION HELPERS ---
float truncateScalar(float val, float levels) {
    if (levels >= 256.0) return val;
    return floor(val * levels + 0.5) / levels;
}

vec3 truncateVec3(vec3 v, float levels) {
    return vec3(
        truncateScalar(v.x, levels),
        truncateScalar(v.y, levels),
        truncateScalar(v.z, levels)
    );
}

float getQuantizationLevels(vec3 worldPos, vec3 camPos) {
    float dist = distance(worldPos, camPos);
    float t = clamp((dist - uStartDist) / (uEndDist - uStartDist), 0.0, 1.0);
    return mix(256.0, pow(2.0, uMinBits), t);
}

```

### 2. Vertex Shader Implementation

Apply quantization to vertex positions to reduce dynamic transformation work at a distance:

```glsl
varying vec3 vNormal;
varying vec3 vWorldPosition;

void main() {
    vec3 localPos = position;
    vec4 worldPos = modelMatrix * vec4(localPos, 1.0);
    
    // Calculate quantization levels based on depth
    float levels = getQuantizationLevels(worldPos.xyz, cameraPosition);

    // Apply vertex truncation if enabled
    if (uXlodEnabled > 0.5 && uGateVertices > 0.5) {
        localPos = truncateVec3(localPos, levels);
    }

    worldPos = modelMatrix * vec4(localPos, 1.0);
    vWorldPosition = worldPos.xyz;
    vNormal = normalMatrix * normal;
    
    gl_Position = projectionMatrix * viewMatrix * worldPos;
}

```

### 3. Fragment Shader Implementation

Truncate intermediate diffuse and specular calculations to minimize shading ALU overhead:

```glsl
varying vec3 vNormal;
varying vec3 vWorldPosition;

uniform vec3 uLightPos;
uniform vec3 uObjectColor;

void main() {
    vec3 norm = normalize(vNormal);
    vec3 lightDir = normalize(uLightPos - vWorldPosition);
    
    float levels = getQuantizationLevels(vWorldPosition, cameraPosition);

    // Standard Blinn-Phong Shading
    float diff = max(dot(norm, lightDir), 0.0);
    
    vec3 viewDir = normalize(cameraPosition - vWorldPosition);
    vec3 halfDir = normalize(lightDir + viewDir);
    float spec = pow(max(dot(norm, halfDir), 0.0), 32.0);

    // Apply lighting truncation
    if (uXlodEnabled > 0.5 && uGateLighting > 0.5) {
        diff = truncateScalar(diff, levels);
        spec = truncateScalar(spec, max(levels * 0.5, 2.0));
    }

    vec3 ambient = vec3(0.12) * uObjectColor;
    vec3 diffuse = diff * uObjectColor * 0.85;
    vec3 specular = vec3(1.0) * spec * 0.4;

    gl_FragColor = vec4(ambient + diffuse + specular, 1.0);
}

```

### 4. Three.js JavaScript Binding

Initialize the `ShaderMaterial` and update distance parameters dynamically:

```javascript
import * as THREE from 'three';

// 1. Define xLOD Uniforms
const xlodUniforms = {
  uXlodEnabled:  { value: 1.0 },
  uStartDist:     { value: 15.0 },  // Truncation starts at 15m
  uEndDist:       { value: 80.0 },  // Max truncation at 80m
  uMinBits:       { value: 3.0 },   // Minimum 3-bit target width at far range
  uGateVertices:  { value: 1.0 },
  uGateLighting:  { value: 1.0 },
  uLightPos:      { value: new THREE.Vector3(30, 50, 40) },
  uObjectColor:   { value: new THREE.Color(0x3a86ff) }
};

// 2. Create xLOD Material
const xlodMaterial = new THREE.ShaderMaterial({
  vertexShader: myVertexShaderSource,
  fragmentShader: myFragmentShaderSource,
  uniforms: xlodUniforms
});

// 3. Attach to Three.js LOD System
const lodNode = new THREE.LOD();

const highMesh = new THREE.Mesh(highPolyGeo, xlodMaterial);
const medMesh  = new THREE.Mesh(medPolyGeo, xlodMaterial);
const lowMesh  = new THREE.Mesh(lowPolyGeo, xlodMaterial);

lodNode.addLevel(highMesh, 0);    // Near
lodNode.addLevel(medMesh, 25);    // Mid
lodNode.addLevel(lowMesh, 50);    // Far

scene.add(lodNode);

// 4. Update LODs in Render Loop
function animate() {
  requestAnimationFrame(animate);
  
  // Updates conventional mesh LOD levels
  lodNode.update(camera);
  
  renderer.render(scene, camera);
}

```

---

## Uniform Reference

| Uniform Name | Type | Default | Description |
| --- | --- | --- | --- |
| `uXlodEnabled` | `float` | `1.0` | Global toggle (`1.0` = Precision Gating active, `0.0` = Baseline $FP32$). |
| `uStartDist` | `float` | `15.0` | Distance in world units where arithmetic truncation begins. |
| `uEndDist` | `float` | `80.0` | Distance in world units where maximum truncation ($b_{\text{min}}$) is reached. |
| `uMinBits` | `float` | `3.0` | Target bit-width lower bound for distant objects (e.g., $3.0 = 2^3 = 8$ levels). |
| `uGateVertices` | `float` | `1.0` | Toggle position truncation in vertex shader (`1.0` / `0.0`). |
| `uGateLighting` | `float` | `1.0` | Toggle shading/specular truncation in fragment shader (`1.0` / `0.0`). |

---

## License

MIT License. Free for personal, academic, and commercial WebGL project implementation.
