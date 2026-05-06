# rperf-remote

Remote rPerf estimator for AIX LPARs — a thin-remote adaptation of [Nigel Griffiths' `rperf` script](https://github.com/nigelargriffiths/rperf) that runs from a central jumphost against any LPAR over SSH, with no installation required on the target.

---

## What it does

Reports the **rPerf** (Relative Performance) figure for an AIX LPAR — IBM's published SPEC-derived performance number for Power Systems hardware. Useful for capacity planning, hardware refresh sizing, and fleet-wide baselining.

For each target LPAR, it produces a line like:

```
trips-testlab 20.23 rPerf estimated based on 1.00 Virtual CPU cores
trips-testlab 20.23 rPerf estimated based on 1.00 Uncapped Entitlement CPU cores
```

The number isn't measured live — it's a lookup against IBM's published Power Systems Facts and Features data, scaled to the LPAR's visible CPU count. Same methodology as the upstream script, just executed remotely.

---

## Why this exists

Upstream `rperf` is a single ksh93 script you have to copy to and execute on each LPAR you want to query. That works fine for a handful of boxes but becomes tedious across a real fleet — copying scripts around, dealing with version drift between hosts, getting permissions right per LPAR.

This adaptation reverses the model: the script runs once on a central jumphost (Linux or AIX), SSHes to each LPAR to gather only the raw discovery data (`lsattr`, `lsdev`, `lparstat`), then performs all the rounding, lookup, and scaling locally. The lookup table is updated in one place — no per-host script distribution.

---

## Architecture

```
┌────────────────────┐         ┌─────────────────┐
│  Linux jumphost    │  ssh    │  AIX LPAR       │
│  ──────────────    │ ──────► │  ──────────     │
│  rperf-remote      │         │  lsattr -El...  │
│  rperf.table       │ ◄────── │  lsdev -Cc...   │
│  (all the logic)   │ KEY=VAL │  lparstat -i    │
└────────────────────┘         └─────────────────┘
        ksh93                       stock AIX
                                  (no install)
```

**Two files on the jumphost:**

| File | Purpose |
|------|---------|
| `rperf-remote` | The wrapper — handles flags, SSH, parallelism, output formatting |
| `rperf.table`  | The IBM-published rating data, sourced at runtime |

**Target-side requirements (AIX LPAR):**
- SSH access with key-based auth (BatchMode-friendly)
- Stock AIX commands: `lsattr`, `lsdev`, `lparstat`, `hostname`
- Nothing installed, nothing copied

---

## Prerequisites

### On the jumphost

- **ksh93 installed** — AT&T `ksh93u+m`. On Ubuntu install via `apt install ksh` (the offline `.deb` is `ksh93u+m_<version>_amd64.deb` from the universe pool). Verify with `ksh --version` — should report `Version AJM 93u+m/...`.
- **OpenSSH 7.6+** for `StrictHostKeyChecking=accept-new` support. Older versions: edit the wrapper to use `StrictHostKeyChecking=no` instead, or pre-populate `~/.ssh/known_hosts` via `ssh-keyscan`.
- **An SSH key** registered in the target user's `~/.ssh/authorized_keys` on each LPAR. Passphraseless (or loaded into `ssh-agent`) — the wrapper uses `BatchMode=yes` and won't tolerate prompts.

### On each AIX LPAR

Just SSH access. Any user with shell access can run `lsattr`/`lsdev`/`lparstat` — no special privileges needed.

---

## Setup

### 1. Drop the files on the jumphost

```bash
mkdir -p ~/jacks-stuff/rperf
cd ~/jacks-stuff/rperf
# Place rperf-remote and rperf.table in this directory
chmod +x rperf-remote
```

The two files must live in the same directory (the wrapper auto-discovers `rperf.table` next to itself). Override with `-t /path/to/table` if needed.

### 2. Set up SSH key authentication

If you don't already have a key for the user that'll run the wrapper:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C "$(whoami)@$(hostname)-rperf"
```

Push it to each LPAR (this is the one and only time you'll be prompted for that LPAR's password):

```bash
ssh-copy-id user@lpar01
# or manually if ssh-copy-id isn't available:
cat ~/.ssh/id_ed25519.pub | ssh user@lpar01 \
  'mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
   cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

Verify:

```bash
ssh -o BatchMode=yes user@lpar01 'lsattr -El sys0 -a modelname -F value'
```

If that prints the MTM (e.g. `IBM,9009-41A`) without prompting, the wrapper will work for that host.

### 3. Build a fleet.list

Plain text file, one host per line. Hostnames or IPs both work. `#` for comments (full-line or inline).

```
# Production AIX fleet
192.168.131.135      # trips-testlab
aix-prod-01.example.com
aix-prod-02.example.com

# Development
aix-dev-01
aix-dev-02
```

**Tip — generate from TSM node data:**
```bash
dsmadmc -id=admin -password=xxx -dataonly=yes \
  "select node_name from nodes where platform_name='AIX'" \
  | awk '{print tolower($1)}' > fleet.list
```

This is the most authoritative source — it's literally every AIX box being backed up.

---

## Usage

### Quick reference

```
rperf-remote [-evhHc] [-u user] [-t table] [-f hostfile]
             [-P parallel] [host ...]
rperf-remote [-evhHc] [-u user] [-t table] -        # read hosts from stdin
```

| Flag | Meaning |
|------|---------|
| `-v` | Verbose (also enables `-e`); shows discovery values, lookup key, matchup args |
| `-e` | Append entitlement-adjusted rating line per host |
| `-h` | Prefix each output line with the LPAR's short hostname |
| `-c` | CSV output mode (header + one row per host, machine-readable) |
| `-j` | JSON output mode (array of objects, properly typed for `jq`/Python/etc.) |
| `-H` | Help |
| `-u user` | SSH username (default: current user) |
| `-t table` | Path to `rperf.table` (default: alongside the wrapper) |
| `-f file` | Read hosts from file (one per line, `#` comments allowed) |
| `-P n` | Parallel SSH workers (default: 8, max: 64, set 1 to disable parallelism) |
| `-` | Read hosts from stdin |

### Examples

**Single host, default output:**
```bash
ksh rperf-remote trips-testlab
```

**Verbose with hostname prefix and entitlement (best for diagnosing one host):**
```bash
ksh rperf-remote -veh trips-testlab
```

**Fleet sweep, 16 parallel workers, hostname-prefixed output:**
```bash
ksh rperf-remote -h -P 16 -u root -f fleet.list
```

**CSV baseline for tracker import:**
```bash
ksh rperf-remote -c -P 16 -u root -f fleet.list > rperf-baseline-$(date +%Y%m%d).csv
# Produces: rperf-baseline-20260506.csv in current working directory
```

**View the CSV nicely formatted in the terminal:**
```bash
column -s, -t < rperf-baseline-20260506.csv | less -#2 -N -S
```

That `column`/`less` combo is the right way to read CSVs at the terminal:
- `column -s, -t` — auto-aligns columns by comma separator
- `less -#2` — left/right scroll moves 2 columns at a time
- `less -N` — show line numbers
- `less -S` — chop long lines instead of wrapping (essential — without `-S` rows wrap and become unreadable)

**Pipe a host list from elsewhere:**
```bash
grep '^aix-' /etc/hosts | awk '{print $2}' | ksh rperf-remote -h -
```

**Combine sources:**
```bash
echo "extra-lpar" | ksh rperf-remote -h -f fleet.list - critical-lpar
# Reads fleet.list AND stdin AND uses critical-lpar from args
```

---

## Output formats

### Default (human-readable)

```
trips-testlab 20.23 rPerf estimated based on 1.00 Virtual CPU cores
trips-testlab 20.23 rPerf estimated based on 1.00 Uncapped Entitlement CPU cores
```

Identical format to upstream `rperf`, with optional hostname prefix from `-h`. Best for ad-hoc queries and copy-pasting into tickets/runbooks.

### Verbose (`-v`)

Shows the full discovery + lookup trace:

```
Information is from public documents from www.ibm.com
-- - IBM Power Systems Performance Report and Archive
-- - rperf-remote Version:1.1   Table:/home/bcadmin/jacks-stuff/rperf/rperf.table
Host=trips-testlab Machine=IBM,9009-41A MHz=2300 Rounded-MHz=2300 CPUs=1 CPU-Type=PowerPC_POWER9
lookup IBM,9009-41A_2300_1
matchup 4 80.9 6 118.6
calculate cpus=1 from 4 80.9
trips-testlab 20.23 rPerf estimated based on 1.00 Virtual CPU cores
trips-testlab 20.23 rPerf estimated based on 1.00 Uncapped Entitlement CPU cores
```

Use this when:
- A host returns `unknown unknown` (you'll see exactly which lookup key didn't match)
- You're not sure whether the rating is official or extrapolated
- You're debugging why a value differs from expectation

### CSV (`-c`)

16 columns, RFC 4180 compliant (values containing commas are quoted):

```
timestamp,host,resolved_hostname,machine,mhz_raw,mhz_rounded,cpus,proctype,lookup_key,rating,units,estimated,mode,ent_cpu,ent_rating,status
2026-05-06T09:14:22Z,trips-testlab,trips-testlab,"IBM,9009-41A",2300000000,2300,1,PowerPC_POWER9,"IBM,9009-41A_2300_1",20.23,rPerf,estimated,Uncapped,1.00,20.23,ok
```

| Column | Description |
|--------|-------------|
| `timestamp` | UTC ISO-8601 of the run |
| `host` | As you passed it on the command line / in fleet.list |
| `resolved_hostname` | What the LPAR reports as `hostname -s` |
| `machine` | MTM (e.g. `IBM,9080-HEU`) |
| `mhz_raw` | Frequency in Hz from `lsattr` |
| `mhz_rounded` | Rounded MHz used for lookup |
| `cpus` | Visible Virtual CPUs |
| `proctype` | e.g. `PowerPC_POWER9`, `POWER10` |
| `lookup_key` | The `MTM_MHz_CPUs` string the script searched for |
| `rating` | rPerf or rOLTP figure, or `unknown` if no table match |
| `units` | `rPerf`, `roltp`, or `unknown` |
| `estimated` | `official` (exact match) or `estimated` (linearly scaled) |
| `mode` | `Capped` or `Uncapped` from `lparstat -i` |
| `ent_cpu` | Entitled Capacity (decimal CPU cores) |
| `ent_rating` | Rating × (ent_cpu / cpus), rounded to 2dp |
| `status` | `ok`, `unknown_mtm`, `discovery_fail`, `ssh_fail` |

Failed hosts still get a row so you can see what was attempted. Filter for problems:
```bash
awk -F, '$NF != "ok" {print}' rperf-baseline-*.csv
```

---

### JSON (`-j`)

Array of objects, one per host, with proper typing (numbers as numbers, `null` for missing values). Same 16 fields as CSV. Designed for `jq` pipelines and programmatic consumption.

```json
[
    {"timestamp": "2026-05-06T09:51:08Z", "host": "p10-prod-01", "resolved_hostname": "p10-prod-01", "machine": "IBM,9080-HEX", "mhz_raw": 3650000000, "mhz_rounded": 3650, "cpus": 40, "proctype": "POWER10", "lookup_key": "IBM,9080-HEX_3650_40", "rating": 1366.80, "units": "rPerf", "estimated": "official", "mode": "Uncapped", "ent_cpu": 10.00, "ent_rating": 341.70, "status": "ok"},
    {"timestamp": "2026-05-06T09:51:08Z", "host": "dead-host", "resolved_hostname": null, "machine": null, "mhz_raw": null, "mhz_rounded": null, "cpus": null, "proctype": null, "lookup_key": null, "rating": null, "units": null, "estimated": null, "mode": null, "ent_cpu": null, "ent_rating": null, "status": "ssh_fail"}
]
```

**Use this when** you want to pipe results into `jq`, a Python script, or anything that prefers structured data over CSV. Examples:

```bash
# Top 5 LPARs by rating
ksh rperf-remote -j -P 16 -f fleet.list \
  | jq -r '.[] | select(.status=="ok") | "\(.rating)\t\(.host)"' \
  | sort -rn | head -5

# Hosts that failed
ksh rperf-remote -j -P 16 -f fleet.list \
  | jq -r '.[] | select(.status != "ok") | "\(.status)\t\(.host)"'

# Aggregate fleet rPerf
ksh rperf-remote -j -P 16 -f fleet.list \
  | jq '[.[] | select(.status=="ok") | .rating] | add'

# Hosts with estimated (vs official) ratings - candidates for table refinement
ksh rperf-remote -j -P 16 -f fleet.list \
  | jq -r '.[] | select(.estimated == "estimated") | "\(.host)\t\(.machine)\t\(.cpus)-core"'
```

The three output modes (`-c`, `-j`, `-v`) are mutually exclusive — pick the one that matches what'll consume the output.

---

## Status values

| Status | Meaning | What to do |
|--------|---------|------------|
| `ok` | Discovered cleanly, lookup matched | Use the rating |
| `unknown_mtm` | LPAR responded but its MTM isn't in `rperf.table` | Add an entry to `rperf.table` (see "Maintenance" below) |
| `discovery_fail` | SSH worked but `lsattr`/`lsdev` returned nothing | Check user permissions on the LPAR; rare |
| `ssh_fail` | SSH itself failed | Check connectivity, keys, BatchMode-incompatible auth |

---

## Maintenance

### Adding new Power generations to `rperf.table`

When IBM releases new hardware, add new arms to the `case` statement in `rperf.table`. The pattern is:

```ksh
IBM,<MTM>_<MHz>_*) matchup <cpus1> <rating1> <cpus2> <rating2> ... ;;
```

For example, if IBM publishes a new Power11 model `IBM,9999-XYZ` at 4.0 GHz with rPerf ratings of 500 (16-core), 1000 (32-core), and 1900 (64-core):

```ksh
IBM,9999-XYZ_40*_*) matchup 16 500.0 32 1000.0 64 1900.0 ;;
```

Source data: search "IBM Power Systems Facts and Features" for the latest PDF — IBM publishes updated tables with each new generation.

The wrapper script doesn't need touching for table updates — only `rperf.table`.

### Updating the wrapper

The wrapper is the same on every jumphost. To roll an update:
1. Test on one box first
2. `scp` the new `rperf-remote` to each jumphost
3. No restart, no service touch — next invocation picks it up

---

## Troubleshooting

### `Permission denied (publickey,password,keyboard-interactive)`

You don't have a working SSH key for the user the wrapper is trying to use. Run with verbose ssh to confirm:
```bash
ssh -vvv -o BatchMode=yes user@lpar01 'echo OK' 2>&1 | tail -30
```
Look for `agent contains no identities` or `no such identity` lines — those tell you keys aren't loaded.

Fix:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""    # if no key exists
ssh-copy-id user@lpar01                              # push to LPAR
```

### `discovery failed (incomplete data from target)`

SSH worked but the target didn't return all expected values. Most likely `lsattr` or `lparstat` isn't available to the user, or `/usr/sbin` isn't in their PATH. Run manually to see:
```bash
ssh user@lpar01 'lsattr -El sys0 -a modelname -F value; lparstat -i | head -5'
```

### `unknown unknown - no table entry for IBM,XXXX-YYY_zzzz_n`

The MTM/MHz/CPU combination isn't in `rperf.table`. Either:
- The hardware is newer than the table (add an entry — see Maintenance above)
- The CPU count is higher than any published config for that MTM (the script doesn't extrapolate upward — only downward from the highest published config)
- An obscure model IBM never published rPerf for

Run with `-v` to see the exact lookup key and decide.

### Sequential when I asked for `-P 16`

Check `wait -n` support on your ksh:
```bash
( true ) & wait -n 2>&1 ; echo "rc=$?"
```
If `rc=2` or you see "unknown option", your ksh doesn't support `wait -n` and the wrapper falls back to **batch parallelism** — runs 16 in parallel, waits for all, runs the next 16, etc. Functionally equivalent for homogeneous fleets, slightly less efficient when one host is much slower than others.

### CSV columns look misaligned in `column -t`

`column -t` doesn't handle quoted CSV — it just splits on the separator. Internal commas in `"IBM,9080-HEX"` get treated as field separators. Use a real CSV parser if you need precision:
```bash
python3 -c "
import csv, sys
for row in csv.reader(open(sys.argv[1])):
    print(' | '.join(f'{c:<20}' for c in row[:5]))
" rperf-baseline-20260506.csv
```

For visual scanning, `column -s, -t < file | less -S` is good enough — the misalignment is cosmetic.

---

## Notes on accuracy

The rPerf figure is IBM's published number, scaled linearly when your LPAR's CPU count is below the smallest published configuration. Caveats inherited from upstream:

- **`estimated` flag** — when the script linearly scales down (e.g. you have a 1-core LPAR but IBM only published 4/8/16-core ratings), it flags the result as estimated. The linear model slightly under-estimates because real SMP scaling isn't perfectly linear.
- **Shared-CPU LPARs** — uses VP count as the cap, ignoring fractional entitlement for the main rating line. The `-e` line accounts for entitlement separately.
- **Rounding artefacts** — small differences (e.g. 30.19 vs 30.20) between this and upstream are printf rounding behaviour in different libc versions, not logic differences.

For capacity planning, treat the figure as ±5% accurate. For hardware refresh sizing where you're comparing within the same Power generation, accuracy is generally better than that.

---

## Common workflows

### Quarterly fleet baseline

```bash
cd ~/jacks-stuff/rperf
ksh rperf-remote -c -P 16 -u root -f fleet.list \
    > rperf-baseline-$(date +%Y%m%d).csv

# Or as JSON for jq-based downstream processing
ksh rperf-remote -j -P 16 -u root -f fleet.list \
    > rperf-baseline-$(date +%Y%m%d).json

# Optional: keep last 8 quarters, prune older
find . -name 'rperf-baseline-*.csv' -mtime +730 -delete
```

### Compare two baselines

```bash
diff <(sort rperf-baseline-20260206.csv) <(sort rperf-baseline-20260506.csv) | less
```

Or for just the rating column:
```bash
join -t, -1 2 -2 2 -o 1.2,1.10,2.10 \
    <(sort -t, -k2 rperf-baseline-20260206.csv) \
    <(sort -t, -k2 rperf-baseline-20260506.csv) \
    | awk -F, '$2 != $3 {print}'
```

### Find LPARs that need attention

```bash
# Hosts that failed entirely
awk -F, '$NF == "ssh_fail" {print $2}' rperf-baseline-*.csv

# Hardware not in our lookup table (might be new gear or renamed MTM)
awk -F, '$NF == "unknown_mtm" {print $2, $4}' rperf-baseline-*.csv

# Estimated (not official) ratings - candidates for table refinement
awk -F, '$12 == "estimated" {print $2, $4, $7"-core"}' rperf-baseline-*.csv
```

### Sum total rPerf across the fleet

```bash
awk -F, 'NR>1 && $10 != "unknown" && $10 != "" {sum += $10} END {printf "Total fleet rPerf: %.2f\n", sum}' \
    rperf-baseline-*.csv
```

Useful for capacity reviews — gives you a single "how much compute do we have" number to track quarter-over-quarter.

---

## Credits and source data

- Upstream `rperf` script by Nigel Griffiths (`nigelargriffiths@hotmail.com`) — https://github.com/nigelargriffiths/rperf
- Lookup table data sourced from IBM Power Systems Facts and Features documents
- This wrapper preserves the upstream maths and lookup logic byte-for-byte; the only changes are the SSH transport, parallelism, CSV output, and externalised lookup table

---

## Version history

| Version | Changes |
|---------|---------|
| 1.0 | Initial thin-remote wrapper, single/multi/file/stdin host input, SSH key auth, externalised `rperf.table` |
| 1.1 | Added `-P` parallelism (default 8 workers), `-c` CSV output mode, deterministic input-order output |
| 1.2 | Added `-j` JSON output mode (array of properly typed objects, `null` for missing fields) |
