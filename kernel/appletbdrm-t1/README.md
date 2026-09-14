# T1 appletbdrm override

This is Linux stable v7.1.8's `appletbdrm` driver adapted for the T1 USB ID,
response flow, and suspend/resume behavior. It deliberately keeps the in-tree
module name and installs under `updates/dkms`, taking precedence over the
in-tree module.

The legacy `apple-ibridge-no-config1-revert.patch` is unnecessary because
T1Bridge does not install the config-1 `apple-ib-drv` stack it modifies.

## Upstream status

The three-patch series in [`upstream/`](upstream/) is a private review draft and
has not been submitted. It is based on kernel.org stable tag `v7.1.8` (tag
object `1af9bbf117b680f903a81485cd74501c334bf538`, commit
`25c76bea853d0db65b51fb4697a47cbfd9e35e76`). Phase 1 hardware acceptance is
complete; suspend/resume was recorded as not run under the host-safety rule.
Owner DCO sign-off and validation against the eventual submission base remain
required before sending the series upstream.

## Failed resume recovery

The T1 resume callback checks both endpoint clear-halt operations and every
display rearm step before restoring the saved DRM mode. A rearm or mode-restore
failure returns its error and marks the display interface for USB core's
deferred unbind/reprobe at system PM completion. Returning an error alone does
not request that recovery. A successful resume retains the existing DRM device;
it need not produce a new userspace `display-open` diagnostic.

The fallback reprobes only the display interface. It does not reset the whole
T1 or change configuration, autosuspend policy, or protected state. Reprobe
recreates the DRM device so the packaged device-bound hardware service can
reopen it and renderers can reconnect. A failed probe leaves the device
unavailable rather than requesting an unbounded reset loop.

`make test sanitize` exercises the production resume callbacks with synthetic
USB and DRM boundaries, including failures at every rearm step and non-T1
behavior. This establishes error propagation and the deferred-rebind request,
not physical recovery. Attended system-suspend validation remains in
[#18](https://github.com/standardagents/t1bridge/issues/18); follow the
[hardware runbook](../../docs/hardware-validation.md#suspend-and-resume).
