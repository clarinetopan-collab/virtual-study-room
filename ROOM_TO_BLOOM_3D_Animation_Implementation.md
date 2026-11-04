# ROOM TO BLOOM — 3D ANIMATION IMPLEMENTATION ADDENDUM FOR MANUS

## Instructions to Manus
Continue editing the EXISTING deployed project; do not recreate it. First create a checkpoint. These snippets are implementation starters to integrate into the actual project, not a claim that the website has been modified. Preserve all uploaded PNGs and GLBs. I inspected the GLBs: the clock has separate `Second Hand`, `Minute Hand`, `Hour Hand`; the CD model has `THICK 3D CD - physical raised disc`; the planning notebook has `Top Royal Blue Hardcover`, `Spine Fabric Ridge`, `Rounded Blue Book Spine`, and individual right-page line meshes. Use these exact mesh names and verify their local axes/pivots in the rendered model. The models may have baked transforms and do not necessarily have usable animation rigs. Never rotate an arbitrary mesh around its center and call that a correct hinged animation. Build a pivot at the actual hinge, or use an added rigged overlay if required.

Install missing packages only: `npm install three @react-three/fiber @react-three/drei`.

## A. GLB inspection / mesh debugging
Create `src/3d/inspectGLB.js`:

```js
export function inspectGLB(root) {
  root.updateWorldMatrix(true, true);
  root.traverse(o => {
    if (!o.isMesh) return;
    console.info('[GLB]', o.name, {
      position: o.position.toArray(),
      rotation: o.rotation.toArray().slice(0,3),
      parent: o.parent?.name,
      geometry: o.geometry?.type
    });
  });
}
export function findNamedMesh(root, name) {
  let result = null;
  root.traverse(o => { if (o.isMesh && o.name === name) result = o; });
  return result;
}
```

## B. Analog clock — animate actual hand meshes
Create `src/3d/AnimatedClock.jsx`:

```jsx
import { useEffect, useMemo, useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';
import { MathUtils } from 'three';
import { findNamedMesh } from './inspectGLB';

// remainingSeconds is supplied by the single shared, timestamp-based timer store.
export default function AnimatedClock({ remainingSeconds = 1500, durationSeconds = 1500, running = false, ...props }) {
  const { scene } = useGLTF('/models/alarm_clock(1).glb');
  const root = useMemo(() => scene.clone(true), [scene]);
  const hands = useMemo(() => ({
    second: findNamedMesh(root, 'Second Hand'),
    minute: findNamedMesh(root, 'Minute Hand'),
    hour: findNamedMesh(root, 'Hour Hand'),
  }), [root]);
  const base = useRef(null);
  useEffect(() => {
    base.current = Object.fromEntries(Object.entries(hands).map(([k, o]) => [k, o?.rotation.z ?? 0]));
    for (const [name, hand] of Object.entries(hands)) if (!hand) console.warn('Missing clock hand:', name);
  }, [hands]);
  useFrame((_, delta) => {
    if (!base.current) return;
    // Clock face normal and hinge pivot MUST be verified for this specific GLB.
    // Assumes face is in XY plane and each hand rotates around its clock-center pivot.
    const elapsed = Math.max(0, durationSeconds - remainingSeconds);
    const angles = {
      second: -2 * Math.PI * ((elapsed % 60) / 60),
      minute: -2 * Math.PI * ((elapsed % 3600) / 3600),
      hour: -2 * Math.PI * ((elapsed % 43200) / 43200),
    };
    Object.entries(hands).forEach(([name, mesh]) => {
      if (!mesh) return;
      const target = base.current[name] + angles[name];
      mesh.rotation.z = MathUtils.damp(mesh.rotation.z, target, 12, delta);
    });
  });
  return <primitive object={root} {...props} />;
}
```

**Mandatory correction:** If the hand meshes have origins away from the clock center, reparent each to a `THREE.Group` positioned at the clock center while preserving world transform. Determine the clock face axis and clockwise sign by inspecting the GLB. The code above only works directly if those pivots/axes are correct. Use `remainingSeconds` from the shared timer, never a second timer inside the model.

## C. CD player — rotate the actual CD only while audio is playing
Create `src/3d/AnimatedCD.jsx`:

```jsx
import { useEffect, useMemo, useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';
import { findNamedMesh } from './inspectGLB';

export default function AnimatedCD({ isPlaying, ...props }) {
  const { scene } = useGLTF('/models/CD(2).glb');
  const root = useMemo(() => scene.clone(true), [scene]);
  const disc = useMemo(() => findNamedMesh(root, 'THICK 3D CD - physical raised disc'), [root]);
  const groupRef = useRef();
  useEffect(() => { if (!disc) console.warn('CD disc mesh missing'); }, [disc]);
  useFrame((_, dt) => {
    if (isPlaying && groupRef.current) groupRef.current.rotation.y += dt * Math.PI * 1.5;
  });
  // The real disc must be reparented into a centered pivot group by Manus.
  // A separate rotating label/decals must also follow that pivot.
  return <group {...props}><primitive object={root} /><group ref={groupRef} /></group>;
}
```

**IMPORTANT: the empty pivot above is deliberately a placeholder for the rigging step; it does NOT yet spin the visible disc.** Manus MUST reparent the `THICK 3D CD - physical raised disc` mesh and all disc labels to a correctly located pivot. Use `attach`/world-transform preservation, inspect disc rotation axis, and verify visually that the visible disc spins. Audio must be controlled by a single HTMLAudioElement and user-initiated play; derive `isPlaying` from actual play/pause/ended events, not an unrelated toggle.

Example shared audio controller hook:

```jsx
import { useEffect, useRef, useState } from 'react';
export function useAmbientAudio(src) {
  const audio = useRef(null);
  const [playing, setPlaying] = useState(false);
  useEffect(() => {
    const a = new Audio(src); a.loop = true; audio.current = a;
    const onPlay = () => setPlaying(true), onPause = () => setPlaying(false);
    a.addEventListener('play', onPlay); a.addEventListener('pause', onPause);
    return () => { a.pause(); a.removeEventListener('play', onPlay); a.removeEventListener('pause', onPause); audio.current = null; };
  }, [src]);
  return {
    playing,
    play: async () => { try { await audio.current?.play(); } catch (e) { console.warn('Audio playback blocked:', e); } },
    pause: () => audio.current?.pause(),
    volume: v => { if (audio.current) audio.current.volume = Math.max(0, Math.min(1, v)); }
  };
}
```

## D. Planning notebook — physically hinged cover + page turning
Create `src/3d/AnimatedNotebook.jsx`:

```jsx
import { useMemo, useRef, useState } from 'react';
import { useFrame } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';
import { MathUtils } from 'three';
import { findNamedMesh } from './inspectGLB';

export default function AnimatedNotebook({ open = false, page = 0, onPageChange, ...props }) {
  const { scene } = useGLTF('/models/planning_book_3d(2).glb');
  const root = useMemo(() => scene.clone(true), [scene]);
  const cover = useMemo(() => findNamedMesh(root, 'Top Royal Blue Hardcover'), [root]);
  const coverPivot = useRef();
  const pagePivot = useRef();
  const [dragging, setDragging] = useState(false);
  useFrame((_, dt) => {
    if (coverPivot.current) coverPivot.current.rotation.y = MathUtils.damp(coverPivot.current.rotation.y, open ? -Math.PI * 0.85 : 0, 7, dt);
    if (pagePivot.current) pagePivot.current.rotation.y = MathUtils.damp(pagePivot.current.rotation.y, page > 0 ? -Math.PI : 0, 9, dt);
  });
  return (
    <group {...props}>
      <primitive object={root} />
      {/* Manus: place the cover mesh inside this pivot at the spine, preserving its world transform. */}
      <group ref={coverPivot} />
      {/* Manus: add a segmented curved page mesh or a properly rigged page here. */}
      <group ref={pagePivot}
        onPointerDown={e => { e.stopPropagation(); setDragging(true); }}
        onPointerUp={e => { e.stopPropagation(); if (dragging) onPageChange?.(page + 1); setDragging(false); }}
      />
    </group>
  );
}
```

**IMPORTANT: this is a wiring scaffold, not a finished opening animation.** Manus must physically move `Top Royal Blue Hardcover` under the spine pivot, create a segmented page with curved deformation, add pointer hit areas, and validate that page dragging works. A plain group with no geometry will not receive pointer events. For the text entry, show an accessible HTML form (Drei `<Html>` or a synchronized DOM overlay) over the open notebook; save tasks in the existing central store. Avoid drawing uneditable text onto the 3D mesh. Do not claim the notebook opens until the real GLB cover visibly rotates about its spine.

## E. Lamp and tea cup
Use actual `table lamp(1).glb` and `tea cup(1).glb`.
- Lamp: add a `pointLight` and change emissive intensity on the `Warm Inner Shade Diffuser` mesh; link brightness to the shared lighting slider. Crossfade the two provided Colourful Study Room images with an opacity transition. Calibrate their alignment before crossfading.
- Tea cup: attach 3–5 translucent steam ribbon/particle meshes above the cup, animate upward with sine-wave drift during active break, stop/hide when break ends. Steam is visual only; use a centralized timestamp-based break timer.

## F. Bin and task paper
Use `rubbish can(1).glb`. Compute a collision region using the actual model bounding box and manually calibrate an opening collider in the model's local space. During throw, move the paper ball along a parabolic path; convert its world position into the bin's local coordinates before testing collision. A successful landing must call the existing idempotent `completeSubtask(taskId, subtaskId)` action exactly once. A missed throw should remain retryable. Provide a keyboard-accessible Complete action. The paper tearing and crumpling effects should integrate the supplied React Bits sources, not be faked with status messages.

## G. Critical integration / acceptance criteria
1. Before implementation, inspect the GLB scene graph and identify which meshes move and which remain static. Verify rotation axes, pivots, geometry and materials.
2. Every visual animation must correspond to a real object, not an invisible or empty group.
3. Use one camera controller for all objects; do not reset focus, audio, or tasks when opening/closing.
4. GLB viewers must use clone instances to avoid shared scene parenting errors.
5. No permanent white hotspot buttons on the illustrated room; use invisible hit areas.
6. Test: visible notebook cover opens at its spine; page turns and editable tasks save; clock hands move with actual timer; visible CD spins only while audio plays; lamp changes light; cup steams during break; paper reaches bin and task completes once.
7. Report any model that cannot be animated properly without rigging or mesh reconstruction, and complete the necessary rigging before marking the feature done.
8. Preserve existing ROOM TO BLOOM features and continue in the existing Manus task/project.
