# PseudoEuicc

LSPosed module that makes ColorOS treat a pluggable eUICC card slot as a built-in eUICC slot, and fills the card EID automatically from the modem layer.

Designed to be used together with the [coloros-esim](https://github.com/GlwFin/coloros-esim) KernelSU module.

## Scope

`com.android.phone`

## How it works

The framework identifies an eUICC slot by the card's ATR announcement (T=15 interface byte per ETSI TS 102 221). Cards that do not announce this capability are not recognized, so the slot is never marked as eUICC and `UiccSlot.mEid` stays empty.

This module hooks `UiccSlot` and `UiccController` at runtime to:

- Force `mIsEuicc` / `mIsRemovable` / `mActive` on the target slot
- Retrieve the real EID from the modem layer (EuiccCard / EuiccPort / TelephonyManager) and fill `mEid` on demand
- Fix `getCardIdForDefaultEuicc` so the framework resolves the correct public card ID

## Slot

The eUICC card must be inserted in **slot 2** (the second card slot). Both SIM slots should have cards inserted for dual SIM mode.

## Releases

A signed APK is provided in [Releases](https://github.com/GlwFin/PseudoEuicc/releases).
