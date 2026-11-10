# anviz-tcb specification

anviz-tcb encodes and decodes the TC_B protocol that Anviz time clocks speak. It
covers frames, the commands a server sends, and the clock's data: users, PINs,
punches, dates and record counts. It does no networking and keeps no state.

Everything here was verified against a real Anviz G100-N (firmware `02.07.A1`).
Implementations must pass every case in `tests.yaml`.

## Conventions

- A byte string is a sequence of bytes. Implementations use the language's
  native type (a byte array, `bytes`, a binary `String`). `tests.yaml` writes
  byte strings as lowercase hex.
- BE is big-endian, LE little-endian.
- `ch` is the device id: always 4 bytes. `00000000` addresses any device.
- An error means the function raises, throws or returns an error value, as is
  idiomatic in the language. Tests only check that an error happens, not which.
- Names below are suggestions; follow the language's naming style
  (`decode_punch`, `decodePunch`, `DecodePunch`).

## Checksum and frames

### crc16(data) -> integer

CRC-16/MCRF4XX of a byte string, 0 to 65535:

```
crc = 0xFFFF
for each byte b:
    crc = crc XOR b
    repeat 8 times:
        if crc AND 1: crc = (crc >> 1) XOR 0x8408
        else:         crc = crc >> 1
return crc
```

No final XOR. `crc16("123456789")` is `0x6F91`.

### has_ret(cmd) -> boolean

True when `cmd >= 0x80`. Responses, realtime punches (`0xDF`) and the pong carry
a status byte (RET) after the command; commands and the heartbeat don't.

### encode_frame(ch, cmd, data, ret = 0) -> bytes

```
without RET (cmd < 0x80):   A5 | ch(4) | cmd(1)          | len(2 BE) | data | crc(2 LE)
with RET (cmd >= 0x80):     A5 | ch(4) | cmd(1) | ret(1) | len(2 BE) | data | crc(2 LE)
```

`len` is the length of `data`. The CRC covers every byte from `A5` through the
end of `data`, and is written low byte first. `ret` is ignored when the command
has no RET.

Errors: `ch` isn't 4 bytes; `data` is longer than 65535 bytes.

### split_frames(buffer) -> { frames, rest }

Splits bytes received from the clock into complete frames. `rest` holds a
trailing partial frame: prepend it to the next bytes received.

Each frame is `{ ch, cmd, ret, data }`: `ch` and `data` are byte strings, `cmd`
and `ret` integers, and `ret` is null for frames without RET. `rest` is a byte
string too.

The algorithm, which the tests pin down exactly:

1. If the buffer is empty, stop.
2. If the first byte isn't `A5`, drop bytes up to the next `A5`. If there's no
   `A5`, drop everything and stop. Go to 1.
3. If fewer than 6 bytes remain, stop.
4. Read `cmd` at offset 5. With RET, the header is 9 bytes (`ret` at offset 6,
   `len` at 7 to 8); without, 8 bytes (`len` at 6 to 7). If the header isn't
   complete, stop.
5. The frame is `header + len + 2` bytes. If it isn't complete, stop.
6. If the CRC doesn't match, drop the first byte (the `A5`) and go to 2.
7. Otherwise emit the frame, remove it, and go to 1.

`rest` is whatever remains when the loop stops.

### pong(ch) -> bytes

The reply to a heartbeat: `encode_frame(ch, 0xFF, "", ret = 0)`. With `ch`
`00000000` it is `a500000000ff0000003cca`.

## Commands

Every command is `encode_frame(ch, code, payload)`, with no RET. The clock
answers with `code | 0x80`.

| Function | Code | Payload |
|---|---|---|
| `cmd_get_device_type(ch)` | `0x48` | none |
| `cmd_get_serial(ch)` | `0x24` | none |
| `cmd_get_datetime(ch)` | `0x38` | none |
| `cmd_set_datetime(ch, datetime)` | `0x39` | `encode_datetime(datetime)` |
| `cmd_get_record_counts(ch)` | `0x3C` | none |
| `cmd_get_punches(ch, kind, count)` | `0x40` | `[kind, count]` |
| `cmd_get_users(ch, kind, count)` | `0x72` | `[kind, count]` |
| `cmd_set_users(ch, users)` | `0x73` | `[number of users]`, then `encode_user` of each |
| `cmd_delete_user(ch, user_id, parts)` | `0x4C` | `user_id` (5 bytes BE), then the parts byte |
| `cmd_clear_punches(ch, mode, amount = 0)` | `0x4E` | `[mode]`, then `amount` (3 bytes BE) |
| `cmd_get_fingerprint(ch, user_id, finger)` | `0x44` | `user_id` (5 bytes BE), `[finger]` |
| `cmd_set_fingerprint(ch, user_id, finger, template)` | `0x45` | `user_id` (5 bytes BE), `[finger]`, `template` |

- `cmd_get_punches` kind: `"all"` = 1, `"new"` = 2, `"continue"` = 0. Count is 1
  to 25. Send `"all"` or `"new"` first, then `"continue"` until a reply holds no
  punches.
- `cmd_get_users` kind: `"first"` = 1, `"continue"` = 0. Count is 1 to 8.
- `cmd_set_users` takes 1 to 8 users. The reply's data has one bit per user, set
  when stored (bit 0 for the first).
- `cmd_delete_user` parts: a list of `"finger1"` (1), `"finger2"` (2), `"pin"` (4)
  and `"card"` (8), added together. Absent or empty means the whole user: `0xFF`.
- `cmd_clear_punches` mode: `"delete_all"` = 0 deletes every punch (users stay);
  `"mark_read"` = 1 marks every punch as read; `"mark_read_count"` = 2 marks the
  first `amount` unread punches as read.
- `finger` is 1 or 2. A template is exactly 338 bytes.

Errors: an unknown `kind`, `mode` or part; a count out of range; zero or more
than 8 users; `finger` not 1 or 2; a template that isn't 338 bytes; any value
too large for its field.

## Dates and times

### encode_datetime({ year, month, day, hour, minute, second }) -> bytes

Six bytes: `[year - 2000, month, day, hour, minute, second]`, in the clock's
local time. 2026-11-07 17:00:20 is `1a0b07110014`.

Errors: year before 2000 or after 2255; not a real date or time (month 13,
February 30, hour 24).

### decode_datetime(bytes) -> string

The reverse, as `YYYY-MM-DDTHH:MM:SS` with no time zone.

Errors: not exactly 6 bytes; not a real date or time.

## Identity

### decode_device_type(bytes) -> string and decode_serial(bytes) -> string

The data of the `0xC8` and `0xA4` replies: ASCII up to the first `00` byte or
the end, with surrounding whitespace removed. `473130302d4e0000` is `G100-N`.

## Punches

### decode_punch(bytes) -> punch

A punch record is 14 bytes:

| Bytes | Field |
|---|---|
| 0 to 4 | user id, BE |
| 5 to 8 | seconds, BE |
| 9 | method |
| 10 | state |
| 11 to 13 | work code, BE |

The result is `{ user_id, time, method, state, work_code }`:

- `time` is 2000-01-01T00:00:00 plus the seconds plus one day, as
  `YYYY-MM-DDTHH:MM:SS`. Use plain calendar arithmetic: no time zone, no daylight
  saving (in practice, compute in UTC and print without a zone). The clock counts from a point exactly one day off, so a
  count of 0 is `2000-01-02T00:00:00`.
- `method`: 1 or 2 is `"fingerprint"` (finger 1 or 2), 4 is `"pin"`, 8 is
  `"card"`. Any other value N is `"unknown(N)"`, N in decimal.
- `state`: 0 `"in"`, 1 `"out"`, 2 `"break-out"`, 3 `"break-in"`, 4
  `"overtime-in"`, 5 `"overtime-out"`, otherwise `"unknown(N)"`.

Errors: not exactly 14 bytes.

### decode_punch_batch(bytes) -> list of punches

The data of a `0xC0` reply: one count byte, then that many 14-byte records. A
count of 0 is an empty list.

Errors: empty input; a length other than `1 + 14 x count`.

## PINs

### pin_encode(pin) -> bytes

`pin` is a string of 1 to 6 ASCII digits, or null (or empty) for no PIN.

```
n = number of digits, v = the PIN's value
bytes = [(n << 4) OR ((v >> 16) AND 0x0F), (v >> 8) AND 0xFF, v AND 0xFF]
```

No PIN is `ffffff`. `1234` is `4004d2`; `0042` is `40002a` (the digit count
keeps the leading zeros).

Errors: anything other than digits; more than 6 digits.

### pin_decode(bytes) -> string or null

`ffffff` is null. Otherwise `n` is the top 4 bits of byte 0, `v` is the other 20
bits, and the PIN is `v` in decimal, left-padded with zeros to `n` digits.

Errors: not exactly 3 bytes.

## Users

### encode_user(user) -> bytes

`user` is `{ user_id, name, pin = null, card = null, method = 0, admin = false }`.
The result is 40 bytes:

| Bytes | Value |
|---|---|
| 0 to 4 | `user_id`, BE |
| 5 to 7 | `pin_encode(pin)` |
| 8 to 11 | `card`, BE; `ffffffff` when null |
| 12 to 31 | the first 10 Unicode code points of `name`, UTF-16 BE, padded with `00` |
| 32 | `01` (department) |
| 33 | `01` (group 1: always allowed) |
| 34 | `method` |
| 35 to 36 | `0000` (enrolled fingers: the clock ignores this on upload) |
| 37 | `00` when there's a PIN (not null, not empty), `ff` when not |
| 38 | `ff` |
| 39 | `c0` when `admin`, `40` when not |

- `method` is the sign-in method: 0 any, 1 card + finger, 2 PIN + finger, 3
  card + PIN, 4 ID + finger.
- Byte 37 is the PIN's high byte. Writing `ff` there with a PIN corrupts it on
  the clock.
- The name is not folded here; call `normalize_device_name` first if the clock
  should show it.

Errors: `user_id` outside 0 to 2^40 - 1; `card` outside 0 to `fffffffe`
(`ffffffff` means no card); an invalid PIN; `method` outside 0 to 255; a name
whose first 10 characters need more than 20 bytes in UTF-16 (only possible with
characters outside the Basic Multilingual Plane).

### decode_user(bytes) -> user

The reverse: `{ user_id, name, pin, card, method, admin, fingers }`.

- `name`: UTF-16 BE code units from bytes 12 to 31, up to the first `0000` code
  unit. Code units are 2-byte aligned: `0100 0041` is "ĀA", not "Ā".
- `card`: null for `ffffffff`.
- `admin`: true when the top two bits of byte 39 are both set.
- `fingers`: the enrolled fingers, from bytes 35 to 36 as a BE number: bit 0 is
  finger 1, bit 1 finger 2. A list in ascending order, such as `[1]` or `[1, 2]`.
- Bytes 32, 33, 37 and 38 are ignored. The clock sets byte 32 to `00` and byte
  38 to `00` on users it stores.

Errors: not exactly 40 bytes.

### normalize_device_name(name) -> string

The clock stores UTF-16 but can't show accented letters. This folds a name to
plain ASCII, at most 10 characters:

1. Replace `ß` with `ss`, `Œ` with `OE`, `œ` with `oe`, `Ø` with `O`, `ø` with
   `o`, `Æ` with `AE`, `æ` with `ae`, and remove `¿` and `¡`.
2. Apply Unicode NFKD.
3. Remove every character above U+007F. This drops the accents NFKD split off,
   and letters with no ASCII form (`Łukasz` becomes `ukasz`).
4. Split on whitespace (space, tab, line feed, carriage return, vertical tab,
   form feed), drop empty pieces, and join with single spaces.
5. Keep the first 10 characters (code points). This can leave a trailing space:
   `Ana María López` becomes `Ana Maria `.

`Martín Núñez` becomes `Martin Nun`.

## Record counts

### decode_record_counts(bytes) -> { users, all, new }

The 18 data bytes of a `0xBC` reply: `users` from bytes 0 to 2, `all` (stored
punches) from 12 to 14, `new` (punches not yet marked read) from 15 to 17, all
BE. The other bytes aren't decoded.

Errors: not exactly 18 bytes.

## Session flow (guidance, not tested)

The library has no networking, but this is how its pieces fit together. In
client mode the clock dials the server on TCP 5010 and keeps the connection
open; the server sends commands, the clock replies.

- Feed every received chunk to `split_frames`, keeping `rest` for the next.
- Answer each heartbeat (`0x7F`) with `pong(ch)`, about every 10 seconds.
- Identify with `cmd_get_device_type("00000000")`; the reply's `ch` is the real
  device id. Use it from then on.
- Realtime punches arrive on their own as `0xDF` frames; `decode_punch` their
  data. The clock stores each punch twice; deduplicate by user and time.
- To catch up, read `decode_record_counts`, then `cmd_get_punches("new", 25)` and
  `"continue"` until empty, then `cmd_clear_punches("mark_read_count", n)` for
  the `n` read.
- Users: `cmd_get_users("first", 8)` and `"continue"` until fewer than 8 come
  back; `cmd_set_users` to create or update. A user rewrite replaces every
  field, so read the user first to keep their card.
- An admin locks the clock's menu. Promote only someone with a finger or card
  enrolled.

## Testing

### tests.yaml

`tests.yaml` holds language-neutral cases, grouped by function:

```yaml
function_name:
  - name: "human-readable description"
    input: ...          # the function's arguments, see the mapping below
    output: ...         # the expected result
    error: true         # instead of output: the call must fail
```

Values:

- Byte strings, in inputs and outputs, are lowercase hex strings. Each function
  says which of its results are byte strings (hex in the tests) and which are
  text (compared as is).
- `null` is the language's null (`nil`, `None`).
- Objects are maps with the keys shown. Compare outputs as whole values: every
  key, and lists in order.

Cases marked `(captured)` use bytes recorded from a real G100-N.

### Input mapping

| Function | `input` |
|---|---|
| `crc16`, `split_frames`, `decode_datetime`, `decode_device_type`, `decode_serial`, `decode_punch`, `decode_punch_batch`, `pin_decode`, `decode_user`, `decode_record_counts` | a hex string |
| `has_ret` | an integer |
| `pin_encode` | a string, or null |
| `normalize_device_name` | a string |
| `encode_frame` | `{ ch, cmd, data, ret? }` |
| `pong`, and commands without a payload | `{ ch }` |
| `cmd_set_datetime` | `{ ch, datetime }` |
| `cmd_get_punches`, `cmd_get_users` | `{ ch, kind, count }` |
| `cmd_set_users` | `{ ch, users }` |
| `cmd_delete_user` | `{ ch, user_id, parts? }` |
| `cmd_clear_punches` | `{ ch, mode, amount? }` |
| `cmd_get_fingerprint` | `{ ch, user_id, finger }` |
| `cmd_set_fingerprint` | `{ ch, user_id, finger, template }` |
| `encode_datetime` | `{ year, month, day, hour, minute, second }` |
| `encode_user` | `{ user_id, name, pin?, card?, method?, admin? }` |

A key marked `?` may be missing: use the default from the function's
description.

### Turning a case into a test

For each function and each case: decode hex inputs to bytes, call the function,
encode byte results back to lowercase hex, and compare with `output`. When the
case has `error: true`, assert that the call fails.

```yaml
decode_punch:
  - name: "PIN (captured)"
    input: "00000000053280d9db0400000000"
    output: {"user_id": 5, "time": "2026-11-07T18:32:59", "method": "pin", "state": "in", "work_code": 0}
```

becomes, in Python:

```python
def test_decode_punch_pin_captured():
    got = decode_punch(bytes.fromhex("00000000053280d9db0400000000"))
    assert got == {"user_id": 5, "time": "2026-11-07T18:32:59",
                   "method": "pin", "state": "in", "work_code": 0}
```

Prefer a test file that loads `tests.yaml` at run time and generates the cases,
so it stays in step when the cases change.
