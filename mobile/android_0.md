# Android Application Pentesting - Part 0
Some tricks to use when testing Android Apps

## Setting proxy
* Go to Settings > WiFi > Your network > Proxy, and fill in the proxy tool's ip address and port
* if there is AP isolation: 
    * In the computer: `adb reverse tcp:8080 tcp:8080`
    * In the android: Set the WiFi proxy using `127.0.0.1` and `8080`
* Apps ignoring the proxy (Flutter)
    - Option 1) ProxyDroid (root is required, available in playstore)
    - Option 2) Transparent Proxy
        * Option A) Use an WiFi adapter to create an AP on your computer and connect the Android
        * Option B) (If the app does not need ipv6) Use your default WiFi router (without AP isolation) 
            - Go to Settings > WiFi > Your network and fill in your computer's ip address as the gateway and the DNS server
            - Search for "How to enable ip forwarding" in your computer's system, if it is a linux:
                * https://linuxconfig.org/how-to-turn-on-off-ip-forwarding-in-linux
            - Set the computer's firewall, if it is a linux:
            ```sh
            iptables -A INPUT -p udp --source $deviceAddr --dport 53 -j ACCEPT
            iptables -A INPUT -p tcp --source $deviceAddr --dport 8080 -j ACCEPT
            iptables -t nat -A PREROUTING -p tcp --source $deviceAddr -j REDIRECT --to-ports 8080
            ```
            - Start a DNS server ignoring ipv6, if it is a linux:
            ```sh
            echo '
            no-resolv
            log-queries
            server=/*/8.8.8.8

            # ignoring ipv6
            address=/*/::

            listen-address=0.0.0.0
            bind-interfaces
            ' > ~/dnsmasq.conf
            sudo dnsmasq -C ~/dnsmasq.conf --no-daemon
            ```

## Setting CA certificate
* Get the certificate file (http://burp, http://mitm.it, …)
* Install the certificate
    - As a User Certificate (via Settings / File Manager)
        - Simple but problematic since Android 7 (https://android-developers.googleblog.com/2016/07/changes-to-trusted-certificate.html)
        - Even so, sometimes it can still work and, if not, you can try: 
            - A) **[No root]** Recompile app: https://github.com/shroudedcode/apk-mitm
            - B) Hook the config: https://medium.com/keylogged/bypassing-androids-network-security-configuration-575819a8f317
            - C) Use a magisk module like MagiskTrustUserCerts
    - As a System Certificate (putting the certificate in /system/etc/security/cacerts)
        - A) Turn /system writable
            - May not work in some devices running [Android 10+](https://android.stackexchange.com/a/220920). Regardless, if the device is running a custom ROM, it can work.
            - https://gist.github.com/morkin1792/b7d72267461121ad3ddbdf4c52785f24
        - B) Use a custom recovery (like TWRP)
            - **[No root straight on Android]** Use the adb in recovery to have root file system access
        - C) Mount a writable filesystem on the certificates directory
            * simple way but not persistent (at least if you just use the command below)
            * android root shell: `mount -t tmpfs tmpfs /system/etc/security/cacerts`

## Bypassing root detection
- Can't you use a **non-rooted device**, installing the CA certificate without root as detailed above?
    - If the app implements certificate pinning:
        - https://github.com/mitmproxy/android-unpinner
        - Frida without root
            - https://lief-project.github.io/doc/latest/tutorials/09_frida_lief.html
            - https://fadeevab.com/frida-gadget-injection-on-android-no-root-2-methods/
            - https://jlajara.gitlab.io/Frida-non-rooted
            - https://koz.io/using-frida-on-android-without-root/
- MagiskDenyList ~~MagiskHide (v23)~~
- Frida scripts
    - https://codeshare.frida.re/@dzonerzy/fridantiroot/
    - https://github.com/sensepost/objection
    - "root" inurl:codeshare.frida.re
- Manual solution
    * Static
        * what look for?
            - generic strings (magisk, supersu, root)
            - stacktrace (logcat)
            - message appearing in the app
            - diff using old versions without the protection
        * jadx, ghidra, ida pro
        * if the app was built in react-native and its bundle is not obfuscated: js-beautify 
    * Dynamic
        * what look for?
            - methods found in static analysis
            - loaded methods/classes names
            - use a debugger and check the stacktrace
            - syscalls hooking
        * Frida
        * Debbuger (gdb via termux)

## Bypassing Play Integrity API (~~SafetyNet Attestation~~)
* Play Integrity Fix (https://github.com/chiteroman/PlayIntegrityFix)
* Universal SafetyNet Fix (https://github.com/kdrag0n/safetynet-fix)

## Bypassing Certificate Pinning
- [pinning](pinning.md)

## Bypassing Developer Mode detection
- Try Frida CodeShare
    - to check: https://codeshare.frida.re/@zionspike/bypass-developermode-check-android/
- Disable developer mode
    * You can let frida server running as daemon (`frida-server -D`) and connect via network
    * Use termux (you can install ssh server and/or frida client)
- Manual analysis
