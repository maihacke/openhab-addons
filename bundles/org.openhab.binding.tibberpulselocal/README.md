# Tibber Pulse Local Binding

This Binding reads the live data of a Tibber Pulse directly from the Tibber bridge on your local network.
It needs no Tibber account and no Internet access.
The bridge pushes a reading about every 3 seconds.
The Binding publishes it on the Channels of a `meter` Thing: net power, import and export energy and, if your meter reports them, voltage, current, power per phase and grid frequency.

## Related Bindings

- Use the [`tibber`](https://www.openhab.org/addons/bindings/tibber/) Binding if you need prices, consumption history, or the cloud-based Pulse live data.
- Use the [`smartmeter`](https://www.openhab.org/addons/bindings/smartmeter/) Binding if you read your meter directly through a serial or USB optical head, without the bridge.

## Supported Things

| Thing Type ID | Type   | Label               | Description                                                       |
|---------------|--------|---------------------|-------------------------------------------------------------------|
| `bridge`      | Bridge | Tibber Pulse Bridge | Connection to one Tibber bridge; receives the stream of readings. |
| `meter`       | Thing  | Tibber Pulse Meter  | One electricity meter, identified by the device id of its Pulse.  |

Add the `bridge` Thing first, preferably by approving its Inbox entry, see [Discovery](#discovery), or manually.
A `meter` Thing always belongs to one `bridge` Thing.
The `bridge` Thing has no Channels.

## Prerequisites

- The Binding talks only to your local network and needs no Tibber account.
- The bridge must be reachable from openHAB at its host name or IP address.
- The password is the code on the QR label of the bridge.
  The Binding sends it with HTTP Basic authentication and the fixed user name `admin`.
- The web interface of the bridge must stay enabled permanently, see the next section.
- The Pulse must be set to the SML transport mode.
  The plaintext, IEC 62056-21 and impulse modes are not supported.

## Keeping the Bridge's Web Interface Enabled

The bridge serves the stream of readings from its web server, and the Binding opens the stream with the web password.
A bridge whose web interface is switched off cannot be used, so the web server has to stay running.
The steps below follow the guide of the Home Assistant project `marq24/ha-tibber-pulse-local`, and they can differ by hardware revision and firmware.

1. Put the bridge into access point mode by unplugging it and plugging it in again, as described in the guide.
   The LED turns green.
1. Connect a phone or laptop to the Wi-Fi network `Tibber Bridge`.
   The Wi-Fi password is the code on the label of the bridge.
1. Open the web interface of the bridge in a browser and log in with the same code.
1. Set the parameter `webserver_force_enable` to `true`, using the console or the parameters page depending on your firmware, and store the parameters.
1. Restart the bridge so that it returns to your home network.

See the [preparation guide](https://github.com/marq24/ha-tibber-pulse-local/blob/main/preparation.md) for details, screenshots and troubleshooting.

## Discovery

### Bridge Discovery

The Binding finds a bridge by itself while the web server of the bridge runs.
The bridge announces that web server on the local network by multicast DNS (mDNS), as the service `Tibber Bridge Web Server` of type `_http._tcp` on port 80, and the Binding listens for this announcement.
Only the instance name `Tibber Bridge Web Server` is recognized, so other devices with a web server do not appear in the Inbox.
A bridge whose web interface is switched off does not announce its web server and does not appear, see [Keeping the Bridge's Web Interface Enabled](#keeping-the-bridges-web-interface-enabled).

The Inbox entry is named "Tibber Pulse Bridge (`bridge01234567`)", after the host label of the bridge.
The host label is the host name of the bridge without `.local`: `Bridge` followed by 8 hexadecimal digits, for example `Bridge01234567`.
The Thing id of the entry is the host label in lower case, so the Thing UID is `tibberpulselocal:bridge:bridge01234567`.
The bridge is offered once, however often it is announced.
A bridge for which a `bridge` Thing with that Thing UID exists, in any status, is not offered again.
After you removed that Thing, the bridge appears again after a manual scan, after a restart of openHAB, or when its announcement changes.
openHAB does not report an announcement that the bridge merely repeats unchanged.

When you approve the entry:

- The entry offers the current address of the bridge as `hostname`.
  It carries no password.
  Enter the password, the code on the label of the bridge, when you approve the entry.
  The password is a required parameter of the `bridge` Thing.
  If an entry is approved without it, for example through the REST API, openHAB creates the Thing but does not initialize it.
  The Thing stays UNINITIALIZED with the detail HANDLER_CONFIGURATION_PENDING, and its status description names the missing parameter `password`.
  When you set the password in the configuration of the Thing, it initializes and goes ONLINE as soon as the bridge delivers data.
- Discovery never opens a connection to the bridge.
  It can neither check the password nor whether the address is reachable.
  An entry only means that the bridge announces its web server.
- The entry carries the IPv4 address of the bridge, because a host without an mDNS-aware resolver may not be able to resolve `.local` names.
  Only when the bridge is announced without an IPv4 address does the entry carry the `.local` host name, for example `Bridge01234567.local`.
  That name may not resolve on your system: replace it by the IP address of the bridge in the configuration of the Thing.
  An IPv6 address is never offered.
- The Thing gets the property `hostLabel`, see [Thing Properties](#thing-properties).

Approve the Inbox entry instead of creating the `bridge` Thing by hand.
Discovery recognizes a configured bridge by its Thing UID only and never compares `hostname` values.
A `bridge` Thing that you created by hand, in the user interface or in a `.things` file, usually has another Thing id than the entry and is then not recognized: the same bridge can appear in the Inbox again.
The same holds when you approve the entry under another Thing id.
Ignore such an entry first and delete it only if you must: a deleted entry can return after a manual scan, after a restart of openHAB, or when the announcement changes, while an ignored entry stays ignored as long as the bridge is announced.
Do not approve it, because two `bridge` Things for the same bridge take the stream from each other, see [Limits and Notes](#limits-and-notes).
If you prefer a `.things` file, use the Thing UID of the entry, for example `tibberpulselocal:bridge:bridge01234567`, so that the bridge is not offered again.
The `bridge` Thing type declares `hostLabel` as its representation property.
A Thing that was created from an Inbox entry carries the property `hostLabel`, so openHAB sets the Inbox entry of the same bridge to ignored when that Thing is ONLINE, also when you approved the entry under another Thing id.
The entry then stays in the Inbox, marked as ignored, and it is removed again when you remove that Thing.
A `bridge` Thing that you created by hand has no property `hostLabel` and does not cause this.
You can switch this behavior off with the system setting "Auto Ignore" of the Inbox.

The descriptions in this section of how openHAB recognizes existing Things, updates addresses, removes Inbox entries and runs scans were checked against the source code of openHAB 5.2.3 and of the 5.3 development version, where they are the same.
Only the following behaviors of openHAB were also observed on a running openHAB 5.2.3 with one real bridge:

- With a `bridge` Thing that was created by hand under another Thing UID already present, the Inbox offered exactly one entry for the bridge, with the label, the Thing UID, the properties `hostLabel` and `hostname`, and the IPv4 address described above.
- After approving that entry, the Thing carried the property `hostLabel` and had the `hostname` in its configuration.
- A Thing that was approved without a password stayed UNINITIALIZED until the password was set, as described above, and was ONLINE within about 5 seconds after that.
- Once the `bridge` Thing was ONLINE, the Inbox offered the `meter` Thing of the Pulse within about 5 seconds.
  The approved `meter` Thing was ONLINE after about 10 seconds, with all 14 Channels and the properties `deviceId` and `meterId`.

The entries were approved through the REST API, not in the user interface.
Everything else in this section comes from the source code and has not been observed on a running system, in particular a manual scan, ignoring an entry, a bridge that withdraws its announcement or is unplugged, a changed address, a disabled or offline Thing, the `.local` host name, approving an entry under another Thing id, and the statement that discovery opens no connection.

The address of the bridge can change, for example when your router hands out another address:

- A Thing that you created from an Inbox entry follows the address that the bridge announces: openHAB updates its `hostname` when the bridge gets another address.
  The `password` stays unchanged.
- An announcement without an IPv4 address can set `hostname` of such a Thing to the `.local` name until the next announcement with an address.
- A Thing defined in a `.things` file is never changed by discovery.
  The same holds for a Thing that you created by hand with another Thing id than the entry.
  A Thing that you created in the user interface with the Thing id of the entry is rewritten like one created from the entry.

The update is a behavior of openHAB, not of this Binding, and it happens only while openHAB runs and the bridge is announced.
Give the bridge a fixed address, for example a DHCP reservation in your router, so that you do not depend on it.

A Thing created from an Inbox entry also replaces a `hostname` that you typed in yourself with the announced address.
If you need a host name that never changes, define the `bridge` Thing in a `.things` file or give it another Thing id than the one of the Inbox entry.

mDNS announcements are not authenticated.
A device on your network could announce the same host name with its own address, and a Thing created from an Inbox entry would then connect to it and send the bridge password in clear text.
On a network that you do not trust, define the `bridge` Thing in a `.things` file with a fixed address.

A manual scan in the Inbox lists the bridges that are announced at that time.
It also removes the entries of bridges that are not announced, including entries that you ignored.
The entry of an announced bridge stays, also when ignored.
A scan ends 5 seconds after it started, however many other bindings take part in the shared network scan; on a system with many such bindings the entry of an announced bridge may be removed and appear again as a new entry.
When a bridge withdraws its announcement or leaves the network, openHAB removes its entry from the Inbox as well, also an ignored one, which then returns as a new entry.
openHAB learns about this up to about 11 seconds after the bridge says goodbye, and only when the announcement expires if the bridge disappears without a goodbye, for example after a power loss.
Sometimes openHAB cannot tell which bridge has gone, and the entry then stays in the Inbox until the next manual scan.
A scan that finds no bridge is not an error.

The announcement was observed on one bridge and one firmware only, the one named in [Limits and Notes](#limits-and-notes).
Other firmware or hardware, such as a Pulse P1 running bridge firmware, may announce another name and is not discovered.
Create such a bridge manually.
Discovery is optional: you can always create the `bridge` Thing manually with its host name and password, for example when multicast DNS does not reach openHAB.

### Meter Discovery

Meters are discovered from the stream of an online `bridge` Thing, whether it was created from an Inbox entry or manually.
Every Pulse device that delivers a valid reading, and for which no `meter` Thing exists, appears in the Inbox as "Tibber Pulse `<deviceId>`" below that bridge.
Accepting the entry creates the `meter` Thing and fills in its `deviceId`.
A manual scan offers the devices seen so far again, for example after you removed a `meter` Thing.
Discovery never opens a connection of its own, because the bridge serves only one stream client.

## Thing Configuration

### `bridge` Thing Configuration

| Name       | Type | Description                                                                  | Default | Required | Advanced |
|------------|------|------------------------------------------------------------------------------|---------|----------|----------|
| `hostname` | text | Host name or IP address of the bridge, without scheme, port, or path.        | N/A     | yes      | no       |
| `password` | text | Password of the bridge web interface, printed as a code on the bridge label. | N/A     | yes      | no       |

The Binding always connects to port 80 of the bridge.
The host name may be a DNS name, an IPv4 address or an IPv6 address without brackets.
Enter it without scheme and without path: a value such as `http://192.0.2.20/` is a configuration error.
Host names with non-ASCII characters (IDN) must be entered in punycode.
The Binding checks only that the value is not empty, not longer than 253 characters and free of spaces, control characters and `/`.
Whether the name or address exists is decided when the Binding connects.
A port in the value, such as `192.0.2.20:80`, is not a configuration error, but the bridge is then not reachable, because the name cannot be resolved.

Do not configure two `bridge` Things for the same bridge.
The bridge serves only one stream client, so the two Things would take the stream from each other.
The Binding does not check for this, also not when two different names point to the same bridge.

Changing `hostname` or `password` closes the stream and opens a new one.

### `meter` Thing Configuration

| Name           | Type    | Description                                                                                                                     | Default | Required | Advanced |
|----------------|---------|---------------------------------------------------------------------------------------------------------------------------------|---------|----------|----------|
| `deviceId`     | text    | Device id of the Pulse, 16 hexadecimal digits. Meters found on the stream of the bridge are offered in the inbox with their id. | N/A     | yes      | no       |
| `staleTimeout` | integer | Seconds without a valid reading after which the meter goes offline and its live values become undefined.                        | 60      | no       | yes      |

The `deviceId` is not case-sensitive.
The `staleTimeout` is given in seconds and must be at least 10.

Changing `deviceId` or `staleTimeout` only initializes that `meter` Thing again.
The stream stays open.

## Channels

All Channels of the `meter` Thing are read-only.
The per-phase Channels and `frequency` are advanced Channels.

| Channel ID      | Type                     | Unit | Description                                                                        | Advanced |
|-----------------|--------------------------|------|------------------------------------------------------------------------------------|----------|
| `power`         | Number:Power             | W    | Net active power, positive when drawn from the grid and negative when fed into it. | no       |
| `energy-import` | Number:Energy            | kWh  | Total energy drawn from the grid. Keeps its last value when the meter goes stale.  | no       |
| `energy-export` | Number:Energy            | kWh  | Total energy fed into the grid. Keeps its last value when the meter goes stale.    | no       |
| `power-l1`      | Number:Power             | W    | Active power on phase L1.                                                          | yes      |
| `power-l2`      | Number:Power             | W    | Active power on phase L2.                                                          | yes      |
| `power-l3`      | Number:Power             | W    | Active power on phase L3.                                                          | yes      |
| `voltage-l1`    | Number:ElectricPotential | V    | Voltage on phase L1.                                                               | yes      |
| `voltage-l2`    | Number:ElectricPotential | V    | Voltage on phase L2.                                                               | yes      |
| `voltage-l3`    | Number:ElectricPotential | V    | Voltage on phase L3.                                                               | yes      |
| `current-l1`    | Number:ElectricCurrent   | A    | Current on phase L1.                                                               | yes      |
| `current-l2`    | Number:ElectricCurrent   | A    | Current on phase L2.                                                               | yes      |
| `current-l3`    | Number:ElectricCurrent   | A    | Current on phase L3.                                                               | yes      |
| `frequency`     | Number:Frequency         | Hz   | Grid frequency.                                                                    | yes      |
| `last-update`   | DateTime                 | -    | Time of the latest valid reading received for this meter.                          | no       |

### Channel Behavior

- A quantity that the meter never reports stays UNDEF and is never published as 0.
  A value that the meter reports as 0 is published as 0.
- `power` is the net power: positive when drawn from the grid, negative when fed into it.
  If the meter reports import and export power separately instead of a net value, the Binding publishes the difference.
  `power` is UNDEF for meters that report no power, and for meters that report only import power.
  The Binding never estimates values.
- Every valid reading updates the Channels whose quantity it contains, also when the value did not change.
  Rules triggered by an update and persistence strategies "on every update" therefore work as expected.
  The update rate is that of the bridge, about one reading every 3 seconds.
- About one reading in ten is truncated by the bridge.
  The Binding drops such a reading silently, without a log entry above debug level, and the next reading follows a few seconds later.
- A live value Channel that has not been updated for `staleTimeout` seconds becomes UNDEF, even while other Channels are still updated.
- When the connection to the bridge is lost, or the `meter` Thing goes stale, `energy-import`, `energy-export` and `last-update` keep their last value, and all other value Channels become UNDEF.
  When the `meter` Thing is initialized again, for example after a change of its configuration, only the live value Channels are set to UNDEF until the next reading arrives.
  The two energy counters and `last-update` are cleared only when the `deviceId` changes.
- A `REFRESH` command re-publishes the cached value of the Channel and does not contact the bridge.
  All other commands are ignored.
- Phase angles, reactive power and energy, tariff registers and meter firmware strings are not published.

### Thing Properties

A `bridge` Thing that was created from an Inbox entry has the property `hostLabel`, the host label of the bridge in lower case, for example `bridge01234567`.

A `meter` Thing has the properties `deviceId` and `meterId`.
The `meterId` is the identifier of the physical meter as reported in its readings.
It identifies your meter, so keep it out of screenshots and bug reports.
The Binding never writes it to the log, and no property contains the password.

## Thing Statuses

### `bridge` Thing Statuses

| Status  | Detail              | Cause                                                                                       |
|---------|---------------------|---------------------------------------------------------------------------------------------|
| UNKNOWN | NONE                | Initialized, no data received from the bridge yet                                           |
| ONLINE  | NONE                | The first data was received from the bridge                                                 |
| OFFLINE | CONFIGURATION_ERROR | The host name is invalid (empty, too long, or contains a space, a control character or `/`) |
| OFFLINE | CONFIGURATION_ERROR | The password is present but empty                                                           |
| OFFLINE | CONFIGURATION_ERROR | The bridge rejected the password (HTTP 401 or 403)                                          |
| OFFLINE | COMMUNICATION_ERROR | The firmware does not offer the stream (HTTP 404)                                           |
| OFFLINE | COMMUNICATION_ERROR | The bridge answered with another HTTP status                                                |
| OFFLINE | COMMUNICATION_ERROR | The bridge cannot be reached, or the host name does not resolve                             |
| OFFLINE | COMMUNICATION_ERROR | The bridge closed the stream; another client may have taken it                              |
| OFFLINE | COMMUNICATION_ERROR | No data was received for 30 seconds                                                         |
| OFFLINE | COMMUNICATION_ERROR | The data from the bridge is invalid                                                         |

A `bridge` Thing without the required `password`, for example one that was created by approving an Inbox entry through the REST API, does not appear in this table.
openHAB keeps it UNINITIALIZED with the detail HANDLER_CONFIGURATION_PENDING until the password is set, and the Binding does not run for it.
The CONFIGURATION_ERROR for an empty password applies only to a password that is present but empty.

After a communication error, the Binding reconnects by itself, first after 2 seconds, then with a delay that doubles up to 60 seconds.
If the firmware does not offer the stream, it tries again every 60 seconds.
After a rejected password it does not try again until you change the configuration of the `bridge` Thing or initialize it again.

### `meter` Thing Statuses

| Status  | Detail              | Cause                                       |
|---------|---------------------|---------------------------------------------|
| UNKNOWN | NONE                | No valid reading has arrived yet            |
| ONLINE  | NONE                | A valid reading arrived                     |
| OFFLINE | COMMUNICATION_ERROR | No valid reading for `staleTimeout` seconds |
| OFFLINE | BRIDGE_OFFLINE      | The bridge is not ONLINE                    |
| OFFLINE | CONFIGURATION_ERROR | Invalid `deviceId` or `staleTimeout`        |
| OFFLINE | CONFIGURATION_ERROR | The `meter` Thing has no bridge             |

The `meter` Thing becomes ONLINE only after its first valid reading, not when the bridge goes ONLINE.
Readings of other meters, dropped readings and readings without any value do not count as valid readings.
The next valid reading after a stale period sets the `meter` Thing ONLINE again.

## Limits and Notes

- The bridge serves exactly one stream client, as observed on the tested firmware.
  A second client, such as the live page of the bridge, a browser tab or a Home Assistant integration, takes the stream away immediately.
  The Binding then shows the `bridge` Thing as OFFLINE and reconnects, which may in turn take the stream from the other client.
  Run only one client at a time, which includes two `bridge` Things for the same bridge: they would take the stream from each other.
- The bridge offers no encryption.
  The Binding talks unencrypted HTTP and websocket-style traffic on port 80, with Basic authentication, so the password can be read by anyone who can listen on your local network.
- The Binding relies on a quirk of the tested firmware: the bridge starts the stream after a plain `GET /ws` request, without the standard websocket handshake.
  Firmware that behaves differently may not work.
- The Binding was tested with a Tibber Pulse IR (node model TFD01) on a bridge with firmware ESP 1811-97099eed / EFR 835-942ac28e, and with an SML meter that reports power, energy, per-phase power, voltage and current, and frequency.
  Other firmware versions and other meters may behave differently.
- Several Pulses on one bridge are told apart by their `deviceId`.
  This was not tested with real hardware.
- Meters read over IEC 62056-21 may report no power.
  Their power Channels stay UNDEF.
- The Binding only reads from the bridge and never changes its settings.
- Not supported in this version: discovery of bridges that do not announce the mDNS service of the tested firmware (for example the service type `_tibber-pulse._tcp` or the instance name `Pulse P1 Web Server`, which are reported for other firmware or hardware), HTTP polling instead of the stream, the API token of newer firmware, and decoded readings delivered by newer firmware.

## Full Example

### `tibberpulselocal.things` Example

```java
Bridge tibberpulselocal:bridge:home "Tibber Pulse Bridge" [ hostname="192.0.2.20", password="<QR code>" ] {
    Thing meter main "Electricity Meter" [ deviceId="0123456789abcdef", staleTimeout=60 ]
}
```

Replace the host name and the password with the values of your bridge.
The password is the code on its label.
The Thing id `home` differs from the id of an Inbox entry, so the Inbox may offer the bridge again: ignore that entry, or use the id of the entry instead, for example `bridge01234567`.

### `tibberpulselocal.items` Example

```java
Number:Power             Grid_Power         "Grid power [%.0f %unit%]"      {channel="tibberpulselocal:meter:home:main:power"}
Number:Energy            Grid_Energy_Import "Energy imported [%.3f %unit%]" {channel="tibberpulselocal:meter:home:main:energy-import"}
Number:Energy            Grid_Energy_Export "Energy exported [%.3f %unit%]" {channel="tibberpulselocal:meter:home:main:energy-export"}
Number:Power             Grid_Power_L1      "Power L1 [%.0f %unit%]"        {channel="tibberpulselocal:meter:home:main:power-l1"}
Number:ElectricPotential Grid_Voltage_L1    "Voltage L1 [%.1f %unit%]"      {channel="tibberpulselocal:meter:home:main:voltage-l1"}
Number:ElectricCurrent   Grid_Current_L1    "Current L1 [%.2f %unit%]"      {channel="tibberpulselocal:meter:home:main:current-l1"}
Number:Frequency         Grid_Frequency     "Frequency [%.1f %unit%]"       {channel="tibberpulselocal:meter:home:main:frequency"}
DateTime                 Grid_Last_Update   "Last update [%1$tF %1$tT]"     {channel="tibberpulselocal:meter:home:main:last-update"}
```

## Acknowledgements

This Binding is an independent implementation.
It is not affiliated with or supported by Tibber, and it contains no code from the projects below or from the SmlLib library that they use.
It builds on the experience that these projects documented:

- [`marq24/ha-tibber-pulse-local`](https://github.com/marq24/ha-tibber-pulse-local) (Apache-2.0): the raw websocket stream of the bridge, the reading of its frames, the filtering by device id, the mapping of the OBIS codes and the tolerance for truncated frames.
- [`jimmyw/ha-tibber-pulse-local`](https://github.com/jimmyw/ha-tibber-pulse-local) (MIT): the approach to reconnecting with an increasing delay, the observation that a closed stream can mean that another client took it, and the documentation of the mDNS service names of the bridge (the `Tibber Bridge Web Server` instance on `_http._tcp`), which were then confirmed on a real bridge.
- [`ProfDrYoMan/tibber_pulse_local`](https://github.com/ProfDrYoMan/tibber_pulse_local) (MIT): the collection of knowledge about the bridge.
