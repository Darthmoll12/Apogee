# APOGEE Log Spec — v0

## File
- One file per recording: `AVddhhmm.CSV` (day, hour, minute from RTC; RTC = label only)
- ASCII CSV, `\n` line endings. First line is the column header.
- A recording started in the same minute as a previous one APPENDS to the
  same file, so the header line can reappear mid-file.
- Flushed every 200 rows (~0.5 s). Power loss can truncate the last line.

## Rows
One row per fast sample (nominal 400 Hz, hardware timer 2 ISR). Slow sensors
are sample-and-hold: the same cached value repeats on every row until the next
read (~every 100 ms, read in loop(), so timing is jittery and NOT logged).

| column       | type    | meaning |
|--------------|---------|---------|
| timestamp    | uint32  | micros() at ISR entry. Wraps at 2^32 us (~71.6 min). |
| ecg          | 0–4095  | AD8232 output, raw 12-bit counts (A0) |
| leadsoff     | 0/1     | 1 = LO- OR LO+ asserted. Which lead is NOT recoverable. |
| ppg1         | 0–4095  | Pulse Sensor Amped, raw counts (A1). Site: TBD |
| ppg2         | 0–4095  | Pulse Sensor Amped, raw counts (A2). Site: TBD |
| imu_t        | uint32_t| Time that IMU sensor was read, can be compared to `timestamp` from ECG and PPG signals |
| accel_x/y/z  | float   | m/s^2, LSM6DS3TRC, held, ~10 Hz, 2 decimals |
| gyro_x/y/z   | float   | rad/s, same, 2 decimals |
| skin_temp    | float   | deg C, MCP9808, held, 2 decimals |

## Channel timing
- timestamp is taken at ISR entry; ECG, LO pins, PPG1, PPG2 are then read
  sequentially (skew = TBD, measure it, expect well under 1 ms).
- Sampling is NOT guaranteed uniform. Use timestamps, never row index / nominal fs.

## Unknowns to fill in (verify, don't assume)
- ADC full-scale: firmware never calls analogReference(), so this is the core's default
- IMU range / ODR: library defaults, read them back via the getters
- Effective fs and jitter: measure from np.diff(timestamp)

## Parser rules
- Drop rows that fail numeric coercion (repeated headers, truncated last line); count them.
- Unwrap timestamp rollover (diff < -2^31 => add 2^32).
- Gap detection: dt > ~1.5x the MEDIAN dt => dropped samples;
  n_missing ~= round(dt / median_dt) - 1.
- dt > ~1 s => new segment (recording restarted in the same file).
- Recover IMU/temp read times by detecting value CHANGES (resolution ~ +/-1 row).