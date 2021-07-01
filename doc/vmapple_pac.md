# ARMv8.3-PAuth in vmapple Guests

Apple CPUs feature an <span style="font-variant: small-caps">implementation
defined</span> ARMv8.3-PAuth cipher, as well as means for securely converting
software-generated 64-bit input keys into PAC keys.  The methods for accessing
this hardware vary across CPU generations.  To isolate guests from these
hardware changes, vmapple guest kernels defer the following operations to the
host hypervisor:

- (re)initializing Apple cipher state
- deriving and reprogramming new PAC keys from a 64-bit input key
- configuring an "EL0 diversifier" value that the hypervisor mixes into PAC keys
  at guest EL0
- temporarily enabling this EL0 diversifier while in guest kernel space

The corresponding hypervisor calls are documented below.  All of these
operations are HVC64 Fast Calls as defined in the [ARM SMC Calling Convention
1.2](https://developer.arm.com/documentation/den0028/).

## APPLE_PAC_SET_INITIAL_STATE

Configures the initial state of Apple's ARMv8.3-PAuth cipher.

**Parameters:**

- `x0`: `0xC1000000`

**Return values**

- `x0`: `0`

## APPLE_PAC_GET_DEFAULT_KEYS

Returns the "default" keys set by `APPLE_PAC_SET_INITIAL_STATE`.  These are
64-bit values that can be passed to `APPLE_PAC_SET_A_KEYS`,
`APPLE_PAC_SET_B_KEYS`, `APPLE_PAC_SET_G_KEYS`, or
`APPLE_PAC_SET_EL0_DIVERSIFIER` to restore the initial key state.


**Parameters:**

- `x0`: `0xC1000001`

**Return values**

- `x0`: `0`
- `x1`: default A key
- `x2`: default B key
- `x3`: default EL0 diversifier
- `x4`: default G key

## APPLE_PAC_SET_A_KEYS

Sets the A key registers (`AP{D,I}AKey`).  The hypervisor securely generates a
set of A keys from a 64-bit input key.


**Parameters:**

- `x0`: `0xC1000002`
- `x1`: 64-bit key

**Return values**

- `x0`: `0`

## APPLE_PAC_SET_B_KEYS

Sets the B key registers (`AP{D,I}BKey`).  The hypervisor securely generates a
set of B keys from a 64-bit input key.


**Parameters:**

- `x0`: `0xC1000003`
- `x1`: 64-bit key

**Return values**

- `x0`: `0`

## APPLE_PAC_SET_EL0_DIVERSIFIER

Sets a diversifier that is mixed into PAC keys when running at guest EL0.  The
hypervisor automatically mixes this diversifier into the PAC key register state
on exit to guest EL0, and automatically removes it from the PAC key register
state on entry to guest EL1.  This diversifier impacts both A and B keys.


**Parameters:**

- `x0`: `0xC1000004`
- `x1`: 64-bit key

**Return values**

- `x0`: `0`

## APPLE_PAC_SET_EL0_DIVERSIFIER_AT_EL1

Sets the EL0 diversifier as in `APPLE_PAC_SET_EL0_DIVERSIFIER`; and explicitly
enables/disables this EL0 key diversifier while running at guest EL1.

This operation is intended for guest EL1 to temporarily use the same keys as a
target guest EL0 process, so that it can auth or sign pointers on that process's
behalf:

```asm
// Switch to target userspace process's keys
MOV64   x0, APPLE_PAC_SET_EL0_DIVERSIFIER_AT_EL1
mov     x1, #1
mov     x2, (target process's diversifier)
hvc     #0

(... use EL0 keys to auth/sign pointers ...)

// Undo key changes
MOV64   x0, APPLE_PAC_SET_EL0_DIVERSIFIER_AT_EL1
mov     x1, #0
mov     x2, (current_task()'s diversifier)
hvc     #0
```

**Parameters:**

- `x0`: `0xC1000005`
- `x1`: `1` to temporarily enable EL0 diversifier at EL1, or `0` to disable it
  again.
- `x2`: 64-bit key

**Return values**

- `x0`: `0`

## APPLE_PAC_SET_G_KEY

Sets the G key registers (`APGAKey`).  The hypervisor securely generates a
G key from a 64-bit input key.


**Parameters:**

- `x0`: `0xC1000006`
- `x1`: 64-bit key

**Return values**

- `x0`: `0`
{"mode":"full","isActive":false}