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

The rig is a **cylinder per bone plus a sphere per joint**, not the prototype's
one scaled capsule per bone: scaling a capsule along its length stretches its
hemispherical caps too, so bone ends ballooned in proportion to limb length.
Spheres at `JOINTS` (derived from `BONES`) keep a constant radius and give the
continuous-body look the capsule was reaching for. `toVec()` writes into scratch
vectors (`_a`, `_b`, `_mid`, …) allocated once in `setupScene()`, since
`updateRig()` runs over every bone every frame during playback and export.
MediaPipe world landmarks are metres, origin near the hip centre, y down: y is
flipped for three.js and x is flipped so the render faces the same way as the
source.

A **neck** cylinder runs from the shoulder midpoint to the head, and the head
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
