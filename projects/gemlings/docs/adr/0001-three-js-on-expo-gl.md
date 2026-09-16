# ADR-0001: Render the world with three.js on expo-gl inside React Native

**Status:** Accepted

## Context

Gemlings needs a real-time 3D world (streamed terrain, dozens of creatures,
lighting, bloom) *and* a large amount of ordinary product surface: a journal,
settings, a tolerant save, screen-reader support, store plumbing. The
obvious split — a game engine for the world, something else for the UI —
means two codebases, two build systems, and two accessibility stories.

## Decision

Build the whole app in React Native (Expo 57) and render the world as one
component using three.js r170 on `expo-gl`, through a ~30-line canvas shim.
Do **not** use the community RN wrapper for three; its releases trail the
React and RN versions in use.

## Consequences

**Positive**

- One codebase, one build, one accessibility layer. The world is a component
  that reports events; the game rules and the save are plain RN.
- The paper UI (journal, sheets) is laid *over* the world, which stays
  mounted — a half-cracked stone stays half-cracked while you read.

**Negative — three shims, each found the hard way**

1. three ≥ r163 throws "WebGL 1 is not supported" on expo-gl. Its test is
   `context instanceof WebGLRenderingContext`; expo-gl defines that global
   and its WebGL2 context matches it. The global is hidden around the
   `WebGLRenderer` constructor and restored after.
2. `GLTFLoader` sniffs `navigator.userAgent`, which RN does not define, and
   decodes with `TextDecoder`, which Hermes may lack. Both are polyfilled
   before the first parse.
3. `fetch()` cannot read bundled Android resources, so GLB bytes and texture
   files come out of the APK through a small native module.

Post-processing (`EffectComposer` → `UnrealBloomPass` → `OutputPass`) works
on expo-gl; a GL error on the first frames drops the composer and renders
straight, so a device that cannot do float render targets still plays.
