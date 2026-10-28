# Genius Air device protocol v1

Target: real ESP-12E, offline AP `Genius_Air`, 192.168.4.1, WebSocket port 81.
No UI-only controller state or simulated transport is permitted in the Ionic app.
Physical coil/motor feedback and actual height are not measured. Reported masks
describe the output bits actually written by firmware, not measured airflow.

## Pins (GPIO numbers, not ESP module pad numbers)

74HC595 DATA=13, CLOCK=14, LATCH=12, active-low OE=16; RXB61 DATA=4;
Learn button=5 (active low); compressor power-stage enable=15 (active high);
addressable RGB data=2 (through 3.3-to-5 V buffer); GPIO0 reserved for programming.
GPIO1 is runtime external watchdog WDI; UART TX is released before arming.
OE requires a physical pull-up to 3.3 V; ULN inputs require an off bias when
595 outputs are high impedance. GPIO15 requires its boot pull-down. GPIO2 needs
its boot pull-up. GPIO15 controls a separate rated power stage, never a 1 A coil
directly through ESP or ULN2003. Start with OE disabled, latch zero, then enable.
Q0/Q1=FL fill/exhaust, Q2/Q3=FR, Q4/Q5=RL, Q6/Q7=RR.
Wheel masks: FL=1, FR=2, RL=4, RR=8. Opposing bits may never overlap.

## Authentication and envelopes

AP password and control key are unique installation secrets in firmware's
`secrets.h` (untracked; example contains placeholders and refuses operation).
The app asks for host and control key. It must not claim connected before a
successful version/device/status handshake. No hard-coded shared control key.

Client connects to `ws://HOST:81/` and sends:
`{"type":"auth","id":"request-id","protocol":1,"key":"...","clientId":"..."}`.
Server replies:
`{"type":"hello","id":"request-id","ok":true,"protocol":1,"deviceId":"Genius_Air-ABCDEF","session":"unpredictable-connection-nonce","status":{...}}`.
Auth failure: `{"type":"hello","id":"...","ok":false,"error":"unauthorized"}` then close.

Every command is:
`{"type":"cmd","id":"unique-id","session":"...","seq":1,"command":"...","payload":{...}}`.
seq strictly increases per connection (1..2147483647). Never replay commands
after reconnect. Server rejects stale seq/session. Only a fresh authenticated
connection may own commands. Command replies:
`{"type":"ack","id":"...","session":"...","ok":true,"status":{...}}` or
same with `ok:false,error:"machine-readable-code"`.
Server pushes `{"type":"status","session":"...","status":{...}}` at least
every 250 ms while authenticated. Client makes status unknown and disconnects
if no valid status/ack has arrived for 1500 ms. Ping is an ordinary `ping`
command every 500 ms; firmware disconnects silent clients after 2000 ms.

## Commands

- `ping`: `{}`; status reply.
- `stop`: `{}`; stop valves/automation, invalidate hold/source. Compressor is
  independent and retains its explicit commanded state.
- `hold.start`: `{holdId,fill,exhaust}`; validate 4-bit disjoint masks, cancel
  automation, start fresh ownership. Timeout fixed at 350 ms; never restore an
  expired id. App must only issue start from a fresh physical press.
- `hold.update`: `{holdId,fill,exhaust}`; only current same-client/id can update
  and renew. Send every 100 ms while held, including zero demand for motion.
- `hold.end`: `{holdId}`; stop that owner's hold. Old/mismatched end cannot stop
  another owner's operation. Release before start ack must still send end.
- `compressor.set`: `{enabled:boolean}`; idempotent desired state.
- `pulse.run`: `{}`; 10,000 ms elapsed, 500 ms on/500 off, ON first, all exhaust.
- `preset.reference`: `{preset:1..3}`; outputs must be idle, declares manually
  established source without motion.
- `preset.run`: `{preset:1..3}`; requires known source and configured source->
  target route. No initial dump/rehome. Source unknown while running; completed
  target becomes source. Same target is a no-op.
- `config.route`: `{from:1..3,to:1..3,wheels:[{flow:0|1|2,ms:0..30000},...4]}`;
  distinct from/to, Off requires 0 ms, all wheels start together/end separately.
- `config.mapping`: `{profile:4|6|12,button:1..profile,action:string}`.
- `config.resetMappings`: `{profile:4|6|12}`.
- `remote.learn`: `{profile:4|6|12,button:1..profile}`; arm learn 20 s, no valve
  demand while learning. `button:0` on profile4 learns the A+C chord frame.
- `remote.forget`: `{profile:4|6|12}`.
- `config.wifi`: `{suffix:string,password:string}`; immutable prefix Genius_Air,
  optional '-' suffix, entire SSID <=32 UTF-8 bytes, password 8..63 ASCII chars;
  empty password retains existing. Save then restart AP only after ack.
- `config.lighting`: `{enabled:boolean,hue:0..359,brightness:0..100,effect:
  solid|breathe|rainbow|slowblink|chase|wave,period:0.5..10}` seconds.
- `config.reset`: `{}`; stop outputs including compressor, reset mappings,
  routes, RGB and Wi-Fi suffix; preserve installation secrets/identity.

Configuration writes require idle valve outputs and atomic successful storage;
reject on failure instead of falsely acknowledging persistence. Wi-Fi change and
reset acknowledge before AP restart. Configuration must be fetched from device,
not reconstructed from optimistic app state. User edits may be local drafts.

Action strings: `disabled`, `fill-fl`, `fill-fr`, `fill-rl`, `fill-rr`,
`exhaust-fl`, `exhaust-fr`, `exhaust-rl`, `exhaust-rr`, `fill-front`,
`exhaust-front`, `fill-rear`, `exhaust-rear`, `fill-all`, `exhaust-all`,
`preset-clicks`, `pulse`.

## Status shape

`{revision:uint,uptime:uint,fillMask:0..15,exhaustMask:0..15,
 compressorCommand:boolean,sourcePreset:0..3,mode:"idle"|"hold"|"pulse"|
 "transition"|"rf",owner:string|null,
 capabilities:{rgb:boolean,rf:boolean,motion:true},
 config:{wifi:{ssid,suffix,passwordSet:true},
 routes:{"1-2":[{flow:0|1|2,ms},...4],...configured only},
 mappings:{"4":[...4action strings],"6":[...6],"12":[...12]},
 lighting:{enabled,hue,brightness,effect,period}},
 learning:null|{profile,button,remainingMs},lastRf:null|{profile,button},
 remoteSlots:{"4":[...5booleans],"6":[...6booleans],"12":[...12booleans]},
 learnResult:null|{id:uint32,profile,button,ok:boolean,error:null|"rf-storage-failed"}}`.
Source 0 means unknown. owner is a connection id or 'rf', not a secret. Never
include Wi-Fi password or control key in status/logs. revision increments when
reported output/mode/config changes. Reject malformed or inconsistent status in
the app; never render it as a successful connected state.

For profile4 remoteSlots[0] is the learned A+C chord and slots1..4 are A..D.
Other profiles use button-1 indexing. remote.learn ACK means only 'armed'.
New learning clears learnResult; durable capture success/failure sets a new
monotonic result id. Success has error:null; failure has rf-storage-failed and
does not change the durable slot. The app must not claim a code was saved merely
because a learn command or RF frame was acknowledged.

## Device boundaries

On boot/reset all output bits and compressor OFF; never resume automation.
On disconnect/keepalive expiry stop that client's valve/automatic ownership;
other owners cannot be stopped by a stale hold.end. Manual/RF preempts auto.
Every wheel has >=100 ms OFF before reversing direction, including across
stop/disconnect/error. Independent watchdog must be serviced only by normal
loop execution, except bounded500ms initialization with all outputs inhibited.
Normal20ms falling WDI pulses and a latched150ms lateness fault pair with
TPS3431factory200ms plus a74LVC1G74 sticky permit gatingOE/compressor;
rawWDOauto-recovery mustnever re-enable oldoutputs. Seehardware/rev_b/PINOUT.fa.md.
Don't block timers/network during valve demand or flash writes.
RF requires actual captured codes/protocol, not codes inferred from photos.
Supported decoded fixed-code repeats can drive manual leases; absent/stale RF
closes manual demand. Accepted one-shot RF pulse/preset runs continue through
normal release; unknown decoded input cancels RF-owned operations. A+C toggles
once after 3 s, suppresses A/C until full release.
RGB capabilities must reflect configured hardware type/count/current budget;
do not claim a measured 500 mA limit or measured height/pressure.
