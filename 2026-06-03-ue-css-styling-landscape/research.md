# UE CSS/Styling Landscape Investigation

Date: 2026-06-03

Scope: `/Users/rayelo/G2/my-g2/ue`

This is a running research note for the styling landscape in the UE Rails app. It focuses on differentiable styling solutions, setup/configuration files, breadth of use, and risks of changing styling layers.

## Working Notes

- The `ue/` repo was inspected read-only.
- Pre-existing untracked files were present before this investigation:
  - `webpack/assets/javascripts/util/react_mount.js`
  - `webpack/spec/util/react_mount.spec.js`
- Generated investigation artifacts are kept outside `ue/` under this directory.

## Questions Being Answered

1. How many differentiable styling solutions are in use?
2. How widely are they used?
3. Which setup files matter?
4. What are the risks of switching styling layers?

## Tailwind v4 Objective Clarification

The Tailwind goal is not to replace every styling layer. The implementation objective is Tailwind v4 support in a way that is not disruptive to UE's current Nessy/Foundation/Sass setup.

The spike should compare:

1. Elevate-first Tailwind v4.
2. A common Tailwind 3.4.x stepping stone between main UE and Elevate.
3. A scoped Tailwind v4 bundle for migrated surfaces.
4. Direct main UE Tailwind v4 only if prototype evidence shows the Sass/Gulp pipeline can support it safely.
