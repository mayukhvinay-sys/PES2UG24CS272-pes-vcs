Q5.1
pes checkout <branch> needs to:
1. Resolve target commit: read .pes/refs/heads/<branch>.
2. Update HEAD to ref: refs/heads/<branch> (or direct hash for detached mode).
3. Read target commit object, then its root tree.
4. Reconstruct working directory to match that tree:
- Write tracked files from blobs
- Create/remove directories
- Delete tracked files not present in target tree
5. Rebuild/update index to match checked-out tree metadata.
Why complex:

- Must avoid clobbering uncommitted work.
- Needs recursive tree traversal and path-level conflict handling.
- Must be atomic enough that partial failures don’t leave mixed states.
Q5.2
Detect dirty-checkout conflict using index + object store:
1. For each tracked path that differs between current branch tree and target branch tree:
- Compare working file against index entry metadata (mtime, size) as fast filter.
- If metadata differs, hash current working file as blob candidate and compare with
indexed blob hash.
2. If working file content != indexed blob and path also changes across branches, block
checkout.
3. Also block when untracked working files would be overwritten by target tree paths.
Core idea: “local modification” = working content diverges from index; “checkout danger” =
same path diverges between current/target trees.
Q5.3
Detached HEAD means HEAD points directly to a commit hash, not a branch ref. New
commits
are created normally, but no branch name advances, so those commits can become
unreachable
later.
Recovery:
1. Find commit hash (from log, reflog-style history, or printed commit IDs).
2. Create/move a branch ref to it: e.g. write hash to .pes/refs/heads/recovered.
3. Set HEAD to ref: refs/heads/recovered to continue safely.
Q6.1
GC algorithm:
1. Build root set from all refs (.pes/refs/heads/*, tags if any, detached HEAD hash if
applicable).
2. Traverse graph (DFS/BFS):
- Commit -> parent commit(s)
- Commit -> root tree
- Tree -> child trees + blobs

3. Store visited hashes in a hash set (unordered_set-style) for O(1) membership.
4. Scan .pes/objects/**; delete any object hash not in reachable set.
For 100,000 commits and 50 branches, visited commits are usually near ~100,000 total
(branches overlap heavily), not 5,000,000. Total visited objects = commits + trees + blobs
reachable from those commits; typically many more than commits, but still linear in
reachable graph size.
Q6.2
Why dangerous concurrently:
- Commit process may write new objects and later update branch ref.
- If GC runs between those steps, new objects may appear unreachable (no ref yet) and get
deleted.
- Then commit finishes updating ref to hashes that GC already removed -> broken history/
reference.
Example race:
1. Commit writes tree/blob/commit objects.
2. Before ref update, GC marks reachability from current refs only.
3. GC deletes those newly written objects as unreachable.
4. Commit updates refs/heads/main to deleted commit hash.
How real Git avoids this:
- Uses object liveness protections (reflogs, grace periods, mtimes/prune windows).
- Uses locking/coordination and conservative pruning.
- Typically prunes only sufficiently old unreachable objects, reducing race risk
drastically.
