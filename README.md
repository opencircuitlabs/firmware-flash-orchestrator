# Firmware Flash Orchestrator

Geometry-aware, verified, resumable firmware programming for embedded targets.

The orchestrator converts sparse firmware segments into aligned program pages and erase blocks, checks target boundaries and protected regions, retries transient adapter failures, verifies every written page, emits progress/checkpoints, validates device identity, supports cancellation, and always closes the programmer.

It consumes the normalized image model produced by [`firmware-image-toolkit`](https://github.com/opencircuitlabs/firmware-image-toolkit) while remaining dependency-free.

## Quick start

```bash
npm install
npm test
node src/cli.js plan examples/image.json examples/target.json plan.json
node src/cli.js simulate plan.json report.json
```

```js
import {
  planFlash, executeFlashPlan
} from '@opencircuitlabs/firmware-flash-orchestrator';

const plan = planFlash(image, target, {
  verify: true,
  erase: true,
  reset: true,
  retries: 1
});

const report = await executeFlashPlan(plan, programmerAdapter, {
  signal,
  resume: previousCheckpoint,
  onProgress(event) {},
  onCheckpoint(checkpoint) {}
});
```

## Target geometry

A target defines `baseAddress`, total `size`, `pageSize`, `eraseSize`, erased byte value, optional `deviceId`, and protected address ranges. Page size must divide erase size, and erase size must divide target size.

Planning rejects:

- images outside the target address space;
- overlapping source segments;
- direct writes into protected ranges;
- erase blocks whose collateral erase would touch a protected range;
- malformed or inconsistent geometry.

Sparse bytes inside a touched page are padded with the target's erased value and reported as a warning. The plan records payload bytes separately from actual programmed bytes.

## Adapter contract

Programmer adapters implement `open`, `identify`, `erase`, `write`, `read`, and `close`; `reset` is optional. Each operation receives an abort signal where applicable. The included `MockFlashAdapter` models address bounds, erase/write alignment, injected transient failures, read corruption, identity, and memory contents.

## Verification and resume

Verification compares exact readback bytes after every page. A page is checkpointed only after it passes. Resume state is bound to both the full sparse-image hash and individual page hashes.

Flash erase granularity is handled safely: if any page in an erase block is incomplete, that block is erased and every planned page within it is rewritten—even pages present in the checkpoint. Fully completed erase blocks are skipped.

## Safety notes

The orchestrator does not infer target geometry or bypass protected memory. Production adapters should additionally enforce voltage, debug-lock, and family-specific programming constraints. Resume checkpoints should be stored only after the target remains connected and powered consistently.

## License

MIT
