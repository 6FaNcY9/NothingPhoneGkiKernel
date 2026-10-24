# Task: turn ~/p/android-lab into a clean private git repo, then run the phone diagnosis

You are running inside `~/p/android-lab` on bandit-lab. direnv + devenv are active, so `adb`, `jadx`, `frida`, `git` and `gh` are on PATH. Work in two phases. Show me the plan for phase 1 before running `git add`.

## Phase 1: repository setup

1. **Inventory.** Run `du -sh * .[^.]* | sort -h` and `find . -size +5M -not -path './.devenv/*' -not -path './.direnv/*'`. Report every file over 5 MB and every directory over 50 MB.
2. **Secret scan before anything is staged.** Search the whole tree, including `rooting/`, `firmware/`, `notes/`, `tools/`, for:
   - `keybox.xml`, `*.keybox`, files containing `<Keybox` or `PRIVATE KEY`
   - `*.img`, `*.bin`, `*.zip`, `*.tar*`, `*.apk`, `*.so`
   - tokens: `grep -rIE '(ghp_|gho_|AIza|AKIA|xox[baprs]-|BEGIN (RSA|EC|OPENSSH) PRIVATE)'`
   - GSF IDs, IMEI or serial numbers in notes: `grep -rIE '\b[0-9a-f]{16}\b|\b[0-9]{15}\b' notes docs`
   List every hit with path and reason. Anything that is a key, keybox, boot image or firmware goes into `.gitignore`, not into the repo.
3. **Reorganise** (use `git mv` semantics only after init; before init plain `mv` is fine):
   - `mkdir -p analysis/logs` and move `androguard.log` and `callgraph.gml` there.
   - Move `TWRP_A53/` and `TWRP_A56/` under `firmware/twrp/`.
   - Create `firmware/README.md` listing every image with `sha256sum` and where it came from, so the images themselves can stay out of git.
   - `mkdir -p kernel diag prompts` and `ln -s AGENTS.md CLAUDE.md`.
4. **.gitignore.** Replace the existing one with entries for: `.devenv/`, `.direnv/`, `firmware/**` except `firmware/README.md`, `kernel/`, `analysis/logs/`, `diag/*` except `diag/*.md`, `*.img`, `*.bin`, `*.zip`, `*.apk`, `*.so`, `*.log`, `keybox*`, `*.keybox`, `rooting/**/*.xml`, `.env`, `result`, `result-*`.
5. **Init and first commit.** `git init -b main`, `git add -A`, then `git status` and show me the staged list. Only after I say OK: commit as "android-lab: initial import of devenv, notes, tools and docs". Then create a **private** GitHub repo with `gh repo create android-lab --private --source . --push`. If `gh` is not authenticated, stop and tell me.
6. Clone the kernel repo into the ignored folder: `git clone -b ccr-2778011e-r85xh2 https://github.com/6FaNcY9/NothingPhoneGkiKernel.git kernel/NothingPhoneGkiKernel`.

## Phase 2: Google sign-in diagnosis

Copy `kernel/NothingPhoneGkiKernel/docs/prompt-google-signin-diagnosis.md` to `prompts/` and follow it exactly. It is read-only against the phone. Its `diag/` output goes into `~/p/android-lab/diag/`. Raw logs stay local; only `diag/REPORT.md` is committed. When the report is written, commit it to android-lab and push, then print the report.

## Rules for both phases

- Never force-push, never delete directories, never `rm` anything larger than a log file without asking.
- Never stage a file the secret scan flagged, even if I seem to ask for it later in a hurry; ask me to confirm by name.
- Keep devenv.nix, devenv.yaml, devenv.lock and .envrc unchanged unless I ask.
