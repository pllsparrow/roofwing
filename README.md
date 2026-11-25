# Roofwing · 檐雀

A small single-player browser game about being a sparrow among rooftops and quiet streets.

**Status — planning.** This repository currently contains the product requirements,
technical design and single-player MVP plan. The playable game is the next milestone.

## The first playable experience

Leave your nest, walk and take flight, glide between buildings, land on a roof, collect
three pieces of nesting material, and carry them home. Close the tab and continue later.
A compact low-poly neighborhood and responsive flight are the heart of the game.

## Project documents

- [Product requirements](docs/PRD.md)
- [Technical specification](docs/SPEC.md)
- [Single-player MVP and acceptance](docs/SINGLE_PLAYER_MVP.md)
- [Technology decisions and implementation order](docs/IMPLEMENTATION_PLAN.md)
- [Asset provenance](ASSETS.md)

The design documents are written in Chinese. Code comments and identifiers will use English.

## Selected technology

TypeScript, Vite, Three.js and Rapier. Small browser-local saves use localStorage.
Vitest will cover simulation rules and save validation. A desktop client is a later milestone after browser MVP acceptance. Desktop keyboard and mouse are the
first supported controls. The intended distribution is a static Web build, with GitHub Pages
as the initial playable-hosting target.

Dependency versions will be pinned in the first implementation commit and installed through
a committed lockfile. Build commands and a demo link will be added after that implementation
passes its checks. No playable release or benchmark result is claimed at this stage.

## Development milestones

1. Flight, landing and camera in a small collision test scene.
2. One neighborhood, nesting-material collection and a complete return-to-nest objective.
3. Save/reopen, pause, recovery and visual/audio polish.
4. Clean-clone build, measured performance and a playable public release.

See the MVP document for the concrete acceptance checklist.

## License

Original project code and original procedural assets are licensed under [MIT](LICENSE).
Third-party dependencies and any future imported assets retain their own licenses. Their
notices and provenance belong in [ASSETS.md](ASSETS.md). The project does not bundle source,
models, screenshots or audio from Sparrow Sim.
