# WAITPKG CPU Feature Fix Documentation

## Issue Description

The WAITPKG CPU feature was not being properly handled during live migration of virtual machines in libvirt. This caused compatibility issues when migrating VMs between hosts with different CPU capabilities, specifically when the WAITPKG feature was involved.

## Technical Background

The WAITPKG feature (Wait Package) is an x86 CPU feature that provides instructions for power-efficient waiting and wake-up operations. It is identified by:

- CPUID Leaf: 0x00000007
- Sub-leaf: 0
- Register: ECX
- Bit: 5 (WAITPKG)

## Problem Details

The original implementation did not properly handle the WAITPKG feature during:
1. CPU feature detection
2. Live migration compatibility checks
3. CPU model comparison

This could lead to:
- Failed migrations between compatible hosts
- Incorrect CPU feature reporting
- Potential stability issues in VMs using the WAITPKG feature

## Solution

The fix implements proper handling of the WAITPKG feature by:

1. Adding explicit CPUID bit detection:
```c
if (STREQ(name, "waitpkg")) {
    /* The waitpkg feature requires CPUID.07H:ECX.waitpkg [bit 5] */
    VIR_DEBUG("Adding CPU dependency for waitpkg feature");
    
    /* Define the CPU feature bit needed for waitpkg */
    cpuid_waitpkg_item.type = VIR_CPU_X86_DATA_CPUID;
    cpuid_waitpkg_item.data.cpuid.eax_in = 0x00000007;
    cpuid_waitpkg_item.data.cpuid.ecx_in = 0x00000000;
    cpuid_waitpkg_item.data.cpuid.ecx = 0x00000020;  /* Bit 5 in ECX */
    virCPUx86DataAdd(cpuData, &cpuid_waitpkg_item);
}
```

2. Ensuring proper feature handling during:
   - CPU model initialization
   - Live migration checks
   - Feature compatibility verification

## Usage

No changes to VM configurations are required. The fix ensures proper handling of the WAITPKG feature automatically.

### Example VM CPU Configuration

```xml
<cpu mode='custom' match='exact'>
  <model fallback='forbid'>Skylake-Server</model>
  <feature policy='require' name='waitpkg'/>
</cpu>
```

## Testing

The fix has been tested with:
1. Live migration between hosts with and without WAITPKG support
2. Various CPU models that support the WAITPKG feature
3. Different libvirt versions and configurations

## Compatibility

This fix maintains backward compatibility with:
- Existing VM configurations
- Previous libvirt versions
- All supported CPU models

## References

- [Intel® Architecture Instruction Set Extensions Programming Reference](https://software.intel.com/content/www/us/en/develop/download/intel-architecture-instruction-set-extensions-programming-reference.html)
- [QEMU CPU Model Documentation](https://qemu.readthedocs.io/en/latest/system/cpu-models-x86.html)
- [libvirt CPU Model Configuration](https://libvirt.org/formatdomain.html#cpu-model-and-topology)