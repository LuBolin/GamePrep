# Game Maths for Unreal Interviews

Game-maths interviews test whether you can translate geometry into stable code and explain spaces, assumptions and edge cases. Memorised formulas are less valuable than deriving them from vectors, drawing the configuration and verifying it visually.

Version target: **UE5.3–UE5.6**. Unreal's mathematical conventions and API types matter; maintained docs may display UE5.7, so exact overloads/types should be checked in the target branch.

## 1. Conventions before calculation

Unreal Editor uses a **left-handed, Z-up** Cartesian system: +X forward, +Y right, +Z up. `FRotator` uses degrees: Pitch about +Y, Yaw about +Z and Roll about +X, with the documented Unreal signs. [SRC-MATH-001] [SRC-MATH-006]

State for every problem:

- point or direction/vector;
- local/component/world/view/tangent/clip/NDC/screen space;
- degrees or radians;
- unit vector or arbitrary length;
- centimetres/seconds or another unit;
- ray (`t≥0`), line (unbounded) or segment (`0≤t≤1`);
- tolerance/degenerate policy;
- handedness and multiplication/composition convention.

Most “mysterious maths bugs” are one unstated convention.

## 2. Scalars and numerical robustness

Floating-point values are approximate. Use domain tolerances rather than exact equality for computed values, but do not apply one global epsilon blindly: position tolerance in centimetres, unit-vector tolerance and time tolerance represent different scales.

Robust habits:

- compare squared distance when only ordering against a squared radius is needed;
- clamp dot products to `[-1,1]` before `acos`;
- guard division by near-zero length/denominator;
- reject or handle NaN/infinity early (`NaN` breaks ordinary comparisons);
- use `IsNearlyEqual/Zero`-style helpers with deliberate tolerance;
- normalise only when direction is needed and define zero-vector fallback;
- keep dimensions/units visible in names;
- wrap angles/deltas to an agreed range;
- prefer geometrically meaningful tests over arbitrary epsilons.

UE5 Large World Coordinates moved core coordinate variants toward double precision, but conversions to float, rendering domains and subtracting nearly equal large values can still lose precision. Precision type is not a replacement for choosing local/translated spaces. [SRC-MATH-002]

## 3. Points, vectors and basic operations

A point is a location; a vector is displacement/direction. They may share `FVector` storage, but transform differently:

- point + vector → point;
- point − point → displacement vector;
- vector + vector → vector;
- adding two points has no general geometric meaning.

For `v=(x,y,z)`:

```text
|v|² = v·v = x²+y²+z²
|v|  = sqrt(v·v)
unit(v) = v/|v|, if |v| is non-zero
distance(a,b) = |b-a|
```

Use `DistSquared`/`SizeSquared` for range checks to avoid square root. Do not compare squared distance with an unsquared radius.

## 4. Dot product

```text
a·b = ax bx + ay by + az bz = |a||b| cos θ
```

For normalised vectors:

- dot `1`: same direction;
- dot `0`: perpendicular;
- dot `-1`: opposite;
- positive/negative: front/back half-space.

Uses:

- field-of-view cone: `dot(forward,toTargetNormal) >= cos(halfAngle)`;
- front/behind test;
- projected scalar component;
- diffuse lighting relation;
- plane signed-distance relation;
- reject velocity into a surface.

Avoid `acos` when only comparing an angle: precompute cosine threshold. If vectors are not normalised, magnitude scales the dot. [SRC-MATH-003]

## 5. Projection, rejection and reflection

Projection of `a` onto non-zero `b`:

```text
proj_b(a) = b * (a·b)/(b·b)
```

If `n` is unit length: `proj_n(a)=n(a·n)`.

Rejection/tangent component: `a - proj_b(a)`.

Reflection of incident vector `v` about unit normal `n`:

```text
r = v - 2(v·n)n
```

Derivation: subtract the normal component twice—once to reach tangent plane, once to mirror. Decide incident direction convention; reversing `v` changes signs. Uses include bounce, slide decomposition and aiming onto a plane.

## 6. Cross product and orientation

`a×b` is perpendicular to both; magnitude is `|a||b|sinθ`, equal to parallelogram area. Operand order reverses sign. In Unreal, verify result against the engine's left-handed axis convention instead of importing a memorised right-hand diagram. [SRC-MATH-001] [SRC-MATH-003]

Uses:

- construct perpendicular/right/up vectors;
- triangle normal/winding;
- signed left/right around an up axis: sign of `up·(forward×toTarget)` under the chosen convention;
- torque lever-arm relation awareness;
- area/degeneracy.

Parallel/near-parallel operands produce near-zero cross product; safe-normalising it needs a fallback axis.

## 7. Angle between vectors without losing sign

Unsigned angle:

```text
θ = acos(clamp(dot(normalize(a), normalize(b)), -1, 1))
```

For a signed planar angle around unit axis `n`, a robust pattern is:

```text
θ = atan2(n · (a × b), a · b)
```

after projecting/normalising as needed. `atan2` preserves quadrant and handles near-180° better than `acos` plus a separate sign.

For simple gameplay steering, compare dot/cross signs without calculating an angle at all.

## 8. Basis and coordinate spaces

An orthonormal basis has three mutually perpendicular unit axes. Coordinates are coefficients in that basis. For basis `(x,y,z)` and vector `v`:

```text
local = (v·x, v·y, v·z)
world = x*local.x + y*local.y + z*local.z
```

This dot-product inverse works because the basis is orthonormal. With non-uniform scale/skew, use the actual inverse transform/matrix.

### Point versus vector transform

A point is affected by scale, rotation and translation. A direction/vector is affected by scale/rotation but not translation; a direction that should ignore scale uses the no-scale operation. Unreal exposes separate transform operations for this reason. [SRC-MATH-004]

Common bug: transforming a direction as a position adds actor location; transforming a socket point as a vector omits location.

### Local/world debugging

1. Write space in variable names (`WorldAim`, `LocalOffset`).
2. Draw basis axes and input/result.
3. Set parent identity; then translation only, rotation only, non-uniform scale.
4. Round-trip `Local → World → Local` within tolerance.
5. Test a known basis vector and point.

## 9. Matrices and homogeneous coordinates

A linear 3×3 matrix transforms vectors/bases. A 4×4 homogeneous transform can combine linear transform and translation by representing a point with `w=1` and direction with `w=0`.

Matrix multiplication composes transforms and is generally non-commutative: order matters. Do not state “TRS order” from memory without naming whether vectors are rows/columns and whether multiplication applies left/right; use the engine API or derive by transforming one known point.

### Inverse and transpose

- inverse undoes a transform when invertible;
- rotation matrix inverse equals transpose only for orthonormal rotation;
- zero scale makes an inverse singular/undefined;
- normals under non-uniform scale require inverse-transpose-style treatment, not ordinary vector transform, then normalisation.

Unreal's geometry transform documentation explicitly treats normals specially with reciprocal scale and rotation. [SRC-MATH-005]

## 10. `FTransform` composition

Conceptually an Unreal transform stores translation, quaternion rotation and 3D scale. Transforming a point follows the type's documented scale/rotate/translate semantics; use `TransformPosition`, `TransformVector`, no-scale variants and inverse counterparts instead of hand-assembling when possible. [SRC-MATH-004] [SRC-MATH-005]

Non-uniform scale plus rotation composition can imply shear that one simple SRT representation cannot exactly preserve. Converting/decomposing between matrices and SRT may lose that component. Avoid casual negative/non-uniform scale in gameplay hierarchies and test composition/inversion rather than assuming algebraic identities.

## 11. Euler angles, Rotators and gimbal lock

Euler/Rotator angles are readable and convenient for editor/control constraints, but they are an ordered sequence of axis rotations. At particular orientations, two rotational axes align—gimbal lock—losing one independent degree of freedom in that parameterisation. This is not “the object cannot rotate”; it is the coordinate representation becoming singular.

Use Rotators for authoring, UI, constrained yaw/pitch and conversions. Use quaternions/basis transforms for composition/interpolation and arbitrary 3D orientation. Repeated Rotator addition can wrap and produce unintuitive shortest-path behaviour; normalise deltas deliberately. [SRC-MATH-006]

## 12. Quaternions

A unit quaternion represents orientation/rotation. Key ideas:

- `q` and `-q` represent the same orientation (double cover);
- multiplication composes rotations and order matters;
- inverse of a unit quaternion reverses rotation;
- quaternions should remain normalised after operations that may drift;
- they avoid Euler singularity for representation/composition but do not remove every control-design issue.

### Slerp

Spherical linear interpolation travels at constant angular rate along the sphere arc. Unreal's `TQuat::Slerp` corrects alignment/shortest path and returns normalised output; full-path and non-normalised variants have different contracts. [SRC-MATH-007]

Before custom interpolation, account for quaternion sign: if dot is negative, negating one endpoint selects the equivalent nearer representation. For tiny angular separation, normalised linear interpolation is a common approximation.

## 13. Lerp and smoothing

Linear interpolation:

```text
lerp(a,b,t) = a + t(b-a)
```

For fixed endpoints and `t∈[0,1]`, it follows a straight line. Common mistakes:

- calling `lerp(current,target,speed*dt)` every frame and claiming constant speed;
- not clamping `t` when overshoot is forbidden;
- lerping Euler components across wrap;
- frame-rate-dependent smoothing with fixed alpha per frame.

Repeated current-to-target lerp is exponential-like easing. For a frame-rate-independent decay with rate `λ`:

```text
alpha = 1 - exp(-λ dt)
current = lerp(current,target,alpha)
```

For constant speed use a move-towards rule bounded by `speed*dt`. For damped spring motion, integrate a specified model and test timestep stability rather than stacking lerps.

## 14. Kinematics and forces

Dimensions:

- position `[L]`;
- velocity `[L/T]`;
- acceleration `[L/T²]`;
- force `[M L/T²]`;
- impulse `[M L/T]`, producing `Δv = J/m` in the simple linear model.

Constant acceleration update awareness:

```text
v_next = v + a dt
p_next = p + v dt + 0.5 a dt²    // analytic for constant a
```

Semi-implicit Euler commonly updates velocity then position and is often more stable for game simulation than explicit Euler, but exact engine physics uses its own solver/integration.

Gravity is acceleration; friction/restitution/contact/torque are physics-model concepts, not scalar tweaks with universal formulas. Distinguish kinematic movement from force-driven rigid-body simulation and do not fight the physics solver by teleporting its authoritative transform.

## 15. Closest point on segment

Given segment `A→B`, point `P`, `d=B-A`:

```text
t = dot(P-A,d) / dot(d,d)
t = clamp(t,0,1)
closest = A + t*d
```

If `dot(d,d)` is near zero, the segment is a point; return A. This primitive supports distance to path, capsule tests, aim assist and broadphase refinement. Unreal provides closest-point helpers in `FMath`. [SRC-MATH-008]

## 16. Ray and plane

Ray: `p(t)=o+t d`, `t≥0`. Plane with unit normal `n` through point `p0`: `n·(x-p0)=0`.

Substitute ray:

```text
denom = n·d
t = n·(p0-o) / denom
```

If `|denom|` is near zero, ray is parallel; separately test whether origin is coplanar. For a segment, accept only its parameter range. Decide one-sided/backface policy.

## 17. Ray and sphere

Sphere centre `c`, radius `r`. Let `m=o-c`; solve:

```text
|m+t d|² = r²
```

which yields quadratic coefficients. With unit `d`, a geometric closest-approach version can reduce operations. Handle:

- discriminant < 0: no real hit;
- ray starts inside: one positive exit hit;
- tangent: one repeated hit;
- both roots behind: line intersects, ray does not;
- non-unit direction: do not use unit-only simplification.

Return nearest acceptable `t`, point and normal `(point-c)/r` when radius is non-zero.

## 18. Ray and AABB (slab method)

For each axis, find `t` interval where ray lies between min/max slabs; intersect the three intervals:

```text
tEnter = max(axis enters)
tExit  = min(axis exits)
hit when tEnter <= tExit and tExit >= 0
```

For near-zero direction on an axis, origin must already lie inside that slab. Avoid unexamined division by zero/NaN. Record which axis produced entry for hit normal. Segment casts clamp parameter to segment interval.

AABB is axis-aligned in the space where tested. For OBB, transform ray into box local space and test local AABB, taking scale/direction parameterisation care.

## 19. Sweeps and continuous collision reasoning

A sweep asks whether a moving shape intersects during an interval, reducing tunnelling compared with endpoint overlap. A useful mental model is Minkowski expansion: sweeping a point against an expanded target (e.g. sphere/capsule radius added to bounds) where applicable.

Continuous collision is not free: choose which fast/small objects require it, define initial overlap, contact time/normal, multiple hits and remaining-time response. Engine sweeps include collision filters, complex/simple geometry and physics details beyond primitive derivations.

## 20. Bounds and culling maths

- sphere bound: cheap rotation-invariant tests, looser fit;
- AABB: cheap overlap, world-axis fit changes with rotation;
- OBB: tighter oriented fit, more expensive;
- frustum: intersection of camera half-spaces/planes.

Sphere-versus-plane uses signed distance and radius. AABB frustum tests use conservative support/extreme points or engine-provided bounds tests. Culling must avoid false negatives; conservative false positives cost later work but do not pop visible objects.

## 21. Rendering spaces

Typical conceptual pipeline:

```text
local/object → world → view/camera → clip → perspective divide → NDC → viewport/screen
```

- view transform expresses world relative to camera;
- projection maps view frustum into homogeneous clip space;
- perspective divide by `w` creates NDC;
- viewport maps NDC to pixels/depth range.

Never assume OpenGL/DirectX depth/sign conventions in UE shader code without checking the RHI/helper macros. Epic's older coordinate-space reference documents UE's clip/screen conventions and translated-world precision, but it is **UE4.27-era**; current rendering code is authoritative. [SRC-MATH-009]

Perspective makes projected size inversely related to depth. Orthographic projection does not foreshorten with distance. A point behind/near camera requires explicit handling before using screen coordinates.

## 22. UV, tangent space and normal maps

UVs parameterise a surface in 2D texture coordinates and may tile or lie outside `[0,1]` depending on addressing/content. Tangent basis `(T,B,N)` converts tangent-space detail to the surface/world basis. Mirrored UVs can flip handedness, commonly encoded with a tangent sign.

Tangent-space normal maps store directions relative to the surface; they are mostly blue because +Z points away from surface. Treat normal data as linear vector data, not colour to gamma-decode arbitrarily, and use the texture/import/material settings expected by Unreal. [SRC-MATH-010]

Under non-uniform scale, normals require special transformation and renormalisation. Interpolated tangent bases/normals may also need normalisation.

## 23. Depth buffer awareness

Depth is produced after projection and is generally non-linear with view distance under perspective. Precision distribution and reversed-Z/infinite-far implementation are renderer/platform details. Consequences:

- equal depth differences do not mean equal world distances;
- reconstructing position requires the correct inverse projection/device-depth convention;
- near-plane choice strongly affects precision;
- z-fighting appears when surfaces are too close relative to depth precision;
- disabling depth tests/early rejection can increase overdraw cost. [SRC-MATH-010]

Use engine material/shader helpers for scene depth conversion rather than inventing a universal formula.

## 24. Linear versus gamma/sRGB

Lighting arithmetic and interpolation should occur in linear-light values. sRGB/gamma encoding allocates display/storage precision perceptually and is not linear energy. Therefore:

- decode colour textures to linear before lighting;
- do not mark data textures (normal/roughness/masks) as sRGB;
- blend/interpolate colours in the intended space;
- output encoding/tone mapping is a later display step.

`FLinearColor` versus packed/display colour types expresses part of this distinction in Unreal; exact texture pipeline settings require target/content verification. [SRC-MATH-011]

## 25. Debugging workflow

### Wrong direction/rotation

1. Print/draw current space, origin and +X/+Y/+Z axes.
2. Verify point versus vector transform.
3. Verify normalisation and zero fallback.
4. Check cross operand order/handedness.
5. Check degree/radian and Rotator axis convention.
6. Reduce parent transform to identity, then add rotation/scale.
7. Round-trip through inverse.

### Intersection misses

1. Draw primitive, origin, direction and accepted parameter interval.
2. Validate bounds/radius and chosen space.
3. Check zero direction/degenerate segment/parallel denominator.
4. Inspect roots/enter-exit interval before filtering.
5. Distinguish line, ray and segment acceptance.
6. Differential-test against engine query or brute sample for generated cases.

### Jitter/drift

Check frame-rate dependence, accumulating Euler angles, non-normalised quaternions/vectors, float/double narrowing, large world coordinates, cancellation, unstable integration and two systems writing transforms.

## 26. Common misconceptions

| Misconception | Better model |
|---|---|
| “FVector is a point.” | Storage can represent points or vectors; semantics determine translation. |
| “Dot product gives the angle.” | It gives magnitude-scaled cosine relation; angle needs normalisation and inverse trig. |
| “Cross product always gives right.” | Order and coordinate handedness determine sign. |
| “Normalise everything.” | It loses magnitude and is undefined near zero; normalise when unit direction is required. |
| “Quaternion has no gimbal lock, so convert freely to Euler.” | Quaternion representation avoids Euler singularity; conversion/control constraints can reintroduce it. |
| “Lerp with speed*dt moves at speed.” | Repeated lerp is easing; use move-towards for constant speed. |
| “Inverse rotation matrix is transpose.” | Only for orthonormal rotation, not arbitrary scale/shear. |
| “Normals transform like vectors.” | Non-uniform scale requires inverse-transpose-style treatment. |
| “Ray hit means quadratic has roots.” | Accept roots only in ray/segment parameter interval. |
| “Depth is world distance.” | Perspective depth is projected/non-linear and API convention dependent. |

## 27. Strong interview answers

### Dot and cross products

> Dot measures alignment and, for unit vectors, is cosine of the angle; I use it for front/FOV/projection without acos when a threshold suffices. Cross returns a perpendicular whose magnitude relates to area/sine; I use it for basis/orientation and signed side tests, while checking operand order and Unreal's handedness. Both need zero/normalisation policy.

### Local versus world

> A coordinate is meaningless without its space. To transform a point I include translation; for a direction I exclude translation and decide whether scale applies. I name variables by space, use `FTransform` position/vector/inverse methods, draw basis axes, and test identity then rotation/non-uniform scale plus a local→world→local round trip.

### Quaternion versus Rotator

> Rotators are readable degree-based Euler parameters useful for editor and constrained yaw/pitch, but the ordered parameterisation has singularities and awkward wrapping/interpolation. Unit quaternions compose arbitrary 3D rotations and support shortest-arc slerp, though multiplication order, normalisation and q/-q equivalence matter. I author/display in Rotators and keep composition/interpolation in quaternions where appropriate.

## 28. Hands-on verification

Extend Project 7C with a visual maths playground:

1. draw vector, unit vector, dot cone, projection/rejection/reflection and cross basis;
2. convert point/vector between nested transforms and expose non-uniform scale failures;
3. compare Euler component lerp, quaternion slerp, repeated lerp and constant-speed move;
4. visualise closest segment point and ray/plane/sphere/AABB intervals/roots;
5. generate degenerates and differential-test against engine helpers/traces where semantics match;
6. display local/world/view/clip/NDC/screen coordinates for a moving point;
7. show tangent/world normals, sRGB/data-texture mistakes and depth non-linearity;
8. record tolerances, units, frame rates and UE version in each test.

## 29. Source map

- Unreal conventions/LWC/types: [SRC-MATH-001] [SRC-MATH-002] [SRC-MATH-003] [SRC-MATH-004] [SRC-MATH-005] [SRC-MATH-006] [SRC-MATH-007] [SRC-MATH-008]
- rendering spaces/material vectors: [SRC-MATH-009] [SRC-MATH-010] [SRC-MATH-011]
- geometry/intersection reference: [SRC-MATH-012] [SRC-MATH-013]
