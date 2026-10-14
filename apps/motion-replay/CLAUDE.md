# apps/motion-replay

Reconstructs a 3D capsule rig from the pose in a video clip. See root `CLAUDE.md`
for repo-wide conventions.

Ported from a standalone prototype. Pipeline: pick a file, sample it at
12/20/30 fps (capped at 900 frames), run MediaPipe Pose Landmarker over each
sampled frame, then drive a three.js rig from the returned world landmarks.

**Dependencies are loaded at runtime, not vendored.** `three` and
`@mediapipe/tasks-vision` come from jsdelivr and the `.task` model from
`storage.googleapis.com`; this is the second app after `rally-clipper` to pull an
ML model over the network. They are **dynamic** `import()` calls inside
`runPipeline()` rather than top-level imports (the prototype's approach): a module
whose top-level import fails never executes, so an unreachable CDN left every
control bound to nothing with no message on screen. Loading on demand means the
UI always renders and a network failure becomes `describeError()` text in the
progress area with the button re-enabled. `setupScene()` is guarded so a second
video reuses the existing renderer and scene.

Sampling uses `seekTo()` (await `seeked` per frame) rather than playing the video,
so no detection is dropped to keep up with playback; this is why it is slow.
Frames with no detection are left null, then back-filled by holding the last
confident pose, so the rig does not pop out of existence mid-clip
(`frames[i].held` marks those). `applySmoothing(alpha)` is an EMA over the world
landmarks where alpha is how much of the *previous smoothed* frame to keep, so 0
is raw and 0.85 heavily damped; it rebuilds `smoothedFrames` from `frames` on
every slider move, so the slider is non-destructive.

The body is a **skinned humanoid** (`mannequin.glb`, the Mixamo X Bot shipped
with the three.js examples, vendored at 2.9MB). `loadMannequin()` replaces its
materials with one white standard material and `updateMannequin()` retargets
MediaPipe's landmarks onto the skeleton each frame. `BONE_AIM` maps a model bone
to a landmark pair, `CHILD_OF` names each bone's child, and `aimBone()` rotates
a bone so the direction to that child lands on the landmark direction. It works
from the bone's *current* child direction rather than an assumed rest axis, so it
does not care which way the rig's bones point. Chains are walked parents first,
because aiming a bone moves its children. The hips take a basis built from the
hip line and the hip-to-shoulder line. Only rotations are driven, never
positions or lengths: MediaPipe world landmarks are already hip-relative, so the
figure performs in place with the model's own proportions, and noisy landmarks
cannot stretch a limb.

## Pose and appearance are separate problems

The reference this app is aimed at is a lacquered white figure of a tennis
player: a clothed male athlete, t-shirt with fabric folds, shorts, laced
shoes. The shipped `mannequin.glb` cannot be that. It is the Mixamo X Bot, a
segmented genderless rig with ball joints and no clothing, and no amount of
material or lighting work changes that silhouette.

So **the body is a parameter, not a constant.** `bodySource` is a URL or an
object URL from the `bodyInput` picker, and the retargeting only needs a
skeleton using Mixamo's bone names (`REQUIRED_BONES` is the list; a file
without them is refused by name rather than left standing in a T-pose). The
shipped mannequin is the default because it is the best thing that can be
vendored here, and because it is honest about being a rig rather than
pretending to be a person.

Checked, and rejected, from the three.js examples: `Soldier.glb` is an
armoured trooper with shoulder plates and pouches, `Michelle.glb` is a
stylised character in a crop top and wide trousers. Rendered all-white both
read worse than the X Bot for this. Both do load and pose correctly through
the picker, which is the useful part: an arbitrary Mixamo rig works.

Once the body can change, anything tuned to the shipped one is a bug. The
camera pad was a flat `0.10` and clipped the helmet off `Soldier`; `meshPad`
is now measured per model at load, as the largest distance the rest-pose mesh
reaches past its own bones.

What is still not solved is the rendering. A closed-form answer needs either
an asset we do not have (a rigged, clothed athlete that can be redistributed)
or a different technique entirely — image-space restyling of the source
footage rather than lighting a mesh. The pose pipeline is deliberately
decoupled from that choice.

The material is a `MeshPhysicalMaterial` with `clearcoat`, not a low-roughness
standard material: the reference has its highlights sitting ON the surface,
and plain low roughness blurs body tone and specular together into shiny
plastic. A clear coat needs something to reflect, so `studioEnvironment()`
paints an equirectangular canvas — near-black surround, soft white box
overhead, lime kicker low and behind — and runs it through `PMREMGenerator`.
No HDRI fetch, and it only touches PBR materials, so the unlit court is
unaffected.

## Silhouette: the default, and the one that actually looks right

Reconstructing geometry and lighting it keeps hitting the same wall — the
mesh is only ever as good as a monocular depth guess, and every artefact in
it shows up as a lump. But the thing being aimed at is a flat image of a
white figure, and that can be made directly.

`styliseFrame()` isolates the body and shades it in 2D, and the court, net
and shadow stay a real 3D scene underneath. `drawSilhouette()` places the
figure by projecting the floor and a figure's height into screen space, so it
stands on the court at the right size.

**Recolouring the photo does not work, and this is the thing to understand
before trying again.** A shirt, shorts and shoes are torso-sized patches of
different albedo. Dividing by a blurred copy of the luminance evens them out
— but any blur wide enough to flatten a torso also flattens the shading, and
anything narrower leaves hair, cap and shoes as black holes. Both were tried;
both look wrong in different ways.

**Shading the MATTE makes the body uniformly white by construction.** The
distance transform gives a rounded cross-section, its gradient gives a
surface normal, and that normal takes a light. The depth model's relief is
added to that height field first, so folds and the chest come through. The
photo's own tones are not used at all: a `photoMix` term was tried and the
version with it at zero was cleaner, because what it mostly contributed was
the shirt's shadow as a smudge.

This needs nothing per frame beyond the matte already stored, so it is cheap
and there is no extra memory.

### Matcap, not a hand-written light term

`buildMatcap()` bakes a lit sphere once and `styliseFrame()` looks it up with
`uv = normal.xy * 0.5 + 0.5`: a normal facing the camera samples the middle,
one turning away samples the rim. It replaced a hand-written
diffuse-plus-specular formula, which is faster and puts the whole material in
one function instead of inside a per-pixel loop.

It also fixed where the rim light goes. Driving the rim off the distance
transform put it a fixed number of pixels inside the outline, so it traced
the silhouette and nothing else. Keyed off the normal it lands wherever the
surface turns away, which includes the interior edges — between the arm and
the body, along the far leg — which is where rim light actually appears.

The height field is in matte pixels, so its gradient scales with how big the
subject is on screen. Normalise it before building the normal, or a large
subject comes out flat and a small one looks like crumpled foil.

Two things that bite:

- A canvas cannot hand out both a WebGL and a 2D context, so the figure needs
  `silCanvas` over the render rather than drawing into it.
- **Export must capture a composite.** `renderCanvas.captureStream()` records
  the WebGL layer only, so the export silently produced an empty court — a
  valid 39KB file of nothing, which is easy to miss because the export still
  "succeeded". `suite2.mjs` asserts on file size for exactly this.
- Align the figure by where the matte actually starts and stops, not by the
  canvas edge: the crop carries padding, and using the edge left it hovering.

The view button cycles **Silhouette → Mesh → Rig**. Overlay still draws the
rig whatever the main view is, because that is its whole purpose; it forces
the mannequin visible for its pass and restores it afterwards.

## Relief mode: outline from the frame, volume from the rig

The reference look is a clothed athlete, and no rig that can be vendored here
is one. But the frame already knows the outline exactly — this person, their
shirt, their shorts, their shoes — and knows nothing about depth. The posed
rig knows the opposite: that a torso is thicker than a forearm and how a chest
curves, with somebody else's outline. Relief mode takes the outline from the
frame and the thickness from the rig, **and never draws the rig**.

`buildRelief()`:

1. `PoseLandmarker` takes `outputSegmentationMasks: true` and returns a person
   matte beside the pose, from the model already loaded, so this costs no
   extra download.
2. Pose the rig and apply the same scale-and-place transform the overlay uses,
   so it sits on the person.
3. Render it from the source view into a depth buffer **twice**, front faces
   then back faces, giving the near and far surface of the body per pixel.
4. Inside the matte, thickness comes from those two surfaces. Where the matte
   reaches past the rig — hair, a loose shirt, shoes — there is no rig depth,
   so it falls back to a rounded cross-section off the matte's own distance
   transform, `sqrt(2Re - e^2)`, blended in over about 14 pixels.
5. Mesh front and back sheets and bridge them around the silhouette.

Things that are not obvious:

- Depth is read with `MeshDepthMaterial` and `RGBADepthPacking`, not a custom
  shader, because the built-in material gets skinning for free. The camera is
  **orthographic**, so `gl_FragCoord.z` is linear in view depth and unpacking
  gives metres with no perspective un-divide. `near`/`far` are pulled tight
  around the body so 24-bit packing is far more precision than needed.
- `readRenderTargetPixels` returns rows bottom-up; the matte is top-down.
- The rig's front and back surfaces **pinch together at its own silhouette**,
  and that edge lands in the middle of the person wherever the rig is
  narrower than they are. Blur the depth pair before combining or that edge is
  stamped into the skin as a ridge.
- The matte must be blurred **before** it is thresholded, or every edge is a
  staircase and the shading reads it as terracing.
- Without the rim bridge the body is two loose sheets and the rim light pours
  straight through.
- The relief is the person's build, not the rig's, so `frameCamera()` and the
  floor read `reliefBounds` in this mode. Leaving them on `manBounds` frames
  the scene for a mannequin that is no longer on screen.

### The depth model

`depth/` is **ARPortraitDepth**, vendored (11MB, four files). It could not be
downloaded in the sandbox — every host that serves it is blocked — so it was
attached by hand. TensorFlow.js only publishes a UMD bundle, so it loads as a
classic `<script>` and is read off `window.tf`, unlike three and MediaPipe
which are ES imports.

Contract, read out of `@tensorflow-models/depth-estimation` rather than
guessed: the person cut out on black, `[0,255]` mapped to `[0,1]`, resized to
**192x256** (width, height — the tensor is `[1,256,192,3]`). Output is
`[1,256,192,1]` and roughly `[0,2]`, **relative, not metric**.

It runs once per frame inside the detection pass and is stored quantised to a
byte over that frame's range: 49KB a frame against 196KB for floats, which is
what makes one per frame affordable. If it fails to load, the pipeline carries
on and relief falls back to the rig alone — `depthguard.mjs` blocks the model
files and checks exactly that.

**Only the model's high-frequency variation is used** — what it says minus a
blurred copy of itself. That is the fabric folds and the chest and the
shoulder blades, the things the rig cannot know. Throwing away its low
frequencies sidesteps the fact that the output is relative: there is no scale
to calibrate and a bad absolute reading cannot move the body.

Two traps, both of which produced convincing-looking rubbish:

- **The sign is not documented**, so it is settled by correlating against the
  rig. Correlating the model's high-frequency detail against the rig's RAW
  depth compares fine detail with overall body shape, which is near
  uncorrelated, so the sign came out of the noise and half the time the
  relief was inverted — hollows where the chest should stand out. High-pass
  the rig the same way first.
- **Scale off a high percentile, not the maximum.** The masked cut-out leaves
  a hard edge the model answers with a few extreme pixels; letting those set
  the scale flattened everything else to nothing.

### Smoothing the mesh, not just the depth field

`taubinSmooth()` runs over the finished mesh. Smoothing the height field
cannot touch the rim, because the front and back sheets are separate there
until they are bridged, so the silhouette kept a hard crease all the way
round however much the depth image was blurred.

Taubin rather than plain Laplacian: a positive pass then a slightly larger
negative one. Repeated Laplacian steps pull every vertex towards the
centroid, so the body visibly deflates a little more each iteration; the
negative pass undoes that while keeping the smoothing.

Borrowed from an Open3D article (`filter_smooth_laplacian`). The rest of that
pipeline is not applicable: it builds its "depth" by stacking the GRAYSCALE
image into a volume and marching cubes through it, so brightness becomes
height. That is a lithophane, not a reconstruction — on a lit white figure it
would carve the lighting into the geometry, highlights bulging and shadows
caving. The one other idea there worth revisiting is marching cubes itself,
which would give better topology around concavities than the grid-plus-rim
mesh here, at the cost of a voxel grid per frame.

### Take the rig's thickness, not its anatomy

Using the rig's front surface literally stamped the X Bot's own chest onto
every body: a smooth bulge over the ribs that is the model's shape, not the
person's. It was easy to mistake for a depth-model artefact — the fix was
found by setting `DEPTH_DETAIL` to 0 and seeing the identical lump.

Both the centre line and the half-thickness are now blurred hard across the
body before use, keeping what the rig genuinely knows (a torso is thicker than
a forearm, roughly where the body sits in depth) and discarding the rest. The
blurred thickness is then capped by the matte's own rounded profile, or the
body inflates past its outline at the wrists and ankles where the blur drags
the torso's thickness outwards.

**Relief is the default view.** The rig is a reference for the volume, not
something anyone wants to look at; it only appears if no body can be built, or
if the Overlay-row toggle is pressed.

Three things had to be true before it could be the default, and each was wrong
at first:

- **Build it once, not during playback.** Each build allocates full-frame
  float buffers and does two GPU readbacks, so doing that inside the playback
  loop pinned the main thread and the page stopped answering. Every frame's
  body is now built up front by `precomputeRelief()` with a progress bar, and
  playback is a buffer swap into one reused geometry. `RELIEF_CACHE_MAX` caps
  the clip length; past it relief is refused with a note rather than
  attempted.
- **Crop to the subject.** A player can be a fraction of the frame, so the
  distance transform, the blurs and the readbacks were spending nearly all
  their time on empty court and then throwing that resolution away decimating
  the mesh. Everything now runs on the matte's bounding box resampled to
  `RELIEF_WORK`. It is both the speed fix and the quality fix: the body gets
  the working grid to itself.
- **Keep the matte from the detection pass.** Re-detecting per frame to get a
  matte doubled the pipeline's work and was the single biggest cost. A float
  matte per frame is tens of MB, which is why it was not kept; but cropped,
  resampled to the working grid and packed one bit per pixel it is about 18KB
  a frame, so `packMatte()` stores it during detection and the landmarker is
  closed at the end of the pass as before.

**Scale matters because the court does not move.** The body is normalised to
`SKELETON_HEIGHT`, the same height the rig uses, over the whole clip at once.
Left at the subject's own metric scale it came out shorter than the rig and
waded through a net built for someone else. `reliefBounds` is padded by about
7% as well, because `frameCamera()` shrinks what it is handed by 8% (the rig's
`manBounds` already carries `meshPad`), and exact bounds cropped the head off
every frame. The floor still reads the unpadded minimum, so the feet stay on
the ground.

**Where it stands:** the body is the person's, closed, correctly scaled
against the court, and it animates. The head is still blobby — a cap merges
into the skull — and fine surface detail is not there, because all of it is
bounded by the matte's resolution. The next real lever is a monocular depth
model over the matte in place of the rig prior, which needs a host this
environment currently denies.

## Measuring pose accuracy

`mannequin.glb` ships with its own animation clips (`walk`, `run`, `idle`,
`sneak_pose`, …). That makes a **ground-truth harness** possible: render the
model playing a known clip to a video, feed that video back through the whole
pipeline, and compare each reconstructed bone direction against the direction
the animation actually had. Without it every change here is guesswork, because
a retargeting bug and a bad camera angle look identical in a screenshot.

What that measured, on a clean full-body three-quarter walk:

- **Per-bone error is about 15 degrees median, and it is mostly MediaPipe, not
  the retargeting.** Every aimed bone lands on its landmark direction to
  within 0.00 degrees, so the limb error *is* the landmark error. The `heavy`
  model only buys about 1.7 degrees over `full`, so there is no easy win there.
- **Subtracting a constant per-bone bias only takes 18.4 down to 14.6**, so
  the limbs are noisy rather than skewed, and no fixed calibration fixes them.
  Do not add one.
- **The torso was the exception**: hip-to-shoulder read 16 degrees forward of
  the real spine axis on the walk. That one is systematic and is what made
  every figure stoop.
- Watch for a degenerate case: on a near-static clip (`idle`, `sneak_pose`)
  the "bias" absorbs the entire pose and the residual collapses to about 1
  degree, which means nothing. Only moving clips say anything.

So the limb fidelity ceiling is the landmarks. What is worth working on is
everything else: where the body sits, how it deforms, and whether the parts
nothing drives look deliberate.

## The two landmark sets, and which one to believe

MediaPipe returns the pose twice. `worldLandmarks` is a metric 3D skeleton
hung off the hip centre; `landmarks` puts each joint in the IMAGE. Only the
second is actually observed — the 3D set's depth is inferred from one view —
and driving the rig from all three of its axes let the one guessed axis
corrupt the two measured ones.

`fuseImageXY()` takes x and y from the image and keeps z from the 3D set,
which is the only depth there is. Image landmarks are normalised to the frame,
so they are put in pixels and then fitted to the metric skeleton with one
uniform scale and offset, solved per frame by weighted least squares over the
visible joints. Fitting rather than assuming a scale keeps it working as the
subject walks toward the camera. It is an orthographic approximation — it
ignores that a nearer joint projects larger — which over one body at a normal
shooting distance is much smaller than the depth noise it replaces.

Measured on the ground-truth harness, fusion on against off:

| clip | limb median off | on |
| --- | --- | --- |
| walk | 12.3 deg | **11.0** |
| run  | 23.3 deg | **22.3** |

Every bone on the walk improved except `Spine2`, and the torso (`Hips`) went
20.3 to 14.3. So it is a consistent win, but a modest one on limbs and nearly
nothing on fast motion, where motion blur degrades the 2D landmarks too. Do
not expect it to rescue bad footage.

**The fused array carries its mapping as `imageMap`, never `map`.** It rides on
an array and the pipeline calls `world.map(...)` in several places, so that
name silently shadows `Array.prototype.map`; the symptom is the run dying
inside `applySmoothing` with the viewer simply never appearing.

Fusion also moves the origin: the fused set is centred on the joint centroid
rather than the hip centre. Nothing downstream cares, because the retargeting
only ever uses differences between landmarks — but the overlay does, and has
to place the hips explicitly.

## Overlay

`renderOverlay()` draws the posed mannequin back onto the frame it came from,
because the only way to know whether a reconstruction matches the footage is
to look at the two together. It runs the fusion's mapping backwards to get an
orthographic camera — orthographic because that mapping is a uniform scale
with no perspective in it — then scales the model so its torso matches the
observed one and puts its hips on the detected hips.

The model keeps its own proportions, so only the hips land exactly. That
mismatch is the point: a limb sitting off the person is the reconstruction
being wrong, where before you could only guess. It shares the one WebGL canvas
with the main view (render overlay camera, copy the canvas out to a 2D canvas
over the video, restore and re-render), so `suite2.mjs` checks both that the
overlay paints and that the normal render survives the toggle.

The hips basis is the one place a sign error hides. The rig's rest pose faces
**+Z with +Y up, which puts its left-to-right body axis on -X**, so the basis
columns are `(-across, up, facing)` and `facing = up x across`, not
`across x up`. Building it the other way round turns the figure a half turn
about its own axis, and that is nearly invisible in the obvious diagnostics:
every bone is aimed in world space *after* the hips are set, so each limb still
lands exactly on its landmark direction and the per-bone aim error stays at
zero. What you see instead is the chest facing the camera on a shot filmed from
behind, and the torso mesh folding around a pelvis pointing the other way —
which reads as a modelling or weighting problem rather than a basis one. Do not
chase it by flipping the sign of `across` on its own: `makeBasis` then gets a
left-handed set, `setFromRotationMatrix` returns nonsense, and the figure swings
into profile, which looks enough like a different bug to send you the wrong way.
`suite2.mjs` pins the invariant directly — hips +Z against `up x across`, hips
+X against `-across` — because no pixel or aim-error check catches it. Those
dots no longer read exactly 1, since the pelvis is deliberately pitched off the
hip-to-shoulder line; the half turn they guard against reads -1.

`across` runs left hip to right hip, matching the body's own left-to-right.

The pelvis does **not** take the whole torso lean. The hips basis uses a
`pelvisUp` pitched back toward the stance leg (the lower knee: a raised knee
says nothing about which way the pelvis points), and `Spine`, `Spine1` and
`Spine2` are aimed along the arc from `pelvisUp` to the hip-to-shoulder line,
so the waist keeps its shape instead of bending at one joint. On a standing
figure this took the torso error from 8.2 degrees to 3.9. This is why the old
"no Spine entry" rule is gone: aiming the spine *was* a double rotation back
when the hips also used hip-to-shoulder as their up axis, and is correct now
that they do not.

The **clavicles** (`LeftShoulder`/`RightShoulder`) are aimed from the shoulder
midpoint out to each shoulder landmark. Without them the arm swings entirely
at the humerus and the shoulder mesh collapses into the chest on any pose away
from rest.

The **neck** is aimed at the EAR midpoint, which beat both alternatives on the
ground-truth harness: 23.6 degrees against 32.5 for not aiming it at all and
48.6 for aiming at the nose. The ears sit either side of the head's centre so
their midpoint barely moves as the head turns, where the nose swings right
round and, on a rear view, sits on the far side of the shoulders.

`relaxHands()` curls the fingers once at load. Nothing in the landmarks drives
fingers, so they otherwise stay splayed in the rest pose all clip, and a pair
of starfish hands is the loudest single tell that the body is a puppet. It
curls about the knuckle line so it does not care what the rig calls its finger
axes, and it works out which way is "closed" by experiment rather than
assumption: bend one joint each way and keep the sign that moves the fingertip
toward the thumb.

**Landmarks below 0.5 visibility are not aimed at.** MediaPipe returns a full
skeleton whatever it can actually see, and filming from behind hides both ears
and the far arm; aiming at those guesses as confidently as the rest is what
made rear views flail. Nothing resets the skeleton between frames, so skipping
an aim holds the last confident pose, which is what you want. `confidenceNote()`
surfaces the same signal to the user, because "detected on 100% of frames" with
every joint at 40% confidence otherwise reads as the app being broken when the
footage was the problem.

**Grounding.** Only rotations are driven, so the hips stay pinned at the origin
and the feet land wherever the pose puts them; on a clip where the legs never
fully extend the whole figure hovers over its own shadow. `groundMannequin()`
takes the lift out through a saturating curve (`GROUND_GIVE`), so a planted
foot sits exactly on the floor while a real jump keeps what is left over
instead of being glued down. It needs the floor before it can run and
`measureMannequin()` needs grounding before it can measure, so that function
does two passes: one with grounding off to find the floor, one with it on to
fit the camera.

The floor goes under the **sole**, not under `manBounds.minY`: those bounds are
padded outwards so the camera clears the mesh, and using them for the ground
put the floor a hand's width below the feet. `soleDrop` measures how far the
foot mesh hangs below the toe bone with `Box3.setFromObject` at load time —
the one moment that call is trustworthy here, because nothing has been posed
yet so bind pose and rest pose are the same thing.

`boneKey()` strips non-alphanumerics from bone names. The file stores
`mixamorig:Hips` but GLTFLoader sanitises node names to `mixamorigHips`, so a
literal lookup silently matched nothing and left the model in its T-pose.

Two sizing bugs both came from `Box3.setFromObject`, which **ignores skinning and
returns bind-pose bounds**. It reported this model as 0.688 tall, so scaling to
1.8 made a 4.8-unit giant standing two thirds of a metre above the floor; and
used for camera framing it returned a box whose top was below the hips. Both are
now measured from bone world positions instead: scale from the rest skeleton
against `SKELETON_HEIGHT`, and framing from `measureMannequin()`, which solves
every sampled frame and records where the bones actually go, so the fit covers
the whole take.

Lighting is a dark studio with a white key and two lime rim lights behind, which
is what lifts the figure off the black background. **Light layers cannot scope a
light to particular objects**: three.js only collects a light if it shares a
layer with the *camera*, so putting the rims on their own layer removed them
entirely rather than restricting them to the body. The court is kept out of
their way by being unlit (`MeshBasicMaterial`) instead. Court lines and the net
are canvas textures rather than geometry. The court is dark grey, not black,
because a black contact shadow is invisible on a black floor.

The contact shadow is a radial-gradient plane under the feet, tracked from the
**toe** bones: the ankle bone sits a quarter of a metre above the ground even
when the foot is planted, so measuring lift from it meant a standing figure
never got a full-strength shadow. It fades and shrinks as the feet leave the
clip's lowest point, so a jump does not drag a hard shadow with it.

The capsule rig below is still built, and is what you get if the model fails to
load; the fallback is exercised by pointing `MANNEQUIN_URL` at a missing file.
It is a **cylinder per bone plus a sphere per joint**, not the prototype's
one scaled capsule per bone: scaling a capsule along its length stretches its
hemispherical caps too, so bone ends ballooned in proportion to limb length.
Spheres at `JOINTS` (derived from `BONES`) keep a constant radius and give the
continuous-body look the capsule was reaching for. `toVec()` writes into scratch
vectors (`_a`, `_b`, `_mid`, …) allocated once in `setupScene()`, since
`updateRig()` runs over every bone every frame during playback and export.
MediaPipe world landmarks are metres, origin near the hip centre, y down, and
z smaller the nearer the camera. `toVec()` is therefore `(x, -y, -z)`: a half
turn about X, determinant +1. It used to negate x as well, which is a
**reflection**, not a rotation — that swaps the skeleton's chirality and makes
`Quaternion.setFromRotationMatrix` return garbage for the hips.

In the capsule fallback a **neck** cylinder runs from the shoulder midpoint to the head, and the head
sits on the nose rather than 0.06 beyond it along the shoulders-to-nose line.
The prototype's placement left the head visibly detached, because that line
points up and forward and the nose is already the topmost landmark.

The camera **fits itself to the rig** rather than sitting at a fixed distance.
The prototype's `position.set(0, 0.2, 2.4)` with a 40 degree vertical field of
view sees about 1.75 world units of height; a standing figure is roughly 1.8
heel to crown, so the head was clipped off the top of every render. `measureRig()`
keeps the point cloud (every joint plus the nose, sampled to at most 240 frames)
and `frameCamera()` solves for the distance that puts all of it inside the
frustum. It has to account for **depth**, not just the bounding box: MediaPipe
puts the nose about 0.3 in front of the hip origin, so the head is markedly
closer to the lens and projects larger, and a box fit still clipped it. The solve
is `camZ >= z + (|y - cy| + pad) / tan(vFov/2)` per point, and the same
horizontally, taking the furthest requirement plus 8%. `frameCamera()` re-runs
from `resizeRenderer()` because the horizontal term depends on aspect. The floor
disc drops to the lowest point of the rig, and deliberately bleeds off the bottom
of the frame: it is a ground plane, not part of the figure.

**Zero detections abort rather than showing an empty stage.** A clip with no
person in it used to land in the viewer with nothing rendered and a note reading
"Pose detected on 0% of sampled frames. Gaps are held from the last confident
detection", when there had been none to hold. `runPipeline()` now throws advice
("works best with one person, full body in frame…") which `describeError()`
passes through unreworded, leaving the user on the upload card.

Export is `MediaRecorder` over `renderCanvas.captureStream()`, render only (not
the side-by-side). MediaRecorder captures in real time, so the export loop walks
frames at playback pace — a 900-frame clip takes as long as the clip does.
