This file is a personal notebook for useful commands I use when developing this utility.

## Debugging

Redistribute v0.9.0
```{sh}
git tag -d v0.9.0
git tag -a v0.9.0 -m "Release version 0.9.0"
git push origin v0.9.0
```

Reinstall v0.9.0
```
cd /usr/ports/local/sysutils/reflink_dedupe
rm -f /usr/ports/distfiles/Dark3clipse-reflink_dedupe-0.9.0-v0.9.0_GH0.tar.gz
make makesum
make clean
make fetch
make reinstall clean
```
