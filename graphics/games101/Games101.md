# Games101 Review

## 1. Linear Algebra

- Normalized Vector
  - If a vector is strictly used to represent direction, it should be normalized.
- Dot Product
  - Application : Metric the distance between two vectors.
- Cross Product
  - Definition : Right-hand Rule;
  - Application : **Determine whether a point lies inside a triangle**
- Homogeneous Coordinates
  - Definition : Adding an extra dimension to represent points and vectors distinctly.
    - $ P = (x,y,z) \to (x,y,z,1)$
    - $\vec v = (x,y,z) \to (x,y,z, 0)$
    - Several truth :
      - point + vector = point
      - vector + vector = vector
      - point - point = vector
      - point + point = point (midpoint)
    - $ P = (x,y,z,w) = (x/w, y/w, z/w, 1)$ when $w \neq 0$.
- Transformation
  - Rotation
  - Translation

$$
\begin{bmatrix}x'\\y'\end{bmatrix} = \begin{bmatrix}a_{11} & a_{12} \\ a_{21} & a_{22} \end{bmatrix}\begin{bmatrix}x\\y\end{bmatrix} + \begin{bmatrix}t_x\\t_y\end{bmatrix}
$$

It can be represented as
$$
\begin{bmatrix}x'\\y' \\ 1\end{bmatrix} = \begin{bmatrix}a_{11} & a_{12} & t_x \\ a_{21} & a_{22} & t_y \\0 & 0 & 1 \end{bmatrix}\begin{bmatrix}x\\y \\ 1\end{bmatrix}
$$

- Rotation matrix is orthogonal matrix : $R^T = R^{-1}$.

## 2. Rasterization

### 2.1 Rasterization

Pipeline (Every Frame) :

1. Model Transformation

   1. Adjust the object to the target place.

2. View Transformation

   1. Move the camera to standard position.

   2. Some Vector is used to define a camera

      1. Camera Position $e$  
      2. Gaze vector $\vec g$ : where the camera gazes at.
      3. Top vector $\vec t$ : Represent the "head direction" of camera

   3. By default, the standard position of camera :

      1. $e = (0,0,0)^T$;
      2. $\vec g = -\vec z = (0,0,-1)^T$
      3. $\vec t = \vec y = (0,1,0)^T$
      4. Notice that $\vec g \times \vec t = \vec x = (1,0,0)^T$

   4. Transformation matrix : 
      $$
      M_{view} = \begin{bmatrix}x_{\vec g\times \vec t} & x_{\vec t} & x_{-\vec g} &0 \\ y_{\vec g\times \vec t} & y_{\vec t} & y_{-\vec g} &0 \\ z_{\vec g\times \vec t} & z_{\vec t} & z_{-\vec g} &0 \\ 0 & 0 & 0& 1 \end{bmatrix}\cdot \begin{bmatrix} 1 & 0 & 0 & -e_x \\ 0 & 1  & 0 & -e_y \\ 0 & 0 & 1 & -e_z \\ 0 & 0 & 0 & 1 \end{bmatrix}
      $$
      

3. Projection -- Perspective Transformation $[l,r] \times [b,t]\times [f,n] \to [-1,1]^3$

   1. (Perspective$\to$ Orthoganal) Transformation

      1. Turn Perspective projection to Orthoganal projection.
         $$
         M_{persp\to orth} = \begin{bmatrix}n & 0 & 0 & 0 \\ 0  & n & 0 & 0 \\ 0 & 0 & n+f & -nf \\ 0 & 0 & 1 & 0\end{bmatrix}
         $$
         where $n $ is $z$ coordinate of **near** plane, $f$ is the far one. 

      2. After $persp \to orth$ transformation, the perspective projection is equivalent to Orthoganal Projection.

   2. (Orthoganal) Transformation.

      1. Transformation from $[l,r]\times [b,t]\times [f,n] \to [-1,1]^3$.
         $$
         M_{orth} = \begin{bmatrix}\frac{2}{r-l} & 0 & 0 & 0\\ 0 & \frac2{t-b} & 0 & 0 \\ 0 & 0 & \frac2{n-f} & 0 \\ 0 & 0& 0 & 1\end{bmatrix}\cdot \begin{bmatrix}1 & 0 & 0 & -(r+l)/2 \\ 0 & 1 & 0 & -(t+b)/2 \\ 0 & 0 & 1 & -(n+f)/2 \\ 0 & 0 & 0 & 1 \end{bmatrix}
         $$
         Translation, then Scale.

      2. __How to get $l,r,b,t$ ?__ From __Aspect_ratio__ and __eye_fovY__ (Known)

         1. Assumption : $l = -r, b = -t$
         2. $aspect = \dfrac{r}{t}$.
         3. $\tan{fovY} = \dfrac{t}{|n|}$

4. Viewport Transformation :

   1. Put the 3D space scaled in $[-1,1]^3$ on the screen space :
      1. $[-1,1]^3 \mapsto [0,width]\times [0,height]\times [0,1]$.
   2. Transformation :
      1. $x_{screen} = width \cdot \dfrac{x + 1}{2}$
      2. $ y_{screen} = height \cdot \dfrac {y+1}2$
      3. $z_{screen} = \dfrac{z+1}2$
   3. Index of pixel : $(x,y)$. 
   4. Centre of pixel : $(x+0.5,y+0.5)$ 

### Aliasing Problem (Jaggies)

Cause : __Undersampling__



__Anti-Aliasing__ : 

- Improve resolution; (Unrealistic)
- **Super**-**Sampling**





### Z-buffer / Depth-buffer

Z-buffer is used for resolving **Occlusion.**

```
Initialize Z-buffer by Infinity for every pixel

for (each triangle):
    for (each pixel) :
        if (z_screen /* of pixel centre */ < value in z-buffer)
        	update frame-buffer /* color value */
        	z-buffer[] = z_screen
```



### 2.2 Shading

Objective : **Applying a material to an object/model.**

### Blinn-Phong Model

This simple model assume that the light reflected by each shading points consists of three parts :

- Specular highlights
- Diffuse Reflection
- Ambient lighting

So the final color
$$
L = L_s + L_d + L_a
$$
It's a local shading model. There are several (**Known**) vectors for calculation.

- $\vec v$ : Viewer direction.
- $\vec l$  : Light Direction. 
- $\vec n$ : Normal vector of shading points.
  - __Above vectors are all normalized__ (For they only sign the direction)
- light intensity $I $.
- Viewer Point, Light Source Position, Shading point's position.

And some laws for fomula :

- __Lambert's cosine law__
  $$
  \cos \theta = \vec n \cdot \vec l
  $$
  It's used in diffuse term.

- __Light Falloff__ : (Law of Conservation of Energy)

  Assume the light source is point.

  __The "Light Energy" on each spherical shell is same__.

  If light intensity $I$ for unit distance $1$, then $I/r^2$  for distance $r^2$.

Figure out each term of Blinn-Phong Shading model.

1. __Diffuse Term__ : 

   Features :

   - Diffuse Light is irrelavent to viewer direction.

   $$
   L_d = k_d \cdot \frac I{r^2} \cdot \max(0, \vec n\cdot \vec l)
   $$

   Where

   - $k_d$ is diffuse coefficient;
   - $r$ is distance between light source and shading point.

2. __Specular Highlight Term__ :

   Features :

   - Reflection direction will absorb high intensity of light.
   - Metrics by $\vec h \cdot \vec n$, where $\vec h = normalized(\vec v + \vec l)$

   $$
   L_s = k_s \cdot \frac I{r^2} \cdot \max(0, \vec h \cdot \vec n)^p
   $$

   Where 

   - $p$ is a constant  to control the reflection range. Usually $100$ to $200$.
   - $k_s$ is specular coefficient.

3. __Ambient Term__ :

   Features : Light absorbed from environment is same. (Approximation)
   $$
   L_a = k_a \cdot I_a
   $$
   Where

   - $k_a$ is ambient coefficient;
   - $I_a$ is environment light intensity. (Given or Known)



### Shading Frequencies

How do we discrete the object into shading points (Then apply Blinn-Phong model ?)

> Usually, the information of __vertices__ (color, position, normal...) is given.

- __Flat shading__ : Shade __each triangle__ 
  - Seen a triangle (and choose a representative point, e.g. centre of gravity) as a "shading point".
  - Triangle is a __flat__ surface, which has a normal $\vec n$. 
- __Gouraud shading__ : Shade each __vertex__.
  - **Each vertex is a shading point.** 
  - Color of points inside the triangle will be calculated by interpolation.
    - Need **BaryCentric** **Coordinates**.
- __Phong Shading__ : Shade each __pixel__.
  - The centre of a pixel is a shading point.
  - During **Rasterization**, use the normal of vertex to __interpolate__ **the normal vector** (need to be normalized) of shading point. 
  - Then use Blinn-Phong model to calculate the color of shading point.
  - *Most commonly used; The best effect/result of shading (among these 3 frequencies); The highest computational complexity.



Q : How to define the __normal of vertex__ ?

A : The average normal vector of adjacent triangles of vertex.
$$
\vec n_v = \frac{\sum_i \vec n_i}{\| \sum_i \vec n_i\|}
$$
$\vec n_i$ is the normal of  $i$-th adjacent traignel. Further more, it could be __weighted__ form.
$$
\vec n_v = \frac{\sum_i w_i\vec n_i}{\| \sum_i w_i \vec n_i\|}
$$
where $w_i$ is weight.



### 2.3 Barycentric Coordinates

It's a **coordinate system for a triangle**.

- For interpolation across triangle and obtain a smoothly vary values.



In 3D space, a point $P$ inside $\Delta ABC$ can be represented as
$$
P = \alpha A + \beta B + \gamma C
$$
where 

- $\alpha + \beta + \gamma = 1$ : It forces  $P$ to lie in the same plane of $\Delta ABC$.
- $\alpha,\beta,\gamma >0$ : Forces $P$ to lie into the triangles. 



Every point $P$ correspond to $(\alpha,\beta,\gamma)$. So if we know some information $V_A,V_B,V_C$ of $A,B,C$, **we can interpolate it to $P$ by**
$$
V_P := \alpha V_A + \beta V_B + \gamma V_C
$$
where

- $V_{(\cdot)}$ can be vertex, color, depth ($z$-coordinate), texture coordinate, etc.

Q : Given $P$ inside the triangle $ABC$, how to calculate it's barycentric coordinate $\alpha,\beta,\gamma$ ?

- Define $S_A = S_{PBC}$, $S_B = S_{PAC}$, $S_C = S_{PAB}$, then
  $$
  \alpha = \frac{S_A}{S_A+S_B+S_C},\beta = \frac{S_B}{S_A+S_B+S_C}, \gamma = \frac{S_C}{S_A+S_B+S_C}
  $$

Q : Then how to get $S_A, S_B, S_C$ ?

- Remind that $\|\overrightarrow {PB} \times \overrightarrow{PC}\| = S_{PBC}/2 = S_A/2$



## 3. Geometry

### 3.1 Implicit



### 3.2 Explicit

Main : parametric equations

#### 3.2.1 Bezier Curves



### 3.3  Mesh Processing