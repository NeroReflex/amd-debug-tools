# Platform support

`amd-s2idle` supports both AMD and Intel platforms.  The s2idle (Low Power
S0 Idle) state is an ACPI feature that is implemented by both vendors, but
the way the platform reports whether the deeper low power substates were
reached differs considerably between them.

The tool detects the CPU vendor by reading `/proc/cpuinfo` and then applies
the checks that are meaningful for that platform.

## Common checks

These checks are applied on all platforms:

* The ACPI FADT advertises the Low Power S0 Idle support
* The kernel is using the `s2idle` suspend method rather than `deep`
* The `LPS0` device interface is not disabled with `sleep_no_lps0`
* The kernel log is available and hasn't wrapped
* The network interfaces have Wake-on-LAN enabled
* Device firmware versions are up to date
* The NVIDIA proprietary driver is configured correctly (if present)

## AMD specific checks

* The `amdgpu` driver is bound to the integrated GPU
* The `amdgpu` `ppfeaturemask` hasn't been overridden
* The `amd_pmc` driver is loaded (it is what puts the SoC into hardware sleep)
* The `pinctrl_amd` GPIO driver is loaded
* The `amd_hsmp` driver isn't conflicting with `amd_pmc`
* The SoC supports hardware sleep over s2idle
* The CPU hasn't been artificially limited with `maxcpus`
* The IOMMU is properly configured on affected platforms
* The storage devices are configured for `DevSlp`/`simple suspend`
* The DMCUB firmware is new enough to power gate the DPIA

## Intel specific checks

* The `intel_pmc_core` driver is loaded

  This driver is what puts the platform into the deeper S0ix substates, so
  without it the system can only reach the shallowest low power state.

* An Intel graphics driver (`i915` or `xe`) is in use

  The integrated GPU coordinates the power states of the display and other
  IP blocks on the platform.

* The Intel display firmware (DMC) is loaded

  The display firmware needs to be present for the platform to reach the
  deeper substates.  This is only checkable with the `i915` driver, as the
  `xe` driver doesn't expose the same interface.

* The `LPS0` device interface is in use

  If the kernel was started with `sleep_no_lps0` the platform will refuse to
  enter the low power S0 idle state.

* `turbostat` is installed

  `turbostat` is used to measure the package C-state residency, which is how
  the time spent in the S0ix state is verified on Intel platforms.

### Intel data capture

On Intel platforms the tool additionally captures the data exposed by the
`intel_pmc_core` driver in `/sys/kernel/debug/pmc_core`:

* `slp_s0_residency_usec` - the S0ix residency counter
* `substate_residencies` - the residency of each S0ix substate
* `substate_requirements` - the IP blocks that are required for each substate
* `substate_status_registers` - the latched status of the IP blocks
* `substate_live_status_registers` - the live status of the IP blocks

The residency counters are cumulative, so the tool reads them before the
system suspends and again after it resumes in order to work out how much
time was spent in the S0ix state during that cycle.  The change in each
substate residency counter is reported as well, which shows which of the
substates was actually entered.

The `substate_requirements` file lists the IP blocks that are required for
each substate.  Any IP block that appears in that file is a potential
blocker for the deeper substates, and the tool reports them.

## NVIDIA proprietary driver support

The NVIDIA proprietary driver is supported on both AMD and Intel platforms.
The driver has to be configured properly for the system to reach the S0ix
state during s2idle, so the following is checked:

* The `NVreg_EnableS0ixPowerManagement` module parameter is set to `1`

  Without this the driver will not put the GPU video memory into self
  refresh mode during s2idle, which means the GPU stays powered on and the
  platform can't reach the deeper substates.  It can be set with:

  ```
  # /etc/modprobe.d/nvidia.conf
  options nvidia NVreg_EnableS0ixPowerManagement=1
  ```

* The GPU supports video memory self refresh

  This is reported by the driver in
  `/proc/driver/nvidia/gpus/<BDF>/power`.  A GPU that doesn't support it
  can't be powered down during s2idle.

* The `nvidia-suspend`, `nvidia-resume` and `nvidia-hibernate` systemd units
  are enabled

  These units are what notify the NVIDIA driver of the suspend and resume
  events.  Without them the driver doesn't get the chance to save and
  restore its state, which can cause the system to fail to resume.  They
  are installed and enabled by the NVIDIA installer:

  ```
  sudo systemctl enable nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service
  ```

* The `nvidia-powerd` daemon is available

  The `nvidia-powerd` daemon is used by the `nvidia-laptop-power-cfg` package
  to configure the NVIDIA driver power state on laptops:

  ```
  sudo systemctl enable --now nvidia-powerd
  ```

  The status of the NVIDIA GPU power management is reported by the driver in
  `/proc/driver/nvidia/gpus/<BDF>/power`:

  ```
  S0ix Power Management:
   Platform Support:          Supported
   Status:                    Enabled
  ```

If the `nvidia-suspend` and `nvidia-resume` systemd units are enabled,
systemd notifies the NVIDIA driver of the suspend and resume events itself.
The tool detects this and does not notify the driver a second time.
