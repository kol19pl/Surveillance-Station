# 9.3.0-12139 x86_64 recording regression at approximately 3000 seconds

Tracking issue: https://github.com/ohyeah521/Surveillance-Station/issues/75

## Observed behavior

- Cam 50 `sscamerad` started at `08:47:44` after Surveillance Station restarted.
- At `09:37:45`, it logged `Exceed header reserved time but no I frame`.
- The elapsed time was approximately 3001 seconds.
- A PCAP showed that RTSP remained established, H.264 continued entering the NAS, and valid IDR frames arrived approximately every second.
- `sscamerad` stayed alive, but new MP4 files stopped being created.

This disproves a network-side “camera stopped sending I-frames” explanation for this reproduction.

## Binary location and timing

In the official 12139 x86_64 `sscamerad`:

- error string file offset: `0x1040E8`;
- unique code reference: file offset `0xD6AE5`, VA `0x4D6AE5`;
- containing function: `StmMuxerExec::ChkExceedTimeClosing`, beginning at file offset `0xD6860`;
- fixed comparison: `cmp esi, 0x4AF`, so forced closing begins at 1200 elapsed seconds.

The measured timing is consistent with an approximately 1800-second segment configuration:

```text
approximately 1800-second recording segment
+ 1200-second wait-for-I-frame/header-reservation grace
= approximately 3000 seconds
```

The 1200-second function also exists in 9.2.5. The regression is therefore treated as a failure to propagate/accept the real I-frame or format state before that old safety timeout expires.

## Candidate patch

The first merged 12139 payload left `sscamerad` identical to the official binary. The follow-up candidate changes only paths that differ when the 9.3 anomaly overrides activate:

| File offset | Original | Candidate | Effect |
|---:|---|---|---|
| `0xB58A9` | `75 1D` | `EB 45` | Jump directly to the normal path, bypassing both the CommonCfg status override and its 93600-second anomaly handler. |
| `0xCFF2D` | `0A 44 24 23` | `90 90 90 90` | Keep the real `MediaBlock::IsAVC1()` result; do not OR in the hidden status/time override. |

Normal-state behavior is unchanged: the original code already reaches the same path when CommonCfg is healthy, and the removed OR is a no-op when the local override is zero.

The candidate deliberately leaves `StmMuxerExec::ChkExceedTimeClosing` unchanged. Disabling that timeout would only keep an oversized current file open and would mask the missing-I-frame state rather than restore proper MP4 rotation.

## Live acceptance test

After installing the candidate and restarting Surveillance Station:

1. Record the exact service and Cam 50 `sscamerad` start times.
2. Save the baseline hashes for all nine installed payload files.
3. Keep the same camera, codec, recording profile, and segment settings used in the reproduction.
4. Run continuously for at least 65 minutes.
5. Confirm a new MP4 is created at the normal segment boundary and creation continues beyond 3000 seconds.
6. Confirm the log does not contain `Exceed header reserved time but no I frame`.
7. Play files immediately before and after both the segment boundary and 3000-second boundary.
8. Confirm RTSP/IDR evidence and `sscamerad` process state remain normal.

Acceptance requires all eight checks. A displayed license count or a running process alone is not closure.

## Rollback

Restore the package-file backups with:

```bash
./activated.sh -r
```

Repository-level rollback and reapply remain available through `rollback.sh` and `reapply.sh` in this directory.
