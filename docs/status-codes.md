# Status codes and alarm words

## Operating status (Modbus holding register 8)

This list was written down during the original setup of the reference unit, from
the status texts shown by the controller. It is **not an official Oilon list**
and has not been checked against Oilon documentation. Codes that are not in the
list show up as `Unknown status <n>`. If you see one, please report the code and
the text your controller shows for it.

The English texts are translations of the Finnish ones. The Finnish texts are
used in [`examples/status_fi.yaml`](../examples/status_fi.yaml).

| Code | English (package) | Suomeksi | Counts as "running" |
|---:|---|---|:---:|
| 0 | Alarm | Hälytys | |
| 1 | Waiting for start button | Odotan käynnistyspainiketta | |
| 2 | Waiting for remote run permission | Odotan etäkäyntilupaa | |
| 9 | Stopping | Pysähtymässä | |
| 10 | No heating demand | Ei lämmitystarvetta | |
| 11 | No cooling demand | Ei jäähdytystarvetta | |
| 17 | Waiting for restart interval | Odotan käynnistysväliä | |
| 18 | Compressor not ready | Kompressori ei ole valmis | |
| 19 | Waiting for discharge gas to cool down | Odotan kuumakaasun jäähtymistä | |
| 20 | Waiting for evaporator flow | Odotan höyrystinvirtausta | |
| 23 | Ground circuit cold | Maapiiri kylmä | |
| 24 | Waiting for condenser flow | Odotan lauhdutinvirtausta | |
| 25 | Condenser hot | Lauhdutin kuuma | |
| 26 | Condenser cold | Lauhdutin kylmä | |
| 27 | Ground circuit hot | Maapiiri kuuma | |
| 30 | Ready to start | Valmis käynnistymään | |
| 31 | Remote control | Etäohjaus | |
| 49 | Starting | Käynnistymässä | |
| 50 | Running | Käynnissä | ✓ |
| 51 | Heating space heating water | Lämmittää lämmitysvettä | ✓ |
| 52 | Heating domestic hot water | Lämmittää käyttövettä | ✓ |
| 54 | Legionella cycle | Legionellatappo | ✓ |
| 55 | Cooling | Jäähdytys | ✓ |
| 56 | Heating space heating water, electric heater on | Lämmittää lämmitysvettä, vastus mukana | ✓ |
| 57 | Heating domestic hot water, electric heater on | Lämmittää käyttövettä, vastus mukana | ✓ |
| 58 | Legionella cycle, electric heater on | Legionellatappo, vastus mukana | ✓ |
| 59 | Remote control | Etäohjaus | ✓ |
| 61 | Evaporator cold | Höyrystin kylmä | |
| 62 | Condenser hot | Lauhdutin kuuma | |
| 63 | Current limit | Virtaraja | ✓ |
| 70 | Heating space heating water with electric heater only | Lämmittää lämmitysvettä vastuksella | ✓ |
| 71 | Heating domestic hot water with electric heater only | Lämmittää käyttövettä vastuksella | ✓ |
| 72 | Legionella cycle with electric heater only | Legionellatappo vastuksella | ✓ |

"Counts as running" is the set used by `binary_sensor.oilon_easyace_compressor`.
Codes 70–72 mean the electric heater is producing heat, but the compressor
probably is not. If you only want the compressor, use
`sensor.oilon_easyace_compressor_frequency > 0` instead.

Codes 25 and 62 both read "condenser hot", and codes 31 and 59 both read "remote
control". They probably come from different stages of the control sequence
(waiting vs. running). This is unverified.

## Compressor status (register 9)

Read as `sensor.oilon_easyace_compressor_status_code`, but not decoded. The
meaning of the values is unknown.

## Alarm words (registers 2–5)

Registers 2, 3, 4 and 5 are 16-bit words. Each bit is probably one alarm. The
package only checks whether any word is non-zero
(`binary_sensor.oilon_easyace_alarm`). The bit-to-alarm mapping is **unknown**.

To map bits on your own unit:

1. Write down the alarm text on the controller display or web UI when an alarm
   is active.
2. Look at the alarm word sensors in Home Assistant history at the same moment.
   To list the bits that are set, paste this into Developer tools → Template:

   ```jinja
   {% set w = states('sensor.oilon_easyace_alarm_word_1') | int(0) %}
   {% for b in range(16) if w | bitwise_and(2 ** b) %}bit {{ b }} {% endfor %}
   ```

3. Note which bit was set and contribute the mapping.

Example automation that notifies when an alarm starts:

```yaml
automation:
  - alias: "Heat pump alarm"
    triggers:
      - trigger: state
        entity_id: binary_sensor.oilon_easyace_alarm
        to: "on"
        for: "00:01:00"
    actions:
      - action: notify.notify
        data:
          title: "Heat pump alarm"
          message: >-
            Alarm words:
            {{ states('sensor.oilon_easyace_alarm_word_1') }} /
            {{ states('sensor.oilon_easyace_alarm_word_2') }} /
            {{ states('sensor.oilon_easyace_alarm_word_3') }} /
            {{ states('sensor.oilon_easyace_alarm_word_4') }}.
            Status: {{ states('sensor.oilon_easyace_status') }}
```
