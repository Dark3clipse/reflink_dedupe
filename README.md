# reflink_dedupe

`reflink_dedupe` is a FreeBSD utility for deduplicating files on filesystems that support copy-on-write reflinks. It efficiently finds duplicate files, replaces them with reflinks, and can optionally monitor directories for live deduplication.

> ⚠️ **Experimental software**: `reflink_dedupe` is in active development. While great care has been taken in ensuring the reflinking functionality cannot cause data loss, **use at your own risk**. The author is **not responsible for any data loss**.

## Purpose

This utility enables software based, retroactive deduplication. Do not use this utility if you want proactive deduplication at the filesystem level.

Examples when you would want to use this tool:
- You have an existing filesystem that supports reflinks but your duplicates are not reflinked. This can for example happen after you have migrated a zfs dataset using zfs send/receive.
- You have configured reflink support but your clients cannot create reflinks. For example, when exposing your zfs data with NFS, clients are not yet able to create reflinked copies when source and target span multiple zfs datasets. This tool can retroactively scan your data for duplicates and replace regular copies with reflinks.
- You do not have the RAM to enable zfs deduplication but want to save storage space by making use of reflinks without requiring clients to use specific copy instructions in order to utilize reflinking functionality.

## Features

- **Indexing**: Scan all files under a deduplication root and store metadata in a SQLite database.
- **Duplicate detection**: Find duplicate files based on their content hash.
- **Reflink deduplication**: Replace duplicates with copy-on-write reflinks.  
  > Note: Your filesystem must support reflinks. You can provide a custom command for creating reflinks if a simple `cp` is not sufficient.
- **Live monitoring**: Watch directories and subdirectories within the deduplication root for changes and automatically deduplicate new or modified files.
- **Multi-threading**: Perform deduplication in parallel for faster processing on large datasets.
- **Statistics collection**: Track the number of reflinks created and the total space saved (in bytes).
- **Conflict management**: In the unlikely event of conflicts, they are renamed and added to a list for manual repair.
- **Dashboard**: A dashboard is rendered providing clear overview of what tasks are running.
- **Reporting**: After each execution a concise report is generated and stored for later reference.

## Preview

Live dashboard:
```
Statistics:
Reflinked data: 10.918 TiB   | Reflinks created: 4068689     | Conflicts found: 0     
Pool bclone used: 15.089 TiB | Pool bclone saved: 22.028 TiB | Pool bclone ratio: 2.45

Tasks in progress:
indexing | threads: 8 | progress: 0% (0/300) ETA: 0s (elapsed: 0s)

Recent logs:
[2025-10-24 20:31:07] Starting task: indexing, max duration: 0, max count: 300, cores: 8
[2025-10-24 20:31:07] Finished task: clean
[2025-10-24 20:30:57] [CLEAN] Total computed. Real: 2747975 files, Effective: 10000 files.
[2025-10-24 20:30:56] Starting task: clean, max duration: 0, max count: 10000, cores: 8
[2025-10-24 20:20:18] Finished task: reflink-duplicates
[2025-10-24 20:20:18] [REFLINK-DUPLICATES] Processing finished.
[2025-10-24 20:20:02] [REFLINK-DUPLICATES] Total computed. Real: 65934954 dupes, Effective: 25 dupes.
[2025-10-24 20:19:48] Starting task: reflink-duplicates, max duration: 0, max count: 25, cores: 8
[2025-10-24 20:19:47] Finished task: find-duplicates
[2025-10-24 20:19:46] [FIND-DUPLICATES] Processing finished.
```

Report:
```
=================== EXECUTION REPORT ======================================================================================
Execution Start : 2025-10-24 20:19:16
Execution End   : 2025-10-24 20:20:18
Total Duration  : 1m 02s

Statistics before:
Reflinked data: 10.918 TiB   | Reflinks created: 4068664     | Conflicts found: 0     
Pool bclone used: 15.089 TiB | Pool bclone saved: 22.028 TiB | Pool bclone ratio: 2.45

Statistics after:
Reflinked data: 10.918 TiB   | Reflinks created: 4068689     | Conflicts found: 0     
Pool bclone used: 15.089 TiB | Pool bclone saved: 22.028 TiB | Pool bclone ratio: 2.45

Tasks:
---------------------------------------------------------------------------------------------------------------------------
Task                 | Skipped      | Max count  | Max duration  | Actual count | Actual duration | Result                    
---------------------------------------------------------------------------------------------------------------------------
Clean                | 0            | 10000      | 0s            | 10000        | 11s             | 0 rows cleaned            
Indexing             | 0            | 300        | 0s            | 300          | 18s             | 0 files indexed           
Find Duplicates      | 0            | 5          | 0s            | 5            | 3s              | 0 dupes found             
Reflink Duplicates   | 0            | 25         | 0s            | 25           | 30s             | 25 reflinks created       
===========================================================================================================================
```

---

## Installation

### Using mediarr repository

Add mediarr repository to your system:
```
mkdir -p /usr/local/etc/pkg/repos
cat <<EOF> /usr/local/etc/pkg/repos/mediarr.conf
mediarr: {
  url: "https://static.mediarr.org/freebsd-repo",
  enabled: yes
}
EOF
```

Install package:
```
pkg update
pkg install reflink_dedupe
```

### Using make

You can install `reflink_dedupe` via my [**FreeBSD ports repository**](https://github.com/Dark3clipse/freebsd-ports). 

> Note: Ensure you have the ports tree installed (refer to the freebsd ports repository for instructions).

```{sh}
git clone https://github.com/Dark3clipse/freebsd-ports.git /usr/local/freebsd-ports
mkdir -p /usr/ports/local/devel
mkdir -p /usr/ports/local/sysutils
ln -s /usr/local/freebsd-ports/devel/docopts /usr/ports/local/devel/docopts
ln -s /usr/local/freebsd-ports/sysutils/reflink_dedupe /usr/ports/local/sysutils/reflink_dedupe
cd /usr/local/freebsd-ports/sysutils/reflink_dedupe
sudo make install clean
sudo cp /usr/local/etc/reflink_dedupe.conf.sample /usr/local/etc/reflink_dedupe.conf
```

Don't forget to edit your configuration file to your needs!

#### Optional Features

WATCHER: Enable live monitoring of directories using fswatch

```{sh}
sudo make install WITH_WATCHER=yes
```

ZFS: Enable ZFS pool integration (disabled by default)

```{sh}
sudo make install WITH_ZFS=yes
```

## Usage

```
Usage:
  reflink_dedupe [options]...

Options:
      -o, --oneshot         Run full deduplication on the configured root and exit.
      -d, --daemon          Run in daemon-mode. Does not immediatelly start deduplicating, but rather waits on an external trigger for scheduling.
      -w, --watcher         Enable fs watcher to react to incoming fs events.
      -i, --interactive     Show an interactive overview screen.
          --print-args      Print command arguments and exit.
          --print-config    Print configuration and exit.
          --schedule        Schedule an execution now. Another instance must be running in daemon mode for this to work.
      -h, --help            Show help options.
      -V, --version         Print program version.
```

### Run in oneshot mode

To run the utility once and perform full deduplication of the specified deduplication root (in the config file):

```
reflink_dedupe --oneshot --interactive
```

### Run as a daemon

To configure your system to run the daemon automatically at boot (recommended):

```
echo 'reflink_dedupe_enable="YES"' | sudo tee -a /etc/rc.conf
```

To start the daemon immediatelly:

```
sudo service reflink_dedupe start
```

## Configuration

`reflink_dedupe` uses the configuration file located at `/usr/local/etc/reflink_dedupe.conf`.

Lines beginning with # are comments. The default configuration contains sections for general settings, logging, operation modes, and watcher settings.

### Example Configuration
```
##################################################
# General Settings
##################################################

DEDUPLICATION_ROOT="/"                   # Root working path for deduplication / indexing
DB="/var/db/reflink_dedupe.db"           # SQLite3 database location
PID_FILE="/var/run/reflink_dedupe.pid"   # PID file location
HASH_CMD="sha256 -q"                     # Command used for hashing files
REFLINK_CMD="cp"                         # Command used for reflinking files
THREADS=""                               # Max threads; leave empty to use system cores
TMP_DIR="/tmp/reflink_dedupe"            # Temporary working directory
LOCK_DIR="/var/lock/reflink_dedupe"      # Directory for lock files
CLEAN_BATCH_SIZE="100"                   # Batch size for database cleaning task.
FIND_DUPLICATES_BATCH_SIZE="1000"        # Batch size for find duplicates task.

##################################################
# Scheduling settings
##################################################

SCHEDULE_CLEAN_MAX_COUNT="0"             # Maximum database entries to check for cleanup per schedule run. Zero indicates no maximum.
SCHEDULE_CLEAN_MAX_DURATION="0"          # Maximum duration in seconds of cleaning per schedule run. Cleaning will be stopped when this duration is reached. Zero indicates no maximum.
SCHEDULE_CLEAN_SKIP="0"                  # Flag to skip cleaning entirely. When set to 1 cleaning will we skipped during scheduled executions.
SCHEDULE_INDEXING_MAX_COUNT="0"          # Maximum hashes to compute per schedule run. Zero indicates no maximum.
SCHEDULE_INDEXING_MAX_DURATION="0"       # Maximum duration in seconds of indexing per schedule run. Indexing will be stopped when this duration is reached. Zero indicates no maximum.
SCHEDULE_INDEXING_SKIP="0"               # Flag to skip indexing entirely. When set to 1 indexing will we skipped during scheduled executions.
SCHEDULE_FIND_DUPLICATES_MAX_COUNT="0"   # Maximum find-duplicate batches to execute per schedule run. Zero indicates no maximum.
SCHEDULE_FIND_DUPLICATES_MAX_DURATION="0"# Maximum duration in seconds of finding duplicates per schedule run. Finding duplicates will be stopped when this duration is reached. Zero indicates no maximum.
SCHEDULE_FIND_DUPLICATES_SKIP="0"        # Flag to skip finding duplicates entirely. When set to 1 finding duplicates will we skipped during scheduled executions.
SCHEDULE_REFLINKING_MAX_COUNT="0"        # Maximum reflinks to create per schedule run. Zero indicates no maximum.
SCHEDULE_REFLINKING_MAX_DURATION="0"     # Maximum duration in seconds of creating reflinks per schedule run. Creating reflinks will be stopped when this duration is reached. Zero indicates no maximum.
SCHEDULE_REFLINKING_SKIP="0"             # Flag to skip creating reflinks entirely. When set to 1 creating reflinks will we skipped during scheduled executions.

##################################################
# Logging
##################################################

DASHBOARD_FILE="/var/log/reflink_dedupe/dashboard.log"  # Interactive dashboard stored to file. Leave empty to disable writing the dashboard to file.
REPORTS_FILE="/var/log/reflink_dedupe/reports.log"      # Execution report file. Leave empty to disable writing reports to file.
LOG_FILE="/var/log/reflink_dedupe/info.log"
LOG_FILE_IMPORTANT="/var/log/reflink_dedupe/important.log"
LOG_FILE_ERRORS="/var/log/reflink_dedupe/errors.log"
LOG_FILE_ACTIONS="/var/log/reflink_dedupe/actions.log"

##################################################
# Operation Modes
##################################################

DRY_RUN="0"                                # Dry-run mode (1 = don’t modify files)
STATISTICS_ZPOOL_MONITOR_ENABLED="0"       # Enable ZFS pool monitoring
STATISTICS_ZPOOL_MONITOR_ZPOOL="rpool"     # ZFS pool to monitor

##################################################
# Watcher Settings
##################################################

WATCH_PATHS=""                             # Relative paths under DEDUPLICATION_ROOT
WATCHER_DEBOUNCE_SECONDS="60"              # Seconds to wait before processing events
WATCH_THREADS="1"                          # Threads for fs watching
```

- Modify DEDUPLICATION_ROOT to the root of the directory tree you want to deduplicate.
- REFLINK_CMD should create a copy-on-write reflink; defaults to cp.
- THREADS controls multi-threading.
- WATCH_PATHS and watcher settings are used if live monitoring is enabled.

## Dependencies

- **Required ports:**
  - docopts — command-line argument parser
  - sqlite3 — for storing file metadata
  - parallel — for multi-threaded operations
- Optional dependencies:
  - fswatch — live file system monitoring
  - zpool — ZFS pool integration (optional)
- Base system utilities (no extra installation required):
  - awk, printf, realpath, flock, pgrep, tput, date, sha256, find, wc, md5, daemon
 
## License

This project is licensed under the BSD 2-Clause License – see the [LICENSE](LICENSE) file for details.
