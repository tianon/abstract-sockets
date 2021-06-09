# Abstract Sockets

Abstract Sockets are a feature of the Linux kernel that lie somewhere between file-based Unix Sockets and port/network-based TCP Sockets.  They are opened and referenced by name, and have several useful/unique properties:

- automatically cleaned up when the last process using them exits
- locking (for singleton-style processes), but without files
- not tied to specific files on-disk / shared across the network namespace (thus usable across chroot/container boundaries)

As you might have surmised (or [already](https://github.com/moby/moby/issues/14767) [known](https://docs.docker.com/engine/reference/run/#network-host) / [heard](https://medium.com/nttlabs/dont-use-host-network-namespace-f548aeeef575)), it's this last item that's the most interesting/troubling (depending on your point of view), especially when deploying containers (or using chroots for building software packages).

While [it may seem odd for these to be tied to the network namespace](https://github.com/containers/bubblewrap/issues/330#issuecomment-533792265) as opposed to the mount namespace or their own kernel namespace, that's the world we're living in, so we get to adapt instead.

The following is an attempt to document the types of applications which might be using abstract sockets by default, and thus be vulnerable/accessible to `--network=host` containers.  Applications using abstract sockets on a given host can be found using something like `netstat -xl | grep ' @'`.

In many cases (such as X11), the "insecure" aspect of abstract sockets [is the *reason* they are used](https://tstarling.com/blog/2016/06/x11-security-isolation/), but **even in those cases, BEFORE suggesting an addition to this list**, we expect you to make an attempt at responsible (security) disclosure to the relevant project so that they can privately make that determination for themselves (*before* they're put on blast).  Thanks!

## Useful References

- https://gitlab.gnome.org/GNOME/glib/-/merge_requests/911#note_529866
- https://gitlab.gnome.org/GNOME/at-spi2-core/-/issues/28#note_992076

## Applications

- **X11 / Xorg**
  - security impact: keylogging, screen capture
  - typical sockets: `@/tmp/.X11-unix/X0`
  - mitigation / further reading:
    - https://tstarling.com/blog/2016/06/x11-security-isolation/
    - `X -nolisten local`

- **D-Bus**
  - security impact: varying (but generally Not Great)
  - typical sockets: `@/tmp/dbus-XXXXXX`
  - mitigation / further reading:
    - https://github.com/netblue30/firejail/issues/801
    - adjust `<listen>unix:tmpdir=/tmp</listen>` in *all❗* config files
      - https://dbus.freedesktop.org/doc/dbus-specification.html#transports
      - `<listen>unix:dir=/tmp</listen>`

- **containerd-shim**
  - security impact: ability to launch additional processes
  - typical sockets: `@/containerd-shim/XXXXXX.sock@`, `@/containerd-shim/moby/XXXXXX/shim.sock@`
  - mitigation / further reading:
    - https://github.com/containerd/containerd/security/advisories/GHSA-36xw-fx78-c5r4
    - update to 1.3.9+, 1.4.3+ (CVE-2020-15257)
