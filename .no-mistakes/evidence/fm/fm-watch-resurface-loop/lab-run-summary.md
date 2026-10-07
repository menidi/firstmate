# Live lab: Claude 2.1.292 primary + supervision host (disposable fm-lab home)

Setup: `bin/fm-lab-home.sh create`, `config/supervision-host`, one fake task `demo` (meta + status, a `sleep` window `fm-demo` on the lab tmux socket), real `claude` started on `tmux -L fm-lab` from the run worktree with `FM_HOME=<lab>`.
Lab workarounds: a temporary gitignored `.fm-secondmate-home` in the worktree (linked worktrees fail the primary-scope check), and `state/.lock` written with the lab Claude pid (session start stands down inside a gate worktree). Both were removed at teardown.

## Fixed code (HEAD 96c0c99)
- 1791392881 `pass-through attended main-only signal: demo.status`. The successor arm 19176 ran in **pgid 19176**, and the hook's group was 98048.
- The hook group 98048 was gone by t+45s, while the successor arm/watcher 19176/19212 were still alive.
- Main drained and acked. 1791392889: the next park's `take-over arm=19176` produced `taken-over`, then a fresh watcher 22288. No `check: rearm-resurface` followed. The marker reads `acked:handling:*`.
- See lab-fixed-supervision-host.log, lab-fixed-watch-cycle-exits.log, and lab-fixed-primary-pane.txt.

## Lab artifact (not the reported bug)
While the lab main had been told to run no tools, it never ran `bin/fm-wake-drain.sh` or the ack. Each take-over restored `announced:downtime`, so a `check: rearm-resurface` came every ~5s. This is the durable re-handling the rewake text describes.
It stopped as soon as main drained and acked (lab-all-supervision-host.log, 1791392544..1791392723).

## Base code (ac0811c host script, swapped in by atomic rename)
- 1791393041 and 1791393269: main-only pass-throughs. Each successor (arm 45622, arm 76811) stayed in the **hook's pgid 47715**.
- In both runs (main turns of ~9s and ~14s), Claude did not tear down the hook's group. The next park took the successor over normally, and no rearm-resurface followed.
- So the root-cause trigger (Claude TERMing the hook's process group after the exit-2 rewake) did not reproduce on this Claude build in the lab. The red/green fixture test models it.
