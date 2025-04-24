# OCSF Common Device Identifier
Anthony Chan,
April 2025

When reporting a cybersecurity event it is important to uniquely identify the endpoints. This identification is more valuable if can enable joining data for the same endpoint that were produced by different tools, including those developed by different vendors. The OCSF [Device object](https://schema.ocsf.io/1.4.0/objects/device) includes a Unique ID field (`uid`) that can aid in this goal. However, that field lacks specific guidance so its usage is not standardized. This document proposes a new OCSF Common Device Identifier field and the associated method to generate a deterministic [Universally Unique Identifier](https://www.rfc-editor.org/rfc/rfc9562.html) (UUID) for Linux, macOS and Windows-based computers. It draws inspiration from [Community ID Flow Hashing](https://github.com/corelight/community-id-spec) and [Common Process ID](https://github.com/ocsf/common-process-id), both already used in OCSF (`community_uid` and `cpid` respectively), but applies it to device identification.

The Common Device Identifier has two main benefits:
1. It simplifies joins by encapsulating the endpoint's operating system and two device identifiers in one value. This hides complexity as different operating systems have incompatible device identification schemes, and adds some diversity that can help disambiguate virtual machines.
2. It promotes privacy by hashing unique device identifiers such that their original values do not need to be exposed. Original unique device identifiers are considered confidential and their use should be avoided in untrusted environments. 

Universal uniqueness cannot be guaranteed due to factors such as inconsistent hardware serialization practices, configurable hardware identifiers in virtual machines, and virtual machine image copying. The design aims to make collision within an organization rare enough to reap the practical benefits. In situations where collision does occur, additional device fields such as those found in `hw_info` and `network_interfaces` can be used to disambiguate.

## Format

An OCSF Common Device Identifier is a [UUID Version 8 name-based value](https://www.rfc-editor.org/rfc/rfc9562#name-example-of-a-uuidv8-value-n) computed using:
- A [Namespace ID](#namespace-id) corresponding to the device's running operating system, and
- A [Name Value](#name-value) obtained by concatenating with comma separation:
  - A version number,
  - A primary device identifer, and
  - A secondary device identifier.

### Namespace ID

The Namespace ID is a UUIDv8 named-based value computed using the [Nil UUID](https://www.rfc-editor.org/rfc/rfc9562.html#name-nil-uuid) and the namespace name. The list of valid Namespace IDs are as follows:

<table border="1">
<th>Operating System<th>Namespace Name<th>Namespace ID
<tr>
  <td>Linux<td>ocsf-common-device-id-linux<td>15dafea5-d4f6-870a-8831-a875a3e69086
<tr>
  <td>macOS<td>ocsf-common-device-id-macos<td>00b8bd88-f54b-805f-b834-e01b21f3ad3e
<tr>
  <td>Windows<td>ocsf-common-device-id-windows<td>afa77f2c-696a-8862-8693-dc7446d7ddca
</table>

### Name Value

The Name Value is an ASCII string in the format:

```
ver=[version],id1=[Primary Unique Device ID],id2=[Secondary Unique Device ID]
```

where:
- `[version]` is the Name Value format version. It is currently `1`.  
- `[Primary Unique Device ID]` and `[Secondary Unique Device ID]` are two original unique device identifers applicable for the device's operating system. If the identifier is a UUID, the value is the textual representation in 8-4-4-4-12 format, containing dashes and lowercase hexadecimal characters (e.g., `550e8400-e29b-41d4-a716-446655440000`). 

The unique device identifiers used in each operating system are as follows:  

<table border="1">
<th>Operating System<th>Primary Unique Device ID (<code>id1</code>)<th>Secondary Unique Device ID (<code>id2</code>)
<tr>
  <td>Linux<td rowspan="3">Device Manufacturer Assigned Hardware UUID<td>Linux Machine ID
<tr>
  <td>macOS<td>Mac Serial Number
<tr>
  <td>Windows<td>Windows Machine GUID
</table>

#### Device Manufacturer Assigned Hardware UUID

For Linux and Windows computers, the Device Manufacturer Assigned Hardware UUID is the UUID member of the System Information structure in the [SMBIOS](https://www.dmtf.org/standards/smbios) information. For Mac computers, it is the Apple-assigned [Hardware UUID](https://theapplewiki.com/wiki/UDID#Mac), also known as [IOPlatformUUID](https://github.com/apple-oss-distributions/xnu/blob/8d741a5de7ff4191bf97d57b9f54c2f6d4a15585/iokit/IOKit/IOKitKeys.h#L264-L265) in the I/O Registry. It maps to the  `uuid` field in the OCSF [Device Hardware Info](https://schema.ocsf.io/1.4.0/objects/device_hw_info) object.

The following table lists example terminal commands that can be used to obtain the hardware UUID: 

<table border="1">
<th>Operating System<th>Command for Device Manufacturer Assigned Hardware UUID
<tr>
  <td>Linux<td><code>dmidecode -s system-uuid</code>
<tr>
  <td>macOS<td><code>ioreg -d2 -c IOPlatformExpertDevice | awk -F\" '/IOPlatformUUID/{print $(NF-1)}'</code>
<tr>
  <td>Windows<td><code>Get-WmiObject Win32_ComputerSystemProduct | Select-Object -ExpandProperty UUID</code>
</table>

#### Linux Machine ID

The [Linux Machine ID](https://man7.org/linux/man-pages/man5/machine-id.5.html) is usually generated during system installation or first boot and can be retrieved from the file `/etc/machine-id`. It maps to the `os_machine_uuid` field in the OCSF [Device](https://schema.ocsf.io/1.4.0/objects/device) object when `os.type_id` is `200` (Linux).

#### Mac Serial Number

The [Mac Serial Number](https://support.apple.com/en-us/102767) is printed on every Mac computer, usually on the underside, near the regulatory markings. It can be retrieved using the terminal command:

```
ioreg -d2 -c IOPlatformExpertDevice | awk -F\" '/IOPlatformSerialNumber/{print $(NF-1)}'
```

Since the serial number is not UUID, the true value including the original letter case provided by macOS should be preserved. It maps to the `serial_number` field in the OCSF [Device Hardware Info](https://schema.ocsf.io/1.4.0/objects/device_hw_info) object when the device's `os.type_id` is `300` (macOS). 

#### Windows Machine ID

The Windows Machine GUID is generated by Windows upon installation and the value is stored in the registry at the path: `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Cryptography\MachineGuid`. It can be obtained in a PowerShell terminal using the command:

```
Get-ItemProperty -Path HKLM:\SOFTWARE\Microsoft\Cryptography -Name "MachineGuid"
```

It Windows Machine GUID maps to the `os_machine_uuid` field in the OCSF [Device](https://schema.ocsf.io/1.4.0/objects/device) object when `os.type_id` is `100` (Windows).

### Examples

#### Linux Example

Given:

```
$ sudo dmidecode -s system-uuid
7444498d-d6e2-4aff-95a7-a916eb297038
$ cat /etc/machine-id
f7743bfa366f496abf323ab50c2313d9
```

Note dashes need to be added to the Machine ID string to create a UUID. 

Then:
- Namespace ID: `15dafea5-d4f6-870a-8831-a875a3e69086`
- Name Value: `ver=1,id1=7444498d-d6e2-4aff-95a7-a916eb297038,id2=f7743bfa-366f-496a-bf32-3ab50c2313d9`

Result (UUIDv8): `439f5338-1bad-8f0c-b2f0-ffef481f703f`

#### macOS Example

Given:

```
$ ioreg -d2 -c IOPlatformExpertDevice | awk -F\" '/IOPlatformUUID/{print $(NF-1)}'
89162807-1B01-5560-98C8-8D380844AB81
$ ioreg -d2 -c IOPlatformExpertDevice | awk -F\" '/IOPlatformSerialNumber/{print $(NF-1)}'
X02YZ1ZYZX
```

Note hexadecimal characters in the `IOPlatformUUID` needs to be changed to lowercase, but the letter case for the non-UUID `IOPlatformSerialNumber` is preserved.

Then:
- Namespace ID: `00b8bd88-f54b-805f-b834-e01b21f3ad3e`
- Name Value: `ver=1,id1=89162807-1b01-5560-98c8-8d380844ab81,id2=X02YZ1ZYZX`

Result (UUIDv8): `09446232-f1e6-8069-a4ab-74476c302b2a`

#### Windows Example

Given:

```
> Get-WmiObject Win32_ComputerSystemProduct | Select-Object -ExpandProperty UUID
48EC0054-B51C-40AD-A03F-DCAA84BB95B9
> Get-ItemProperty -Path HKLM:\SOFTWARE\Microsoft\Cryptography -Name "MachineGuid"
82be363f-2c7d-4da9-b0a2-d4d80d4156c7
```

Note hexadecimal characters in the Computer System Product UUID needs to be changed to lowercase.

Then:
- Namespace ID: `afa77f2c-696a-8862-8693-dc7446d7ddca`
- Name Value: `ver=1,id1=48ec0054-b51c-40ad-a03f-dcaa84bb95b9,id2=82be363f-2c7d-4da9-b0a2-d4d80d4156c7`

 Result (UUIDv8): `e085e446-1c51-8fc4-9b95-2499b071639d`