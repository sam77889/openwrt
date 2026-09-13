# VS010 Adblock Memory Safety Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prevent three-feed adblock reloads from exhausting VS010 RAM, preserve the last valid SmartDNS blocklist on failure, and add on-demand zram swap.

**Architecture:** Add source-testable shell helpers for strict memory admission, fatal worker propagation, safe temporary cleanup, and atomic publication. Keep processing in tmpfs, shorten raw-file lifetime, and add OpenWrt's maintained `zram-swap` package to the VS010 image.

**Tech Stack:** OpenWrt Makefiles, BusyBox `ash`, coreutils `sort`, `/proc/meminfo`, UCI, shell regression tests.

## Global Constraints

- Keep `adguard`, `adguard_tracking`, and `certpl`, plus SmartDNS and HTTPS DNS Proxy.
- Default `adb_memreserve` to 16 MiB.
- Calculate headroom as `MemAvailable + SwapFree`; malformed or missing values fail closed.
- Pre-sort checks require the reserve plus `adb_srtmem`; other checks require the reserve.
- Keep high-churn files on tmpfs, not NAND.
- Replace the active blocklist only after complete rendering and validation.
- Recursively remove only validated `adb_basedir/adblock.<pid>.<suffix>` directories.
- Preserve unrelated worktree changes. Do not stage the user-owned edits already in `target/linux/qualcommax/image/ipq50xx.mk`.
- Do not alter Ethernet, WAN negotiation, DTS port mapping, or WCSS reserved memory.

---

## File map

- `feeds/packages/net/adblock/files/adblock.sh`: all runtime safeguards and source-only test mode.
- `feeds/packages/net/adblock/files/adblock.conf`: 16 MiB default reserve.
- `feeds/packages/net/adblock/tests/test-memory-safety.sh`: deterministic shell regressions.
- `feeds/packages/net/adblock/Makefile`: release bump.
- `target/linux/qualcommax/image/ipq50xx.mk`: VS010 `zram-swap` selection.
- `target/linux/qualcommax/image/tests/test-vs010-packages.sh`: image package regression.

### Task 1: Test mode and deterministic memory admission

**Files:**
- Modify: `feeds/packages/net/adblock/files/adblock.sh:14-160,2584-2598`
- Modify: `feeds/packages/net/adblock/files/adblock.conf:1-12`
- Create: `feeds/packages/net/adblock/tests/test-memory-safety.sh`

**Interfaces:**
- Produces: `f_headroom()`, `f_memcheck(phase, extra_mib)`, `ADB_TEST_MODE=1`, and `ADB_MEMINFO`.

- [ ] **Step 1: Write the failing harness**

Create this executable file:

```sh
#!/usr/bin/busybox ash
set -eu
script_dir="$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)"
test_root="$(mktemp -d)"
trap 'rm -rf "${test_root}"' EXIT HUP INT TERM
failures=0

assert_success() { "$@" || failures=$((failures + 1)); }
assert_failure() { if "$@"; then failures=$((failures + 1)); fi; }
write_meminfo() {
	printf 'MemAvailable: %s kB\nSwapFree: %s kB\n' "$1" "$2" >"${test_root}/meminfo"
}

ADB_TEST_MODE=1 . "${script_dir}/../files/adblock.sh"
adb_awkcmd="$(command -v awk)"
adb_errorlog=/dev/null
adb_meminfo="${test_root}/meminfo"
adb_memreserve=16
adb_srtmem=8

write_meminfo 12288 0
assert_failure f_memcheck startup 0
write_meminfo 12288 8192
assert_success f_memcheck startup 0
write_meminfo 20480 0
assert_failure f_memcheck pre-sort 8
write_meminfo 24576 0
assert_success f_memcheck pre-sort 8
printf 'MemAvailable: invalid kB\nSwapFree: 8192 kB\n' >"${adb_meminfo}"
assert_failure f_memcheck startup 0
printf 'MemAvailable: 32768 kB\n' >"${adb_meminfo}"
assert_failure f_memcheck startup 0

[ "${failures}" -eq 0 ] || exit 1
printf 'ok - adblock memory safety\n'
```

- [ ] **Step 2: Prove the test fails**

Run `chmod +x feeds/packages/net/adblock/tests/test-memory-safety.sh && feeds/packages/net/adblock/tests/test-memory-safety.sh`.

Expected: non-zero because runtime setup executes or `f_memcheck` is undefined.

- [ ] **Step 3: Add strict memory helpers**

Add defaults beside `adb_srtmem`:

```sh
adb_memreserve="16"
adb_meminfo="${ADB_MEMINFO:-"/proc/meminfo"}"
```

Replace `f_mem()` and append these helpers:

```sh
f_mem() {
	local mem mode="${1}"
	if [ "${mode}" = "float" ]; then
		mem="$("${adb_awkcmd}" '$1=="MemAvailable:" && $2~/^[0-9]+$/ {printf "%.2f",$2/1024; found=1} END{if(!found)exit 1}' "${adb_meminfo}" 2>>"${adb_errorlog}")" || mem=""
	else
		mem="$("${adb_awkcmd}" '$1=="MemAvailable:" && $2~/^[0-9]+$/ {printf "%s",int($2/1024); found=1} END{if(!found)exit 1}' "${adb_meminfo}" 2>>"${adb_errorlog}")" || mem=""
	fi
	printf '%s' "${mem:-"0"}"
}

f_headroom() {
	"${adb_awkcmd}" '
		$1=="MemAvailable:" && $2~/^[0-9]+$/ {mem=$2; hm=1}
		$1=="SwapFree:" && $2~/^[0-9]+$/ {swap=$2; hs=1}
		END {if(!hm || !hs) exit 1; printf "%s",int((mem+swap)/1024)}
	' "${adb_meminfo}" 2>>"${adb_errorlog}"
}

f_memcheck() {
	local phase="${1}" extra="${2:-"0"}" observed required
	case "${adb_memreserve}" in ""|*[!0-9]*) adb_memreserve="16";; esac
	case "${extra}" in ""|*[!0-9]*) extra="0";; esac
	required="$((adb_memreserve + extra))"
	observed="$(f_headroom)" || observed=""
	if [ -z "${observed}" ] || [ "${observed}" -lt "${required}" ]; then
		f_log "info" "low-memory rejection, phase: ${phase}, required: ${required} MiB, observed: ${observed:-"invalid"} MiB"
		return 1
	fi
}
```

Before runtime-directory creation add:

```sh
if [ "${ADB_TEST_MODE:-"0"}" = "1" ]; then
	return 0 2>/dev/null || exit 0
fi
```

Add `option adb_memreserve '16'` after `adb_debug` in `adblock.conf`.

- [ ] **Step 4: Verify and commit**

Run:

```bash
sh -n feeds/packages/net/adblock/files/adblock.sh
sh -n feeds/packages/net/adblock/tests/test-memory-safety.sh
feeds/packages/net/adblock/tests/test-memory-safety.sh
```

Expected: zero; harness prints `ok - adblock memory safety`.

Commit only the three task files:

```bash
git add feeds/packages/net/adblock/files/adblock.sh feeds/packages/net/adblock/files/adblock.conf feeds/packages/net/adblock/tests/test-memory-safety.sh
git commit -m "test: cover adblock memory admission"
```

### Task 2: Safe run directories and fatal markers

**Files:**
- Modify: `feeds/packages/net/adblock/files/adblock.sh:641-663,1843-2110`
- Modify: `feeds/packages/net/adblock/tests/test-memory-safety.sh`

**Interfaces:**
- Produces: `f_tmpvalid(path)`, `f_stalecleanup()`, `f_failmark(phase, rc)`, `f_failed()`, `f_preparefeed()`, `f_waitcheck()`, and idempotent `f_rmtemp()`.

- [ ] **Step 1: Add failing marker and cleanup cases before the harness's final assertion**

```sh
adb_rmcmd="$(command -v rm)"
adb_basedir="${test_root}/base"
adb_rundir="${test_root}/run"
adb_pidfile="${adb_rundir}/adblock.pid"
mkdir -p "${adb_basedir}" "${adb_rundir}"
adb_tmpdir="${adb_basedir}/adblock.$$.current"
stale="${adb_basedir}/adblock.999999.stale"
unrelated="${adb_basedir}/tmp.keep"
mkdir -p "${adb_tmpdir}" "${stale}" "${unrelated}"
adb_failfile="${adb_tmpdir}/.fatal"
assert_success f_failmark pre-sort 137
assert_success f_failed
grep -qx 'pre-sort 137' "${adb_failfile}" || failures=$((failures + 1))
assert_success f_rmtemp
[ ! -e "${adb_tmpdir}" ] || failures=$((failures + 1))
[ -d "${stale}" ] || failures=$((failures + 1))
[ -d "${unrelated}" ] || failures=$((failures + 1))
assert_success f_stalecleanup
[ ! -e "${stale}" ] || failures=$((failures + 1))
[ -d "${unrelated}" ] || failures=$((failures + 1))

signal_base="${test_root}/signal-base"
signal_run="${test_root}/signal-run"
signal_record="${test_root}/signal-path"
mkdir -p "${signal_base}" "${signal_run}"
if /usr/bin/busybox ash -c '
	ADB_TEST_MODE=1 . "$1"
	adb_basedir="$2"; adb_rundir="$3"; adb_pidfile="$3/adblock.pid"
	adb_errorlog=/dev/null; adb_rmcmd="$(command -v rm)"; adb_cores=1; adb_srtmem=8
	f_temp
	printf "%s\n" "${adb_tmpdir}" >"$4"
	kill -TERM $$
' ash "${script_dir}/../files/adblock.sh" "${signal_base}" "${signal_run}" "${signal_record}"; then
	failures=$((failures + 1))
fi
signal_tmp="$(cat "${signal_record}")"
[ ! -e "${signal_tmp}" ] || failures=$((failures + 1))
```

- [ ] **Step 2: Prove the new test fails**

Run `feeds/packages/net/adblock/tests/test-memory-safety.sh`.

Expected: non-zero because `f_failmark` is undefined.

- [ ] **Step 3: Implement validated cleanup and marker helpers**

Replace the current temp helpers with:

```sh
f_tmpvalid() {
	local path="${1}" name pid suffix
	case "${path}" in "${adb_basedir}"/adblock.*.*);; *) return 1;; esac
	name="${path##*/}"; pid="${name#adblock.}"; pid="${pid%%.*}"
	suffix="${name#*.}"; suffix="${suffix#*.}"
	case "${pid}" in ""|*[!0-9]*) return 1;; esac
	case "${suffix}" in ""|*[!A-Za-z0-9]*) return 1;; esac
	[ "${path}" != "${adb_basedir}" ]
}

f_stalecleanup() {
	local dir name owner
	for dir in "${adb_basedir}"/adblock.*.*; do
		[ -d "${dir}" ] || continue
		f_tmpvalid "${dir}" || continue
		name="${dir##*/}"; owner="${name#adblock.}"; owner="${owner%%.*}"
		if ! kill -0 "${owner}" 2>/dev/null; then
			"${adb_rmcmd}" -rf -- "${dir}" 2>>"${adb_errorlog}" ||
				f_log "info" "temporary directory cleanup failed, path: ${dir}"
		fi
	done
}

f_failmark() {
	[ -n "${adb_failfile}" ] || return 1
	printf '%s %s\n' "${1}" "${2:-"1"}" >"${adb_failfile}"
}
f_failed() { [ -n "${adb_failfile}" ] && [ -s "${adb_failfile}" ]; }

f_preparefeed() {
	local prep_rc
	f_list prepare
	prep_rc="${?}"
	if [ "${prep_rc}" != "0" ]; then
		f_failmark "prepare:${src_name}" "${prep_rc}"
	fi
	return "${prep_rc}"
}

f_waitcheck() {
	! f_failed
}

f_temp() {
	if [ -d "${adb_basedir}" ]; then
		f_stalecleanup
		adb_tmpdir="$(mktemp -p "${adb_basedir}" -d "adblock.${$}.XXXXXX")"
		adb_failfile="${adb_tmpdir}/.fatal"
		adb_tmpload="$(mktemp -p "${adb_tmpdir}" -tu)"
		adb_tmpfile="$(mktemp -p "${adb_tmpdir}" -tu)"
		adb_srtopts="--temporary-directory=${adb_tmpdir} --compress-program=gzip --parallel=${adb_cores} --buffer-size=${adb_srtmem:-"8"}M"
		trap 'f_rmtemp' EXIT
		trap 'f_rmtemp; exit 1' HUP INT TERM
	else
		f_log "err" "the base directory '${adb_basedir}' does not exist/is not mounted yet"
	fi
	[ ! -s "${adb_pidfile}" ] && printf '%s' "${$}" >"${adb_pidfile}"
}

f_rmtemp() {
	[ -f "${adb_errorlog}" ] && [ ! -s "${adb_errorlog}" ] && "${adb_rmcmd}" -f "${adb_errorlog}"
	if [ -n "${adb_tmpdir}" ] && [ -d "${adb_tmpdir}" ] && f_tmpvalid "${adb_tmpdir}"; then
		"${adb_rmcmd}" -rf -- "${adb_tmpdir}" 2>>"${adb_errorlog}" ||
			f_log "info" "temporary directory cleanup failed, path: ${adb_tmpdir}"
	fi
	if [ -s "${adb_pidfile}" ] && [ "$(cat "${adb_pidfile}" 2>/dev/null)" = "${$}" ]; then : >"${adb_pidfile}"; fi
	adb_tmpdir=""; adb_failfile=""
}
```

For both feed workers, call the marker-aware wrapper and exit with its status:

```sh
				f_preparefeed
				exit "${?}"
```

Replace the main-loop throttle with the following so no later feed is queued after the parent observes the marker:

```sh
		if [ "${cnt}" -ge "${adb_cores}" ]; then
			wait -n
			f_waitcheck || break
		fi
		cnt="$((cnt + 1))"
	done
	wait
```

After that final `wait`, abort before prune/merge:

```sh
	if f_failed; then
		read -r fail_phase fail_rc <"${adb_failfile}"
		f_log "info" "update aborted after worker failure, phase: ${fail_phase:-"unknown"}, rc: ${fail_rc:-"1"}"
		[ -s "${adb_rtfile}" ] && f_jsnup "error"
		return 1
	fi
```

- [ ] **Step 4: Verify and commit**

Run syntax plus the harness; expect zero and unrelated `tmp.keep` retained. Commit the two adblock files with `git commit -m "fix: stop adblock after worker failure"`.

### Task 3: Bound feed lifetime and enforce every memory gate

**Files:**
- Modify: `feeds/packages/net/adblock/files/adblock.sh:123-180,228-245,536-558,1271-1338,1927-2140`
- Modify: `feeds/packages/net/adblock/tests/test-memory-safety.sh`

**Interfaces:**
- Consumes: Task 1 memory helpers and Task 2 fatal markers.
- Produces: `f_feedcleanup()` and safe non-zero returns at startup, pre-download, pre-sort, pre-merge, and pre-render.

- [ ] **Step 1: Add a failing feed-cleanup test**

```sh
raw="${test_root}/raw"; category="${test_root}/category"; archive="${test_root}/archive"
remove_file="${test_root}/remove"; prepared="${test_root}/prepared"
printf x >"${raw}"; printf x >"${category}"; printf x >"${archive}"
printf x >"${remove_file}"; printf x >"${prepared}"
src_tmpload="${raw}"; src_tmpcat="${category}"; src_tmparchive="${archive}"
src_tmprmfile="${remove_file}"; src_tmpfile="${prepared}"
assert_success f_feedcleanup
[ ! -e "${raw}" ] && [ ! -e "${category}" ] && [ ! -e "${archive}" ] || failures=$((failures + 1))
[ -e "${remove_file}" ] && [ -e "${prepared}" ] || failures=$((failures + 1))

sort_fail="${test_root}/sort-fail"
printf '#!/bin/sh\nexit 137\n' >"${sort_fail}"
chmod +x "${sort_fail}"
printf 'example.com\n' >"${raw}"
src_name=sortcase; src_rset='feed 1'; src_url=https://example.invalid/list
src_rc=0; adb_action=reload; adb_tld=0; adb_sortcmd="${sort_fail}"; adb_srtopts=""
adb_backupdir="${test_root}"; adb_tmpdir="${test_root}"; adb_failfile="${test_root}/.fatal-sort"
f_chkdom() { cat; }
f_count() { :; }
f_log() { :; }
assert_failure f_preparefeed
assert_success f_failed
grep -qx 'prepare:sortcase 137' "${adb_failfile}" || failures=$((failures + 1))
merge_called=0
publish_called=0
if f_waitcheck; then merge_called=1; publish_called=1; fi
[ "${merge_called}" -eq 0 ] || failures=$((failures + 1))
[ "${publish_called}" -eq 0 ] || failures=$((failures + 1))

printf 'example.com\nexample.org\n' >"${raw}"
printf x >"${category}"; printf x >"${archive}"
src_name=successcase; adb_sortcmd="$(command -v sort)"; adb_failfile="${test_root}/.fatal-success"
adb_gzipcmd="$(command -v gzip)"; adb_backupdir="${test_root}"
assert_success f_preparefeed
[ ! -e "${raw}" ] && [ ! -e "${category}" ] && [ ! -e "${archive}" ] || failures=$((failures + 1))
[ -s "${prepared}" ] || failures=$((failures + 1))
```

Run the harness; expect failure because `f_feedcleanup` is undefined.

- [ ] **Step 2: Implement cleanup and correct the preparation status**

Add:

```sh
f_feedcleanup() {
	"${adb_rmcmd}" -f -- "${src_tmpload}" "${src_tmpcat}" "${src_tmparchive}"
}
```

Add `prep_rc` to the local-variable declaration at the start of `f_list`. Replace its `prepare` case with the following. A sort status such as 137 is fatal and cannot be converted into a successful restore; download failure retains the existing restore behavior.

```sh
	"prepare")
		file_name="${src_tmpfile}"
		if [ -s "${src_tmpload}" ]; then
			if [ "${adb_tld}" = "1" ]; then
				f_chkdom ${src_rset} <"${src_tmpload}" |
					"${adb_awkcmd}" 'BEGIN{FS="."}{for(f=NF;f>1;f--)printf "%s.",$f;print $1}' |
					"${adb_sortcmd}" ${adb_srtopts} -u >"${src_tmpfile}" 2>>"${adb_errorlog}"
			else
				f_chkdom ${src_rset} <"${src_tmpload}" |
					"${adb_sortcmd}" ${adb_srtopts} -u >"${src_tmpfile}" 2>>"${adb_errorlog}"
			fi
			prep_rc="${?}"
			if [ "${prep_rc}" = "0" ] && [ -s "${src_tmpfile}" ]; then
				f_list backup
				out_rc="${?}"
				f_feedcleanup
			else
				[ "${prep_rc}" = "0" ] && prep_rc="4"
				out_rc="${prep_rc}"
				f_log "info" "preparation of '${src_name}' failed, rc: ${prep_rc}"
				[ "${adb_action}" = "reload" ] && f_etag "${src_name}" "" "" "" "1"
				f_feedcleanup
				"${adb_rmcmd}" -f -- "${src_tmpfile}"
			fi
		else
			f_log "info" "download of '${src_name}' failed, url: ${src_url}, rule: ${src_rset:-"-"}, categories: ${src_cat:-"-"}, rc: ${src_rc}"
			if [ "${adb_action}" != "boot" ] && [ "${adb_action}" != "start" ]; then
				[ "${adb_action}" = "reload" ] && f_etag "${src_name}" "" "" "" "1"
				f_list restore
				out_rc="${?}"
			fi
		fi
		;;
```

- [ ] **Step 3: Add startup and worker admission checks**

In `f_load`, after `f_conf` and before `f_dns`:

```sh
	case "${adb_action}" in
	"boot"|"start"|"reload"|"restart") f_memcheck "startup" 0 || return 1;;
	esac
```

Change initialization to propagate failure:

```sh
if [ -S "/var/run/ubus/ubus.sock" ] && ! f_load; then exit 1; fi
```

Replace the update-time flush branch in `f_dns` with initialization only, so neither low memory nor `adb_dnsflush` truncates a valid active list before the transaction commits:

```sh
			if [ ! -f "${adb_finaldir}/${adb_dnsfile}" ]; then
				printf '%b' "${adb_dnsheader}" >"${adb_finaldir}/${adb_dnsfile}"
			fi
```

At the top of both workers:

```sh
				if ! f_memcheck "pre-download:${src_name}" 0; then
					f_failmark "pre-download:${src_name}" 1
					exit 1
				fi
```

Immediately after download and before preparation in both workers:

```sh
				if ! f_memcheck "pre-sort:${src_name}" "${adb_srtmem}"; then
					f_feedcleanup
					f_failmark "pre-sort:${src_name}" 1
					exit 1
				fi
```

Before merge and render respectively:

```sh
	f_memcheck "pre-merge" 0 || { [ -s "${adb_rtfile}" ] && f_jsnup "error"; return 1; }
```

```sh
		f_memcheck "pre-render" 0 || { [ -s "${adb_rtfile}" ] && f_jsnup "error"; return 1; }
```

- [ ] **Step 4: Verify and commit**

Run `sh -n` on both scripts and run the harness. Expect zero and raw/category/archive gone. Commit the two files with `git commit -m "fix: bound adblock update memory"`.

### Task 4: Transactional blocklist publication

**Files:**
- Modify: `feeds/packages/net/adblock/files/adblock.sh:1368-1395,2125-2150`
- Modify: `feeds/packages/net/adblock/tests/test-memory-safety.sh`

**Interfaces:**
- Produces: `f_publish(candidate, active)`; final rendering targets a run-local candidate.

- [ ] **Step 1: Add a failing publication test**

```sh
adb_mvcmd="$(command -v mv)"
active="${test_root}/active.list"; candidate="${test_root}/candidate.list"
printf 'old-rule\n' >"${active}"; : >"${candidate}"
assert_failure f_publish "${candidate}" "${active}"
[ "$(cat "${active}")" = old-rule ] || failures=$((failures + 1))
printf 'new-rule\n' >"${candidate}"
assert_success f_publish "${candidate}" "${active}"
[ "$(cat "${active}")" = new-rule ] || failures=$((failures + 1))
```

Run the harness; expect failure because `f_publish` is undefined.

- [ ] **Step 2: Add atomic publication**

```sh
f_publish() {
	local candidate="${1}" active="${2}"
	[ -s "${candidate}" ] || return 4
	"${adb_mvcmd}" -f -- "${candidate}" "${active}"
}
```

Replace the `f_list final` case with this complete candidate renderer:

```sh
	"final")
		src_name=""
		file_name="${adb_tmpdir}/${adb_dnsfile}.candidate"
		if [ -s "${adb_tmpdir}/tmp.add.allowlist" ]; then
			"${adb_sortcmd}" ${adb_srtopts} -u "${adb_tmpdir}/tmp.add.allowlist" -o "${adb_tmpdir}/tmp.add.allowlist" || return "${?}"
		fi
		{
			[ -n "${adb_dnsheader}" ] && printf '%b' "${adb_dnsheader}"
			[ -s "${adb_tmpdir}/tmp.add.allowlist" ] && "${adb_catcmd}" "${adb_tmpdir}/tmp.add.allowlist"
			[ "${adb_safesearch}" = "1" ] && "${adb_catcmd}" "${adb_tmpdir}/tmp.safesearch."* 2>>"${adb_errorlog}"
			if [ "${adb_dnsdeny}" = "1" ]; then
				f_dnsdeny "${adb_tmpdir}/${adb_dnsfile}"
			else
				"${adb_catcmd}" "${adb_tmpdir}/${adb_dnsfile}"
			fi
		} >"${file_name}"
		out_rc="${?}"
		;;
```

In `f_main`, publish only on complete success:

```sh
		if f_list final && f_publish "${adb_tmpdir}/${adb_dnsfile}.candidate" "${adb_finaldir}/${adb_dnsfile}"; then
			chown "${adb_dnsuser}" "${adb_finaldir}/${adb_dnsfile}" 2>>"${adb_errorlog}"
		else
			f_log "info" "final blocklist rendering failed; active blocklist retained"
			[ -s "${adb_rtfile}" ] && f_jsnup "error"
			return 1
		fi
```

Move the existing symlink/shifted-backup handling immediately after this block. Replace the no-merge branch that writes a header with:

```sh
	else
		f_log "info" "no merge input; active blocklist retained"
		[ -s "${adb_rtfile}" ] && f_jsnup "error"
		return 1
	fi
```

- [ ] **Step 3: Verify and commit**

Run syntax and the full harness. Expected: empty candidate leaves `old-rule`; non-empty candidate atomically changes it to `new-rule`. Commit with `git commit -m "fix: publish adblock lists atomically"`.

### Task 5: Add zram to the VS010 image

**Files:**
- Modify: `target/linux/qualcommax/image/ipq50xx.mk:301-327`
- Create: `target/linux/qualcommax/image/tests/test-vs010-packages.sh`

**Interfaces:**
- Produces: exactly one `zram-swap` token in `Device/unicom_vs010`.

- [ ] **Step 1: Write the failing static test**

```sh
#!/bin/sh
set -eu
script_dir="$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)"
block="$(awk '/^define Device\/unicom_vs010$/{on=1} on{print} on&&/^endef$/{exit}' "${script_dir}/../ipq50xx.mk")"
[ -n "${block}" ] || { printf 'VS010 block not found\n' >&2; exit 1; }
count="$(printf '%s\n' "${block}" | awk '{for(i=1;i<=NF;i++)if($i=="zram-swap")n++} END{print n+0}')"
[ "${count}" -eq 1 ] || { printf 'expected one zram-swap token, found %s\n' "${count}" >&2; exit 1; }
printf 'ok - VS010 includes zram-swap\n'
```

Run it and expect `found 0`.

- [ ] **Step 2: Select zram and rerun the test**

Add this line after `luci-app-adblock \` without removing packages:

```make
		zram-swap \
```

Run the static test and expect `ok - VS010 includes zram-swap`.

- [ ] **Step 3: Preserve the dirty Makefile boundary**

Inspect `git diff -- target/linux/qualcommax/image/ipq50xx.mk`. Do not stage that file because the device block is user-owned uncommitted work. Commit only the executable test:

```bash
git add target/linux/qualcommax/image/tests/test-vs010-packages.sh
git commit -m "test: require zram in VS010 image"
```

### Task 6: Release bump and build verification

**Files:**
- Modify: `feeds/packages/net/adblock/Makefile:10`
- Verify: all task files.

**Interfaces:**
- Produces: `PKG_RELEASE:=4` and build/test evidence.

- [ ] **Step 1: Change `PKG_RELEASE:=3` to `PKG_RELEASE:=4`**

Use this exact value:

```make
PKG_RELEASE:=4
```

- [ ] **Step 2: Run local regressions**

```bash
sh -n feeds/packages/net/adblock/files/adblock.sh
sh -n feeds/packages/net/adblock/tests/test-memory-safety.sh
sh -n target/linux/qualcommax/image/tests/test-vs010-packages.sh
feeds/packages/net/adblock/tests/test-memory-safety.sh
target/linux/qualcommax/image/tests/test-vs010-packages.sh
git diff --check
```

Expected: every command returns zero.

- [ ] **Step 3: Build and inspect target metadata**

```bash
make defconfig
make package/adblock/compile V=s
make package/zram-swap/compile V=s
make target/linux/compile V=s
make json_overview_image_info V=s
rg -n 'unicom_vs010|zram-swap' .config tmp/info .targetinfo.json 2>/dev/null
```

Expected: builds return zero and generated VS010 metadata contains `zram-swap`.

- [ ] **Step 4: Commit only the clean release file**

```bash
git add feeds/packages/net/adblock/Makefile
git commit -m "build: bump adblock package release"
```

Confirm with `git status --short` that unrelated changes and the edited `ipq50xx.mk` remain unstaged.

### Task 7: Hardware acceptance after build and flash

**Files:**
- No source changes.

**Interfaces:**
- Consumes: freshly built/flashed VS010 image.
- Produces: serial evidence for zram, OOM-free reload, cleanup, and active-list preservation.

- [ ] **Step 1: Confirm zram on the router**

```sh
sed -n '/^MemTotal:/p;/^MemAvailable:/p;/^SwapTotal:/p;/^SwapFree:/p' /proc/meminfo
cat /proc/swaps
zramctl 2>/dev/null || true
```

Expected: `/dev/zram0` is listed with logical size near half of `MemTotal`.

- [ ] **Step 2: Reload all three feeds while monitoring**

Run `/etc/init.d/adblock reload`; in parallel sample:

```sh
while :; do
	date
	sed -n '/^MemAvailable:/p;/^SwapFree:/p;/^Shmem:/p' /proc/meminfo
	du -sk /tmp/adblock.* 2>/dev/null || true
	sleep 2
done
```

Expected: reload completes, serial remains responsive, no OOM/killed-process log appears, and run directories disappear.

- [ ] **Step 3: Verify DNS and blocklist state**

```sh
ubus call adblock status
wc -l /tmp/smartdns/adb_list.overall
/etc/init.d/smartdns status
/etc/init.d/https-dns-proxy status
nslookup openwrt.org 127.0.0.1
```

Expected: non-zero block count, both DNS services running, lookup successful.

- [ ] **Step 4: Force rejection and prove preservation**

```sh
sha256sum /tmp/smartdns/adb_list.overall >/tmp/adb.before
uci set adblock.global.adb_memreserve='4096'
uci commit adblock
/etc/init.d/adblock reload; test "$?" -ne 0
sha256sum /tmp/smartdns/adb_list.overall >/tmp/adb.after
cmp /tmp/adb.before /tmp/adb.after
uci set adblock.global.adb_memreserve='16'
uci commit adblock
```

Expected: phase/required/observed rejection is logged and `cmp` returns zero.

- [ ] **Step 5: Confirm no accumulation**

```sh
find /tmp -maxdepth 1 -type d -name 'adblock.*' -print
du -sk /tmp
logread | tail -200 | grep -E 'oom-kill|Out of memory|Killed process'
```

Expected: no package run directory remains, tmpfs returns near baseline, and the final search has no output.

## Completion criteria

- Shell syntax and both regression harnesses pass.
- `adblock` and `zram-swap` compile for the selected OpenWrt configuration.
- VS010 metadata contains `zram-swap` exactly once.
- Worker failure prevents further queueing after detection and skips merge/render.
- Every admission or processing failure retains the previous active blocklist.
- Successful and catchably interrupted runs remove only package-owned temporary state.
- Hardware reload completes without OOM and `/dev/zram0` is active.
