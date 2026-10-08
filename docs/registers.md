# Registers and object addresses

Everything on this page was **observed on one Oilon EasyAce ground-source heat
pump** running in a home. It is not an official Oilon document. Addresses, scaling
and meanings may differ by model, controller software version and installer
configuration. Treat each row as a starting point and check it against the
controller's own display or web UI before you rely on it.

"Observed range" is the min/max of Home Assistant's hourly long-term statistics on
the reference unit over about four months (summer to autumn). Winter values are
not included.

## Modbus TCP

- Port `502`, unit/slave id `1` (Home Assistant default)
- Function: read holding registers (`input_type: holding`)
- Addresses are the 0-based addresses Home Assistant uses
- 16-bit values are signed (`int16`) with scale `0.01` unless noted
- 32-bit counters are `uint32` with the low word first, so Home Assistant needs
  `swap: word`

| Addr | Type | Scale | Unit | Meaning (as used in the package) | Observed range / notes |
|---:|---|---:|---|---|---|
| 0 | int16 | 1 | – | Connection test | Used only to check that the bus answers |
| 2 | uint16 | 1 | – | Alarm word 1 (bit field) | 0 = no alarm. See [status-codes.md](status-codes.md) |
| 3 | uint16 | 1 | – | Alarm word 2 (bit field) | 0 = no alarm |
| 4 | uint16 | 1 | – | Alarm word 3 (bit field) | 0 = no alarm |
| 5 | uint16 | 1 | – | Alarm word 4 (bit field) | 0 = no alarm |
| 8 | int16 | 1 | – | Operating status code | See [status-codes.md](status-codes.md) |
| 9 | int16 | 1 | – | Compressor status code | Not decoded (meaning unknown) |
| 13 | int16 | 0.01 | % | Relative power | 0 – 100 |
| 30 | int16 | 0.01 | °C | Suction gas temperature | Not in the package: constant 0.00 on the reference unit |
| 31 | int16 | 0.01 | °C | Discharge (hot) gas temperature | Not in the package: constant 0.00 on the reference unit |
| 32 | int16 | 0.01 | °C | Brine (ground loop) in | 6.0 – 23.3 |
| 33 | int16 | 0.01 | °C | Evaporating temperature | −1.8 – 20.0 |
| 34 | int16 | 0.01 | °C | Brine (ground loop) out | 1.2 – 23.0 |
| 35 | int16 | 0.01 | °C | Heating return | Not in the package: constant −310.82 on the reference unit |
| 36 | int16 | 0.01 | °C | Condensing temperature | 22.3 – 64.6 |
| 37 | int16 | 0.01 | °C | Heating flow (supply) | 21.3 – 63.6 |
| 42 | int16 | 0.01 | °C | Domestic hot water tank | 44.4 – 60.5 |
| 51 | int16 | 0.01 | A | System current | 0 – 11.1 |
| 63 | int16 | 0.01 | °C | Heating target (supply setpoint) | 21.0 – 27.6 |
| 64 | int16 | 0.01 | °C | Hot water target | 55.5 – 60.0 |
| 76 | int16 | 0.01 | K | Suction superheat | Not in the package: constant +310.82 on the reference unit |
| 77 | int16 | 0.01 | Hz | Compressor frequency | 0 – 50 |
| 78 | int16 | 0.01 | kW | Electric (backup) heater power | 0 – 6.0 |
| 79 | int16 | 0.01 | kW | Ground circuit power (heat taken from the ground) | 0 – 34.7 (occasional implausible spikes) |
| 80 | int16 | 0.01 | kW | Electrical input power | 0 – 7.0 |
| 81 | int16 | 0.01 | kW | Heating output power | 0 – 37.1 (occasional implausible spikes) |
| 82 | uint32 | 1 | kWh | Ground energy, cumulative | Tens of MWh. Needs `swap: word` |
| 84 | uint32 | 1 | kWh | Electrical energy, cumulative | Needs `swap: word` |
| 86 | uint32 | 1 | kWh | Heating energy, cumulative | Needs `swap: word` |
| 89 | int16 | 0.01 | – | Heating COP (instantaneous) | 0 – 15.4 (high values are start/stop transients) |
| 92 | uint32 | 1 | h | Compressor running hours | Needs `swap: word` |
| 94 | uint32 | 1 | – | Compressor starts | Needs `swap: word` |

### Registers left out of the package

On the reference unit, registers 30, 31, 35 and 76 (suction gas temperature,
discharge gas temperature, heating return, suction superheat) only ever returned
constant values (0.00, 0.00, −310.82 and +310.82), whatever the compressor was
doing. Either the sensors are not used in this configuration or the mapping
differs for this firmware (unverified). They are listed above for completeness
but are not part of the package. If your unit shows real values there, add them
back as `int16`, `scale: 0.01` sensors.

Notes:

- All temperature registers in the package set `nan_value: 31082`, so a raw
  ±31082 is reported as *unknown* instead of ±310.82 (Home Assistant's Modbus
  integration checks both signs).
- The REST temperatures have an availability template (`abs < 300`) for the same
  reason.
- The ratio of cumulative heating energy to electrical energy on the reference
  unit matched the controller's own "average COP" object (about 4.5). That is
  how the kWh unit of the counters was checked.

### Unknown registers

Only the registers above were mapped. Other addresses have not been checked. If
you find out what they mean, please
open an issue or a pull request and say which model and software version you
have.

## Controller web server JSON API (`jsongen.html`)

The controller's built-in web server has a JSON endpoint. Its URL format matches
the JSON interface of Siemens Climatix controllers. See the README section
"Which controller is this?" for sources and caveats.

```
GET http://<host>/jsongen.html?FN=Read&OA=<oa1>&OA=<oa2>&...&PIN=<pin>&LNG=-1&US=1
GET http://<host>/jsongen.html?FN=Write&OA=<oa>;<value>&PIN=<pin>&LNG=-1&US=1
Authorization: Basic (user "JSON" on the reference unit)
```

- `OA` is an *object address*: an opaque Base64 string (8 bytes). It must be
  URL-encoded in the query string (`=` → `%3D`, `;` → `%3B`). Several `OA`
  parameters can be read in one request.
- `PIN` is the web UI PIN. It decides which objects you can read and write.
- `LNG` and `US` are copied from the requests the web UI sends. They appear to
  select the language and the unit system. **This is unverified.**
- The response looks like `{"values": {"<oa>": [<value>, ...], ...}}`. Most objects
  return a list and the value is the first element. On the reference unit,
  `ACIdJ096AAE=` (average COP) returned a plain number.

### Object addresses (OA) used

| OA code | Meaning | Unit | R/W in the package | Observed range / notes |
|---|---|---|---|---|
| `AyKQeXo7AAE=` | Outdoor temperature | °C | R | 4.5 – 33.4 (summer–autumn) |
| `AyJzpXo7AAE=` | Heating circuit 2 supply temperature | °C | R | 21.5 – 26.1 |
| `ACPOV3o7AAE=` | Average (filtered) outdoor temperature | °C | R | 7.9 – 24.1 |
| `ACOUmNm6AAE=` | Compressor start delay remaining | s | R | 0 – 4 |
| `ACIdJ096AAE=` | Average COP | – | R | ≈ 4.5. Returns a number, not a list |
| `ACMpF096AAE=` | Electric heater energy, cumulative | kWh | R | Small values (heater rarely used) |
| `AyJSdHo7AAE=` | Evaporating pressure | bar | R | 999.9 = value unavailable |
| `AyJbCXo7AAE=` | Condensing pressure | bar | R | 999.9 = value unavailable |
| `ACMk5cPwAAE=` | Circuit 2 heat curve: maximum supply temp | °C | R/W | Setpoint |
| `ACN6HsPwAAE=` | Circuit 2 heat curve: minimum supply temp | °C | R/W | Setpoint |
| `ACN1L8PwAAE=` | Circuit 2 heat curve: point 1 | °C | R/W | Setpoint, coldest outdoor temperature point* |
| `ACMWH8PwAAE=` | Circuit 2 heat curve: point 2 | °C | R/W | Setpoint* |
| `ACM3D8PwAAE=` | Circuit 2 heat curve: point 3 | °C | R/W | Setpoint* |
| `ACPQf8PwAAE=` | Circuit 2 heat curve: point 4 | °C | R/W | Setpoint* |
| `ACPxb8PwAAE=` | Circuit 2 heat curve: point 5 | °C | R/W | Setpoint* |
| `ACOSX8PwAAE=` | Circuit 2 heat curve: point 6 | °C | R/W | Setpoint, warmest outdoor temperature point* |

\* The reference installation's dashboard labels points 1–6 as outdoor
temperatures −32, −22, −12, −2, +8 and +18 °C. These breakpoints **have not been
checked against Oilon documentation**. Check the curve screen of your own
controller.

On the reference unit, writes have only been used on the eight heat curve
objects above, in 0.5 °C steps. Other objects have not been written.

### Finding OA codes for your own unit

OA codes are not published. You can find them in the requests that the controller's
own web UI sends:

1. Open the heat pump's web UI in a desktop browser on your LAN and log in as
   usual.
2. Open the developer tools (F12) and go to the **Network** tab. Filter by
   `jsongen`.
3. Open the page in the web UI that shows the value you want (for example the
   heat curve screen). Each request lists the `OA=` parameters it reads. The
   **Response** tab shows `{"values": {"<oa>": ...}}`, so you can match each code
   to the number shown on screen.
4. Change a value through the web UI once and look for the `FN=Write` request.
   It shows the OA code and the format (`<oa>;<value>`) of that setting.
5. Decode `%3D` back to `=` when you copy codes into YAML templates. Keep them
   URL-encoded inside the resource URLs.

Do not share screenshots or HAR files without removing the `PIN=` parameter and
the `Authorization` header first.
