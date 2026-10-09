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
+X against `-across` — because no pixel or aim-error check catches it.

`across` runs left hip to right hip, matching the body's own left-to-right.

`BONE_AIM` deliberately has **no Spine entry**. The hips basis already takes
its up axis from the hip-to-shoulder line, so aiming the spine along roughly
that same line applies the torso rotation a second time and doubles the
figure over.

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

The neck is not aimed and the head just follows the torso. Aiming it along
shoulders-to-nose threw the head right back on any rear view: the nose is then
on the far side of the shoulders and barely above them, so a small landmark
error swings the aim wildly. Less expressive, right far more often.

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
