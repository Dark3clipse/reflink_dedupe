# reflink_dedupe

`reflink_dedupe` is a FreeBSD utility for deduplicating files on filesystems that support copy-on-write reflinks. It efficiently finds duplicate files, replaces them with reflinks, and can optionally monitor directories for live deduplication.

> ⚠️ **Experimental software**: `reflink_dedupe` is in active development. While care has been taken in writing the reflinking functionality, **use at your own risk**. The author is **not responsible for any data loss**.

## Features

- **Indexing**: Scan all files under a deduplication root and store metadata in a SQLite database.
- **Duplicate detection**: Find duplicate files based on their content hash.
- **Reflink deduplication**: Replace duplicates with copy-on-write reflinks.  
  > Note: Your filesystem must support reflinks. You can provide a custom command for creating reflinks if a simple `cp` is not sufficient.
- **Live monitoring**: Watch directories and subdirectories within the deduplication root for changes and automatically deduplicate new or modified files.
- **Multi-threading**: Perform deduplication in parallel for faster processing on large datasets.
- **Statistics collection**: Track the number of reflinks created and the total space saved (in bytes).

---

## Installation

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

### Optional Features

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

##################################################
# Logging
##################################################

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
