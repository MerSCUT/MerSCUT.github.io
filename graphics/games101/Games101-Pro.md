# Games101 Review

## 1. Linear Algebra

### Basic Concepts

- **Normalized Vector**: If a vector is strictly used to represent direction, it should be normalized (length = 1).
- **Dot Product**: Used for finding the angle between vectors (cosine) and projection, or a type of distance metric between two vectors.
- **Cross Product**:
  - **Definition**: Follows the Right-hand Rule.
  - **Application**: **Determining whether a point lies inside a triangle.**

### Homogeneous Coordinates

- **Definition**: Adding an extra dimension to represent points and vectors distinctly.
  - **Point**: $P = (x, y, z) \to (x, y, z, 1)$
  - **Vector**: $\vec{v} = (x, y, z) \to (x, y, z, 0)$
  - **Key Properties**:
    - point + vector = point
    - vector + vector = vector
    - point - point = vector
    - point + point = midpoint (Result is $(2x, 2y, 2z, 2)$, which simplifies to $(x, y, z, 1)$).
  - **Canonical Form**: $P = (x, y, z, w) \equiv (x/w, y/w, z/w, 1)$ when $w \neq 0$.

### Transformations

- **Types**: Rotation, Translation, Scale, Shear.

- **Affine Transformation** (Linear Map + Translation):
  $$
  \begin{bmatrix}x'\\y'\end{bmatrix} = \begin{bmatrix}a_{11} & a_{12} \\ a_{21} & a_{22} \end{bmatrix}\begin{bmatrix}x\\y\end{bmatrix} + \begin{bmatrix}t_x\\t_y\end{bmatrix}
  $$
  

- **Homogeneous Matrix Representation**:
  $$
  \begin{bmatrix}x'\\y' \\ 1\end{bmatrix} = \begin{bmatrix}a_{11} & a_{12} & t_x \\ a_{21} & a_{22} & t_y \\0 & 0 & 1 \end{bmatrix}\begin{bmatrix}x\\y \\ 1\end{bmatrix}
  $$
  

- **Note**: The Rotation matrix is an **orthogonal matrix**, meaning $R^T = R^{-1}$.

------

## 2. Rasterization

### 2.1 The Rasterization Pipeline

**Pipeline Overview (Per Frame):**

1. **Model Transformation**

   - Objective: Transform objects from **Local Space** to **World Space**.

2. **View Transformation**

   - Objective: Transform **World Space** to **Camera (View) Space** (Move the camera to the <u>canonical position</u>).

   - **Camera Definition**:

     1. Position: $e$
     2. Gaze Direction (Look-at): $\vec{g}$
     3. Up Vector: $\vec{t}$ 

   - **Standard (Canonical) Camera Configuration**:

     1. Position at origin: $e = (0,0,0)^T$
     2. Looking down $-Z$: $\vec{g} = (0,0,-1)^T$
     3. Up direction is $Y$: $\vec{t} = (0,1,0)^T$
     4. Right direction: $\vec{g} \times \vec{t} = (1,0,0)^T$

   - Transformation Matrix ($M_{view}$):

     Composed of a translation matrix $T_{view}$ (moving $e$ to origin) and a rotation matrix $R_{view}$ (aligning axes).

   $$
   M_{view} = R_{view} \cdot T_{view} = \begin{bmatrix}x_{\vec{g}\times \vec{t}} & x_{\vec{t}} & x_{-\vec{g}} &0 \\ y_{\vec{g}\times \vec{t}} & y_{\vec{t}} & y_{-\vec{g}} &0 \\ z_{\vec{g}\times \vec{t}} & z_{\vec{t}} & z_{-\vec{g}} &0 \\ 0 & 0 & 0& 1 \end{bmatrix}\cdot \begin{bmatrix} 1 & 0 & 0 & -e_x \\ 0 & 1  & 0 & -e_y \\ 0 & 0 & 1 & -e_z \\ 0 & 0 & 0 & 1 \end{bmatrix}
   $$

   

3. **Projection Transformation**

   - Objective: Map the view frustum $[l,r] \times [b,t] \times [f,n]$ to the **Canonical View Volume** $[-1,1]^3$.

   **A. Perspective-to-Orthographic ($M_{persp \to ortho}$)**

   - "Squishes" the frustum into a cuboid.

     $$M_{persp\to ortho} = \begin{bmatrix}n & 0 & 0 & 0 \\ 0  & n & 0 & 0 \\ 0 & 0 & n+f & -nf \\ 0 & 0 & 1 & 0\end{bmatrix}$$

   - Note: $n$ is the $z$-coordinate of the **near** plane, $f$ is the **far** plane.

   **B. Orthographic Projection ($M_{ortho}$)**

   - Maps the cuboid center to the origin and scales it to $[-1, 1]^3$.

     $$M_{ortho} = \begin{bmatrix}\frac{2}{r-l} & 0 & 0 & 0\\ 0 & \frac{2}{t-b} & 0 & 0 \\ 0 & 0 & \frac{2}{n-f} & 0 \\ 0 & 0& 0 & 1\end{bmatrix}\cdot \begin{bmatrix}1 & 0 & 0 & -\frac{r+l}{2} \\ 0 & 1 & 0 & -\frac{t+b}{2} \\ 0 & 0 & 1 & -\frac{n+f}{2} \\ 0 & 0 & 0 & 1 \end{bmatrix}$$

   **C. Deriving Frustum Bounds ($l, r, b, t$)**

   - Given: **Aspect Ratio** and **Field of View Y** ($fovY$).
   - Assumptions: Symmetric frustum ($l = -r, b = -t$).
   - Formulas:
     1. $\tan(fovY/2) = \frac{t}{|n|}$
     2. $aspect\_ratio = \frac{r}{t}$

4. **Viewport Transformation**

   - Objective: Map the Canonical Cube $[-1,1]^3$ to Screen Space $[0, width] \times [0, height]$ ($\times [-1,1]$).
   - **Equations**:
     - $x_{screen} = width \cdot \frac{x_{canonical} + 1}{2}$
     - $y_{screen} = height \cdot \frac{y_{canonical} + 1}{2}$
     - $z$ is usually preserved for depth buffering.
   - **Pixel Definitions**:
     - Indices: $(x, y)$ (integers).
     - Centers: $(x + 0.5, y + 0.5)$.

#### Aliasing (Jaggies)

- **Root Cause**: **Undersampling** (Sampling frequency is too low compared to the signal frequency).
- **Solutions (Anti-Aliasing)**:
  - Increase Resolution (Impractical/Expensive).
  - **Super-Sampling Anti-Aliasing (SSAA / MSAA)**: Sample multiple points per pixel and average them.

#### Z-buffer (Depth-buffer)

- **Purpose**: Handling **Visibility / Occlusion**.

- **Algorithm**:

  ```
  // Initialize Z-buffer with Infinity (or max depth) for every pixel
  for (each triangle T) {
      for (each pixel (x,y) inside T) {
          z_current = interpolated_depth(x, y);
          if (z_current < z_buffer[x,y]) {   // Closer to camera
              pixel_color[x,y] = shading_result;
              z_buffer[x,y] = z_current;
          }
      }
  }
  ```

------

### 2.2 Shading

**Objective**: Applying a material to an object/model.

#### The Blinn-Phong Reflectance Model

A local illumination model assuming light consists of three components:

1. **Ambient**: Constant background light.
2. **Diffuse**: Light scattered in all directions.
3. **Specular**: Shininess/Highlights.

$$
L = L_a + L_d + L_s
$$



**Key Vectors (All Normalized):**

- $\vec{v}$: View direction (from shading point to eye).
- $\vec{l}$: Light direction (from shading point to light source).
- $\vec{n}$: Surface normal.
- $\vec{h}$: **Halfway vector** (bisector of $\vec{v}$ and $\vec{l}$).

**Physical Laws:**

- **Lambert’s Cosine Law**: Energy received is proportional to the cosine of the angle between the normal and light direction ($\cos \theta = \vec{n} \cdot \vec{l}$).
- **Light Falloff (Inverse Square Law)**: Light intensity decays over distance $r$. Intensity = $I / r^2$.

#### 1. Diffuse Term

- **Characteristic**: View-independent (looks the same from any angle).
  $$
  L_d = k_d \cdot \frac{I}{r^2} \cdot \max(0, \vec{n}\cdot \vec{l})
  $$
  Where

  - $k_d$: Diffuse coefficient (Surface Color).

#### 2. Specular Term (Blinn-Phong)

- **Characteristic**: View-dependent; simulates the reflection of the light source.

- **Mechanism**: Measures how close the normal $\vec{n}$ is to the halfway vector $\vec{h}$.

  - $\vec{h} = \text{normalize}(\vec{v} + \vec{l})$

  $$
  L_s = k_s \cdot \frac{I}{r^2} \cdot \max(0, \vec{n} \cdot \vec{h})^p
  $$

  Where

  - $k_s$: Specular coefficient (usually white).
  - $p$: **Shininess** factor (controls the size of the highlight; usually $100 \sim 200$).

#### 3. Ambient Term

- **Characteristic**: Constant approximation of global illumination (indirect light).

$$
L_a = k_a \cdot I_a
$$



#### Shading Frequencies

Blinn-Phong is used to calculate **the color at shading points.**

How do we discretize shading application across the mesh?

1. **Flat Shading**: Shade **each triangle**.
   - Calculates shading once per face (using the face normal).
   - Result: Faceted look.
2. **Gouraud Shading**: Shade **each vertex**.
   - Calculate shading at vertices, then interpolate **colors** across the triangle.
3. **Phong Shading**: Shade **each pixel**.
   - Interpolate **normal vectors** across the triangle.
   - Calculate shading for every pixel using the interpolated normal.
   - **Pros**: Best visual result (smoothest highlights).
   - **Cons**: Highest computational cost.

Vertex Normal Calculation:

Weighted average of the normals of adjacent faces.
$$
N_v = \frac{\sum_i w_i \vec{n}_i}{\| \sum_i w_i \vec{n}_i\|}
$$
(Where $w_i$ is typically the area of the adjacent triangle $i$).