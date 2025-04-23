# OCSF Common Device Identifier
Anthony Chan,
April 2025

When reporting a cybersecurity event it is important to uniquely identify the endpoints. This identification is more valuable if can enable joining data for the same endpoint that were produced by different tools, including those developed by different vendors. The OCSF [Device object](https://schema.ocsf.io/1.4.0/objects/device) includes a Unique ID field (`uid`) that can aid in this goal. However, that field lacks specific guidance so its usage is not standardized. This document proposes a new OCSF Common Device Identifier field and the associated method to generate a deterministic [Universally Unique Identifier](https://www.rfc-editor.org/rfc/rfc9562.html) (UUID) for Linux, macOS and Windows-based computers. It draws inspiration from [Community ID Flow Hashing](https://github.com/corelight/community-id-spec) and [Common Process ID](https://github.com/ocsf/common-process-id), both already used in OCSF (`community_uid` and `cpid` respectively), but applies it to device identification.

The Common Device Identifier has two main benefits:
1. It simplifies joins by encapsulating the endpoint's operating system and two device identifiers in one value. This hides complexity as different operating systems have incompatible device identification schemes, and adds some diversity that can help disambiguate virtual machines.
2. It promotes privacy by hashing unique device identifiers such that their original values do not need to be exposed. Original unique device identifiers are considered confidential and their use should be avoided in untrusted environments. 

Universal uniqueness cannot be guaranteed due to factors such as inconsistent hardware serialization practices, configurable hardware identifiers in virtual machines, and virtual machine image copying. The design aims to make collision within an organization rare enough to reap the practical benefits. In situations where collision does occur, additional device fields such as those found in `hw_info` and `network_interfaces` can be used to disambiguate.

## Format

An OCSF Common Device Identifier is a [UUID Version 5](https://www.rfc-editor.org/rfc/rfc9562.html#name-uuid-version-5) value computed using:
- A [Namespace ID](#namespace-id) corresponding to the device's running operating system, and
- A [Name Value](#name-value) obtained by concatenating with comma separation:
  1. A version number, 
  2. A primary device identifer, and
  3. A secondary device identifier.

> :information_source: 
> Since UUIDv5 requires a SHA1 hash and SHA1 has been deprecated in many applications, the design will be updated to use a [named-based UUIDv8 value with SHA-2 hash](https://github.com/ietf-wg-uuidrev/rfc4122bis/blob/83d95da3418a92a13dfa5222c30795244f24f565/draft-ietf-uuidrev-rfc4122bis.md#example-of-a-uuidv8-value-name-based-uuidv8_example_name). The general design still applies.

### Namespace ID

The Namespace ID is a UUIDv5 value computed using the [Nil UUID](https://www.rfc-editor.org/rfc/rfc9562.html#name-nil-uuid) and the namespace name. The list of valid Namespace IDs are as follows:

<table border="1">
<th>Operating System<th>Namespace Name<th>Namespace ID
<tr>
  <td>Linux<td>ocsf-common-device-id-linux<td>85ffa100-3861-5855-b4a8-ab55931834e3
<tr>
  <td>macOS<td>ocsf-common-device-id-macos<td>f2af7a9b-b3a6-51f2-b7af-fbcc57489bcd
<tr>
  <td>Windows<td>ocsf-common-device-id-windows<td>c81710e6-f7a3-53dc-b104-e6d3606b4015
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
- Namespace ID: `85ffa100-3861-5855-b4a8-ab55931834e3`
- Name Value: `ver=1,id1=7444498d-d6e2-4aff-95a7-a916eb297038,id2=f7743bfa-366f-496a-bf32-3ab50c2313d9`

Result (UUIDv5): `3de24de4-162b-5fa2-8c00-8155220a3485`

#### macOS Example

Given:

```
$ ioreg -d2 -c IOPlatformExpertDevice | awk -F\" '/IOPlatformUUID/{print $(NF-1)}'
89162807-1B01-5560-98C8-8D380844AB81
$ ioreg -d2 -c IOPlatformExpertDevice | awk -F\" '/IOPlatformSerialNumber/{print $(NF-1)}'
X02YZ1ZYZX
```

Note hexadecimal characters in the `IOPlatformUUID` needs to be changed to lowercase, but the letter case for `IOPlatformSerialNumber` is preserved.

Then:
- Namespace ID: `f2af7a9b-b3a6-51f2-b7af-fbcc57489bcd`
- Name Value: `ver=1,id1=89162807-1b01-5560-98c8-8d380844ab81,id2=X02YZ1ZYZX`

Result (UUIDv5): `a3df0964-83b6-5dbe-838e-516f9804a8d6`

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
- Namespace ID: `c81710e6-f7a3-53dc-b104-e6d3606b4015`
- Name Value: `ver=1,id1=48ec0054-b51c-40ad-a03f-dcaa84bb95b9,id2=82be363f-2c7d-4da9-b0a2-d4d80d4156c7`

 Result (UUIDv5): `d9db161e-cf9b-5832-a30a-dfa5d06f0deb`