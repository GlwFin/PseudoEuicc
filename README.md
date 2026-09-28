# PseudoEuicc

LSPosed module that makes ColorOS treat a pluggable eUICC card slot as a built-in eUICC slot, and fills the card EID automatically from the modem layer.

Used together with the [coloros-esim](https://github.com/星坠青川/coloros-esim) KernelSU module to enable native eSIM settings on China ColorOS.

## Scope

`com.android.phone`

## Build

```
javac --release 8 -cp android.jar;xposedbridge-api-82.jar app/src/com/example/pseudoeuicc/Main.java
d8 --release --min-api 29 classes/*.class
```

Signed APK is provided in Releases.
