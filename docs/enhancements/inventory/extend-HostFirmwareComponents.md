# Align HostFirmwareComponents with SimpleUpdate API

## Problem Statement

NIC firmware updates do not work as expected because the
[HostFirmwareComponents (HFC)
CRD](https://github.com/metal3-io/baremetal-operator/blob/main/apis/metal3.io/v1alpha1/hostfirmwarecomponents_types.go)
from metal3.io implies that updates target specific NICs using a component
identifier (adapter ID or serial number), but this does not reflect the actual
behavior. The underlying [Redfish SimpleUpdate
API](https://www.dmtf.org/sites/default/files/standards/documents/DSP2062_1.0.2.pdf)
only accepts the firmware image URL and applies the update to all matching
devices, ignoring the component identifier.

## Constraints

- We can only use SimpleUpdate API, the URL to firmware file is only information passed to the BMC.
- SimpleUpdate/Targets parameter exists, but is not implemented by Ironic and not be fully
  supported by vendors. That means, all NICs of the same firmware a vendor will
  be updated at once.
- We cannot extract metadata from the firmware file in a standard/vendor
  agnostic way, and we cannot validate firmware file contents. Only the BMC can
  validate and determine applicability. Firmware version information is embedded
  in the file and only known by the user providing the URL. The system
  cannot determine if an update is needed without attempting it.
- There is a unique firmware file per vendor and model.

## Background Info: Ironic and Simple Update

The
[Redfish SimpleUpdate action](https://redfish.dmtf.org/schemas/v1/UpdateService.v1_17_0.json)
updates software components via a software image file at a URI. The action is
triggered via POST to `/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate`
with **Parameters:**
- `ImageURI` (required): The URI of the software image to install
- `Targets` (optional): Array of URIs indicating where to apply the update

From the [UpdateService schema](https://redfish.dmtf.org/schemas/v1/UpdateService.v1_17_0.json):
"If this parameter is not present or contains no targets, the service shall
apply the software image to all applicable targets."

Ironic implementation does not use the `Targets` parameter. From the
[Ironic firmware updates guide](https://docs.openstack.org/ironic/latest/admin/firmware-updates.html):
"At the present time, targets for the firmware update cannot be specified. In
testing, the BMC applied the update to all applicable targets on the node."

## Use Case

Update all **Intel NIC adapters** of node server-01 to firmware version **22.5.7**

## Proposed HFC.spec

Align the user-facing CR with the actual API behavior.

Extend HostFirmwareComponents to use just `component: nic` which indicates that
we do network interfaces (not bios) and provide a list of URLs of all the
firmware we need.


```yaml
apiVersion: metal3.io/v1alpha1
kind: HostFirmwareComponents
metadata:
  name: server-01
spec:
  updates:
    - component: nic
      url:
      - http://firmware-repo/E810_NVMUpdatePackage_v4_30.bin
      - http://firmware-repo/fw-ConnectX4Lx-rel-14_32_1010.bin
      - http://firmware-repo/fw-ConnectX6Dx-rel-22_38_1002.bin
```

- There is no validation anywhere, URLs is given to BMC
- BMC will process the file, and will apply if possible e.g. if valid firmware for Intel E810, all matching NICs will be upgraded.
- The URL / firmware file name is given by the user

The HFC.spec will be populated by the `HardwareProfile.clcm.openshift.io` CR processed by `oran-o2ims` operator.

### Update BMO to support URL list

```
PR: Change HostFirmwareComponents API to support flat NIC firmware list

PR Description

Change the HostFirmwareComponents CRD API to accept a flat list of NIC
firmware URLs instead of requiring specific adapter targeting.
```

## Proposed HFC.Status

```yaml
status:
  components:
  - component: nic      // generic type nic
    id: xxxxx   // unique stable identifier like serial
    model: 0x8086 0x1593 // vendor+product
    currentVersion: 23.0.8
    initialVersion: 23.0.8
```

### Ironic Implementation

Repository: https://opendev.org/openstack/ironic

```
Summary: [RFE] Add hardware model and serial number to firmware API for network adapters

Description:
The firmware API currently exposes firmware versions for network adapters but
lacks hardware identification data (PCI vendor/product IDs, model names, and
serial numbers). This makes it difficult to uniquely identify physical hardware,
correlate firmware components with inventory data.

Proposed API Change:

API Endpoint: GET /v1/nodes/{uuid}/firmware

Add model and serial_number fields to firmware component data for NICs:

Expected Response Format:
{
"firmware": [
  {
    "component": "nic:NIC.Integrated.1",
    "initial_version": "20.0.17",
    "current_version": "20.0.17",
    "last_version_flashed": "20.0.17",
    "model": "0x8086 0x1593",          // NEW FIELD: PCI vendor/product IDs
    "serial_number": "C8:1F:66:C7:A2:3C",  // NEW FIELD: Unique hardware ID
    "created_at": "2025-06-26T01:33:13+00:00",
    "updated_at": "2025-07-02T13:25:38+00:00"
  }
]
}

Field Formats:
- model (PCI IDs preferred): "0xVENDOR 0xPRODUCT" (e.g., "0x8086 0x1593")
- model (fallback): "Manufacturer Model" if PCI IDs unavailable
- serial_number: Vendor-specific format (often MAC address of first port)
```

### Update gophercloud with new fields

Repository: https://github.com/gophercloud/gophercloud

BMO doesn't call Ironic API directly - it uses gophercloud as the client library.
```
PR Title: Add Model and SerialNumber fields to FirmwareComponent
```

### Update BMO to consume enriched response

Repository: https://github.com/metal3-io/baremetal-operator

```
BMO PR Title

Add Model and ID fields to HostFirmwareComponents status

Description:
Extend HostFirmwareComponents status to include hardware identification data
for firmware components.

Changes:
- Add ID field (serial number) to FirmwareComponentStatus
- Add Model field (PCI vendor+product IDs) to FirmwareComponentStatus
- Change Component field from "nic:XXX" to generic type "nic"
- Update GetFirmwareComponents to map new gophercloud fields
- Requires gophercloud PR: "Add Model and SerialNumber fields to FirmwareComponent"

Depends-On: gophercloud/gophercloud#XXXX
```


## NIC Information Available in BareMetalHost and HardwareData

Both CRs share the same `*HardwareDetails` struct containing NIC information:

```go
// https://github.com/metal3-io/baremetal-operator/blob/a4765e4d/apis/metal3.io/v1alpha1/baremetalhost_types.go#L687
// BareMetalHostStatus contains hardware details
type BareMetalHostStatus struct {
  HardwareDetails *HardwareDetails `json:"hardware,omitempty"`
}

// https://github.com/metal3-io/baremetal-operator/blob/a4765e4d/apis/metal3.io/v1alpha1/hardwaredata_types.go#L226
// HardwareDataSpec contains hardware details
type HardwareDataSpec struct {
  HardwareDetails *HardwareDetails `json:"hardware,omitempty"`
}

// HardwareDetails contains NIC array
type HardwareDetails struct {
  NIC []NIC `json:"nics,omitempty"`
  // ... other hardware fields
}

// https://github.com/metal3-io/baremetal-operator/blob/a4765e4d/apis/metal3.io/v1alpha1/hardwaredata_types.go#L144-L177
// NIC struct with vendor information
type NIC struct {
  // ... other NIC fields
  Model     string `json:"model,omitempty"`  // Format: "0x8086 0x1593" (vendor+product IDs)
  // ... other NIC fields
}
```

During hardware inspection, the controller retrieves inventory data from Ironic
and converts it using `hardwaredetails.GetHardwareDetails()`. The controller
first updates `BareMetalHost.Status.HardwareDetails`, then creates/updates the
HardwareData CR with identical data.

Relevant Information: 
```
  Model     string `json:"model,omitempty"`  // Format: "0x8086 0x1593" (vendor+product IDs)
```

### Sushy-Tools for Libvirt

Repository: https://opendev.org/openstack/sushy-tools [21]


## References

- Metal3 baremetal-operator: https://github.com/metal3-io/baremetal-operator
- Ironic: https://opendev.org/openstack/ironic
- gophercloud: https://github.com/gophercloud/gophercloud
