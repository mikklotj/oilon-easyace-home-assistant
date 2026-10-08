# Oilon EasyAce heat pump in Home Assistant

A Home Assistant **package** (plain YAML, no custom component) that reads an
**Oilon EasyAce** ground-source heat pump over the local network:

- **Modbus TCP**: status, alarms, temperatures, power, energy counters, compressor
  data
- **Controller web server JSON API** (`jsongen.html`): outdoor temperature,
  pressures, average COP, heat curve, and optional heat curve writes

The package comes from a working installation (Home Assistant 2026.9) and has
been cleaned up for sharing. It is **not** an official integration and is not
affiliated with or endorsed by Oilon. See [Safety](#safety).

*Suomenkielinen tiivistelmä: [Suomeksi](#suomeksi).*

## Contents

```
packages/oilon_easyace.yaml        the package (Modbus + REST + templates)
secrets.example.yaml               the secrets the package expects
docs/registers.md                  Modbus registers and JSON object addresses (OA)
docs/status-codes.md               status codes, alarm words
examples/status_fi.yaml            optional Finnish status sensor
examples/electric_heater_notification.yaml
examples/heat_curve_script.yaml    heat curve write script (read Safety first)
examples/energy_dashboard.md       Energy dashboard and COP sensors
```

## What you get

| Group | Entities (examples) |
|---|---|
| Status | `sensor.oilon_easyace_status` (text), `..._status_code`, `binary_sensor.oilon_easyace_compressor`, `binary_sensor.oilon_easyace_alarm`, 4 × alarm words |
| Temperatures | heating flow, brine in/out, hot water tank, heating and hot water targets, evaporating/condensing, outdoor, average outdoor, heating circuit 2 flow |
| Power | electrical power, heating power, ground circuit power, electric heater power, current, compressor frequency, relative power |
| Energy (kWh) | electrical, heating and ground energy, electric heater energy |
| Performance | instantaneous COP, controller average COP, compressor runtime and starts |
| Refrigerant | evaporating/condensing temperature and pressure |
| Heat curve | circuit 2 curve maximum, minimum and points 1–6 (read, optional write) |

The full list with addresses, units and observed ranges is in
[docs/registers.md](docs/registers.md).

## How it works

The heat pump controller has two local interfaces:

1. **Modbus TCP** on port 502. Home Assistant's built-in
   [Modbus integration](https://www.home-assistant.io/integrations/modbus/) reads
   holding registers.
2. **A web server** with a JSON endpoint:
   `http://<host>/jsongen.html?FN=Read&OA=<object address>&PIN=<pin>&LNG=-1&US=1`,
   protected with HTTP basic auth. Home Assistant's
   [RESTful integration](https://www.home-assistant.io/integrations/rest/) polls
   it, and [`rest_command`](https://www.home-assistant.io/integrations/rest_command/)
   sends `FN=Write` requests. Each value is identified by an **OA** (object
   address), a short Base64 string such as `AyKQeXo7AAE=`.

Some values are only available through one interface. That is why the package
uses both.

### Which controller is this?

- The `jsongen.html?FN=Read|Write&OA=<base64>&PIN=<pin>` URL format is the JSON
  interface of **Siemens Climatix** controllers. Climatix is a controller family
  for equipment makers (OEMs) of, among others, heat pumps
  ([ebm-papst: Siemens Climatix OEM controls](https://www.ebmpapst.com/ae/en/products/control-electronics/approved-controllers/Siemens/siemens-building-technology-controller.html)).
  An independent .NET library for the Climatix JSON API uses the same
  `JSONGEN.HTML?FN=`, Base64 point ids, PIN and basic auth
  ([markroskaric/ClimatixRestApi](https://github.com/markroskaric/ClimatixRestApi)).
  [st0ff3r/hvac_grapher](https://github.com/st0ff3r/hvac_grapher) also polls a
  `jsongen.html?fn=read&pin=…&oa=…` endpoint.
- **Unverified:** that the EasyAce controller *hardware* is a Siemens Climatix
  model. Only the API matches. Check the label on the controller or ask Oilon.

### Does Oilon publish a Modbus register list?

As of October 2026, no public Modbus register list for the EasyAce turned up in
searches of oilon.com and [docs.oilon.com](https://docs.oilon.com/docs/public/ohpc/index.html).
The documentation there covers Oilon's own service tools (Heat Pump Configurator,
Local Monitor). Oilon also publishes an
[EasyAce quick guide (PDF)](https://oilon.com/wp-content/uploads/2023/03/M8007_EasyAce_quick_guide.pdf)
for the EasyAce app. It was not checked for Modbus details. Ask Oilon or your installer for
the register list of your model and software version. If you get an official
list, please link it in an issue rather than copying the PDF into this repository.

All registers in this repository were mapped on one unit by comparing values
with the controller's display and web UI.

### Finding OA codes for your own unit

Open the heat pump's web UI in a desktop browser, open the developer tools
(Network tab, filter `jsongen`) and watch the requests the page sends. Each
request lists the OA codes it reads, and the response shows the values. Changing
a setting once in the web UI shows the matching `FN=Write` request. Step-by-step
instructions: [docs/registers.md → Finding OA codes](docs/registers.md#finding-oa-codes-for-your-own-unit).

## Requirements

- Home Assistant 2024.10 or newer (the examples use the `triggers:`/`actions:`
  syntax). Developed on 2026.9.
- An Oilon EasyAce heat pump with its controller connected to your LAN.
  **A wired connection is strongly recommended.** See
  [Troubleshooting](#troubleshooting).
- Modbus TCP reachable on port 502 of the controller. Test with
  `nc -vz <host> 502`. If the port is closed, ask your installer whether Modbus TCP
  can be enabled on your unit.
- For the REST part: the controller web UI **PIN** and the password of the
  controller's **JSON API user** (`JSON` on the reference unit). If you do not
  know them, ask your installer. This repository does not document any default
  credentials.
- A fixed IP address for the heat pump (DHCP reservation in your router).

## Installation

1. **Enable packages** in `configuration.yaml`, if you have not already:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

2. **Copy** `packages/oilon_easyace.yaml` to `<config>/packages/oilon_easyace.yaml`.

3. **Add the secrets.** Copy the entries from
   [`secrets.example.yaml`](secrets.example.yaml) into your `<config>/secrets.yaml`
   and replace `<HOST>`, `<PIN>` and `<PASSWORD>`.

   Why are complete URLs stored as secrets? The controller expects its PIN in
   the query string. `!secret` can only replace a *whole* YAML value (not part of
   a string), and templates such as `resource_template` cannot read secrets. Storing
   each full URL in `secrets.yaml` keeps the PIN out of the package file. The
   write URL secret contains the Jinja placeholders `{{ oa }}` and `{{ value }}`.
   They work because `rest_command`'s `url` is a template field, which Home
   Assistant renders after it has loaded the secret.

4. If you only want Modbus, delete the `rest:` and `rest_command:` sections, and
   the REST secrets are then not needed.

5. **Check the configuration** (Developer tools → YAML → Check configuration) and
   **restart** Home Assistant.

6. Optional extras in [`examples/`](examples/): Finnish status text, electric
   heater notification, Energy dashboard and COP sensors, heat curve script.

### Upgrading from an earlier personal version

The `unique_id`s are the same as in the original installation. If entities with
these unique IDs already exist, Home Assistant keeps their **existing entity IDs**,
and the templates at the end of the package (which use the new English entity
IDs) will not find them. Either rename the entities in the UI to match, or change
the entity IDs in the templates.

## Safety

- **Reading** values is harmless. **Writing** (the `rest_command` and
  [`examples/heat_curve_script.yaml`](examples/heat_curve_script.yaml)) changes how
  your heat pump heats your home. A wrong heat curve can make the house too cold
  (frozen pipes in winter), too hot (underfloor heating limits), or make the heat
  pump use the expensive electric heater.
- Only write to objects you have identified on **your own unit**, with values you
  could also set from the controller's own UI. Write down the original values
  first.
- Do not run write automations in a tight loop. The controller may store settings
  in non-volatile memory, which has limited write endurance (unverified for this
  controller, but common for such devices).
- The JSON API PIN gives access to settings. Keep it in `secrets.yaml`, do not put
  the heat pump on the internet, and do not share browser HAR files or
  screenshots that show `PIN=` or the `Authorization` header.
- Use at your own risk. This project is not affiliated with, endorsed by or
  supported by Oilon. "Oilon" and "EasyAce" are names of their owners and are used
  here only to identify the product.

## Known quirks

| Symptom | Cause / fix |
|---|---|
| Missing suction/discharge gas temperature, superheat or heating return | Left out on purpose: on the reference unit these registers only returned constant values. See [registers.md](docs/registers.md#registers-left-out-of-the-package). |
| Pressure shows 999.9 bar | The value is unavailable. Handled with availability templates. |
| Energy counters show billions of kWh | 32-bit registers need `swap: word`. Already set in the package. |
| COP jumps to 10–15 | Instantaneous COP at compressor start/stop. Use average or daily COP ([examples/energy_dashboard.md](examples/energy_dashboard.md)). |
| Heating/ground power spikes above 30 kW | Rare, short spikes seen on the reference unit. Filter them with a `filter` sensor if they disturb graphs. |

## Troubleshooting

- **Modbus entities unavailable / timeouts in the log**
  - Use a wired network connection for the heat pump. On the reference
    installation, a Wi-Fi link caused frequent Modbus and REST timeouts.
  - Keep scan intervals modest (15–60 s, as in the package). Polling faster does
    not help and loads the controller.
  - If other systems also poll the controller over Modbus, stop them for a while
    to rule out conflicts.
  - Increase `timeout` in the `modbus:` hub (5 s in the package).
- **REST sensors unavailable**
  - Open the read URL from `secrets.yaml` in a browser on the same network. A
    login prompt means the basic auth details are wrong. An empty or error
    response usually means a wrong PIN or OA code.
  - Make sure every `=` in OA codes is written as `%3D` *inside the URLs*, but as
    `=` in the `value_template`s.
  - The `rest:` timeout is 15 s. Keep the number of REST resources small and put
    many `OA=` parameters in one URL, as the package does, so the controller gets
    fewer requests.
- **Values look wrong.** Compare with the controller display. Addresses and
  scaling may differ on your model or software version. Please open an issue with
  your model, software version and what you found.
- **Status shows "Unknown status N".** The code is not in the list. Please report
  the code and the text your controller shows.

## Related projects

As of October 2026, no other Home Assistant integration for Oilon heat pumps was
found on GitHub or the Home Assistant community forum. Related work:

- [markroskaric/ClimatixRestApi](https://github.com/markroskaric/ClimatixRestApi):
  .NET library for the Siemens Climatix JSON API (read and write by Base64 id).
- [andreasc1/homeassistant-climatix-ic](https://github.com/andreasc1/homeassistant-climatix-ic):
  Home Assistant integration for Siemens RDS110 thermostats through the
  **Climatix IC cloud**. Different device and cloud-based, but the same Climatix
  family.
- [Home Assistant Modbus integration](https://www.home-assistant.io/integrations/modbus/),
  [RESTful integration](https://www.home-assistant.io/integrations/rest/),
  [RESTful command](https://www.home-assistant.io/integrations/rest_command/).

If you know of another Oilon integration, please open an issue so it can be
linked here.

## Contributing

Issues and pull requests are welcome, especially:

- register or OA mappings for other EasyAce models and software versions (please
  name the model and software version)
- alarm word bit meanings and missing status codes
- corrections to anything marked *unverified*

Never include your PIN, passwords, IP addresses or HAR files in issues.

## Suomeksi

Tämä repositorio sisältää Home Assistant -paketin **Oilon EasyAce**
-maalämpöpumpun lukemiseen paikallisverkosta. Paketti ei ole Oilonin virallinen
eikä Oilonin tukema.

- **Modbus TCP** (portti 502): tila, hälytykset, lämpötilat, tehot,
  energialaskurit, kompressorin tiedot.
- **Ohjaimen web-palvelimen JSON-rajapinta** (`jsongen.html`, sama muoto kuin
  Siemens Climatix -ohjaimissa): ulkolämpötila, paineet, keskimääräinen COP ja
  lämpökäyrä. Lämpökäyrän pisteitä voi myös kirjoittaa (valinnainen).

**Asennus lyhyesti:** ota `packages` käyttöön `configuration.yaml`:ssa, kopioi
`packages/oilon_easyace.yaml` Home Assistantin `packages`-kansioon, kopioi
`secrets.example.yaml`:n rivit omaan `secrets.yaml`:iin ja korvaa `<HOST>`,
`<PIN>` ja `<PASSWORD>`. Koko URL on salaisuus, koska ohjaimen PIN-koodi kulkee
URL:ssa eikä `!secret` toimi merkkijonon osana. Tarkista konfiguraatio ja
käynnistä Home Assistant uudelleen.

**Huomioita:**

- Käytä lämpöpumpulle **kiinteää verkkoyhteyttä**. Wi-Fin kautta Modbus- ja
  REST-kyselyt aikakatkeavat usein. Pidä kyselyvälit maltillisina (15–60 s).
- Imukaasun ja kuumakaasun lämpötilat, tulistus ja lämmityksen paluu on jätetty
  pois, koska referenssilaitteessa ne näyttivät vain vakioarvoja. 999,9 bar
  tarkoittaa, että paine ei ole saatavilla.
- 32-bittiset energialaskurit tarvitsevat `swap: word` -asetuksen.
- Tilakoodit suomeksi: [`examples/status_fi.yaml`](examples/status_fi.yaml) ja
  [`docs/status-codes.md`](docs/status-codes.md).
- Omat OA-koodit löydät selaimen kehittäjätyökaluilla (Network-välilehti,
  suodatin `jsongen`) lämpöpumpun omasta web-käyttöliittymästä. Ohjeet:
  [docs/registers.md](docs/registers.md#finding-oa-codes-for-your-own-unit).

**Turvallisuus:** lukeminen on vaaratonta, mutta **kirjoittaminen muuttaa
lämmityksen toimintaa**. Väärä lämpökäyrä voi jäähdyttää talon, ylittää
lattialämmityksen rajat tai käynnistää kalliin sähkövastuksen. Kirjaa
alkuperäiset arvot ylös ennen muutoksia. Käyttö omalla vastuulla.

## License

[MIT](LICENSE)
