# Zone Protection CLI Snippets

Reference snippets for building Zone Protection profiles. Tune thresholds to measured CPS baselines per environment, don't apply these values blind.

## Create a zone protection profile with flood protection

```
set network profiles zone-protection-profile ZoneProtect flood tcp-syn enable yes
set network profiles zone-protection-profile ZoneProtect flood tcp-syn alert 1000
set network profiles zone-protection-profile ZoneProtect flood tcp-syn activate 5000
```

## Reconnaissance protection (port scan / host sweep)

```
set network profiles zone-protection-profile ZoneProtect scan tcp-port enable yes action block-ip duration 300
set network profiles zone-protection-profile ZoneProtect scan udp-port enable yes action block-ip duration 300
set network profiles zone-protection-profile ZoneProtect scan host-sweep enable yes action block-ip duration 300
```

## Apply profile to a zone

```
set vsys vsys1 zone untrust network zone-protection-profile ZoneProtect
```

## Commit

```
commit
```

## Notes

- Take average/peak CPS baseline measurements before setting Alarm/Activate/Maximum thresholds. Use Panorama Health Monitor or `show running resource-monitor` for CPU/session data.
- On PAN-OS 10.0+, AIOps can provide Zone Protection profile threshold recommendations based on system telemetry, use that as a starting point instead of guessing.
- Packet Buffer Protection is separate from Zone Protection and should be enabled/tuned independently.
