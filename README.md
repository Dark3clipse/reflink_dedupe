# reflink_dedupe

`reflink_dedupe` is a FreeBSD utility for deduplicating files on filesystems that support copy-on-write reflinks. It efficiently finds duplicate files, replaces them with reflinks, and can optionally monitor directories for live deduplication.

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

You can install `reflink_dedupe` via my [https://github.com/Dark3clipse/freebsd-ports](**FreeBSD ports repository**):

```{sh}
git clone https://github.com/Dark3clipse/freebsd-ports.git /usr/local/freebsd-ports
cd /usr/local/freebsd-ports/sysutils/reflink_dedupe
sudo make install clean
```

### Optional Features

FSWATCH: Enable live monitoring of directories using fswatch

```{sh}
sudo make install WITH_WATCHER=yes
```

ZPOOL: Enable ZFS pool integration (disabled by default)

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
