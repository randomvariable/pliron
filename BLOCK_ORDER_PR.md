# Cache same-block operation positions for dominance queries

## Problem

`strictly_precedes_in_block` walks forward from its first operation on every query. SSA verification asks this question for operand definitions repeatedly, so a large straight-line block can take quadratic time even when its operation and edge counts grow linearly. A production AMDGPU compiler profile attributed 99.31% of sampled CPU time to this walk.

## Change

Store a lazy operation-to-position index in the block's operation list. The first same-block order query indexes the block; later queries compare two hash-map positions. Every intrusive-list insertion or unlink invalidates the containing block's index, including interior insertions that do not change the list head or tail. Moving an operation therefore invalidates both source and destination blocks. Unlinked operations, identical operations and operations in different blocks still return false.

The cache is owned by its block and does not survive block destruction or operation-list mutation. Verification rules and their callers are unchanged.

## Regression coverage

The unit regression counts operations visited while indexing rather than asserting a wall-clock ratio: 128, 256, 512 and 1024 operation blocks each visit exactly their own operation count despite repeated forward and reverse queries. A second test warms the index then exercises append, interior insertion, move, front/back insertion, cross-block movement and unlink.

## Validation

- `LLVM_SYS_231_PREFIX=/usr cargo test -p pliron --lib basic_block::order_tests -- --nocapture`: 2 passed, 0 failed.
- Before the change, the captured attention program lowered and verified 205,742 IR operations in 1.089 s. The captured recurrent program did not finish its first lowering/verification within a conservative 690 s observation bound (the command was stopped after 700.06 s including startup); a 3 s CPU profile assigned all 163 samples to `strictly_precedes_in_block`.
- `LLVM_SYS_231_PREFIX=/usr cargo test -p pliron --lib`: all 273 core unit tests passed.
- After pinning the fix, the same attention capture lowered/verified in 0.691 s (205,742 operations); a second whole-tree verification took 0.628 s. The recurrent capture lowered/verified in 1.893 s (2,260,301 operations); a second whole-tree verification took 1.234 s. The recurrent first-pass observation improves by at least 364x against the conservative baseline bound without removing any verifier.
- Feature branch `fix/block-position-index` and integration branch `rv-main` are pushed at `3dd6ce7c8e65f82888ee1c62178af8ccddb6afc5`. No upstream pull request has been opened.
