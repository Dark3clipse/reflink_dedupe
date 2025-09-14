 
## Debugging

Redistribute v0.9.0
```
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
 
## License

This project is licensed under the BSD 2-Clause License – see the [LICENSE](LICENSE) file for details.
