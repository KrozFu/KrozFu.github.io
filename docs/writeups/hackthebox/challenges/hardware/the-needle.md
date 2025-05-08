# The Needle Challenge

### Context

In this challenge, you must analyze a binary, which comes from a device, which is a firmware for security tests, so you must find the flag within this file.

Challenge description:
> As a part of our SDLC process, we've got our firmware ready for security testing. Can you help us by performing a security assessment?

### Solution

#### Requirements

- file
- binwalk
- grep
- find

#### Step 1 - Security analysis

A quick search of the binary properties revealed the following characteristics, where it can be seen that it is a file for Linux, with an ARM kernel.

```bash
file firmware.bin      
firmware.bin: Linux kernel ARM boot executable zImage (kernel >=v3.17, <v4.15) (big-endian, BE-32, ARMv5)
```

#### Step 2 - Analysis with Binwalk

To analyze this file, we use `Binwalk`, a tool that helps us analyze the code in this file, searching for hidden files. This tool then extracts information that is obfuscated or contained within the file.

```bash
binwalk -e firmware.bin
```

The following information can be observed, where there is a directory
called `_firmware.bin.extracted`.

```bash
binwalk -e firmware.bin

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
14419         0x3853          xz compressed data
14640         0x3930          xz compressed data
```

When you enter and look at its contents, you can see several files.

```bash
➜ ls
firmware.bin  _firmware.bin.extracted
➜ ls _firmware.bin.extracted 
3853.xz  3930  3930.xz  83948.squashfs  squashfs-root  squashfs-root-0
```

#### Step 3 - Using grep

We then use grep to search for any matching credentials, users, or directories that were used when configuring the firmware, performing a specific search for the word `login`, where something in particular is found. The respective user was found with a specific directory `Device_Admin:$sign`,

```ruby
➜ grep -rn "./" -e login

grep: ./_firmware.bin.extracted/squashfs-root/sbin/rpcd: binary file matches
./_firmware.bin.extracted/squashfs-root/etc/inittab:3:::askconsole:/usr/libexec/login.sh
./_firmware.bin.extracted/squashfs-root/etc/config/rpcd:2:config login
./_firmware.bin.extracted/squashfs-root/etc/scripts/telnetd.sh:7:       if [ -f "/usr/sbin/login" ]; then
./_firmware.bin.extracted/squashfs-root/etc/scripts/telnetd.sh:9:               telnetd -l "/usr/sbin/login" -u Device_Admin:$sign      -i $lf &
./_firmware.bin.extracted/squashfs-root/etc/profile:40:in order to prevent unauthorized SSH logins.
./_firmware.bin.extracted/squashfs-root/lib/upgrade/common.sh:133:                              *procd*|*ash*|*init*|*watchdog*|*ssh*|*dropbear*|*telnet*|*login*|*hostapd*|*wpa_supplicant*|*nas*|*relayd*) : ;;
./_firmware.bin.extracted/squashfs-root/lib/preinit/99_10_failsafe_login:5:failsafe_netlogin () {
./_firmware.bin.extracted/squashfs-root/lib/preinit/99_10_failsafe_login:12:    ash --login
./_firmware.bin.extracted/squashfs-root/lib/preinit/99_10_failsafe_login:13:    echo "Please reboot system when done with failsafe network logins"
./_firmware.bin.extracted/squashfs-root/lib/preinit/99_10_failsafe_login:17:boot_hook_add failsafe failsafe_netlogin
grep: ./_firmware.bin.extracted/squashfs-root/lib/libc.so: binary file matches
grep: ./_firmware.bin.extracted/squashfs-root/usr/sbin/dropbear: binary file matches
grep: ./_firmware.bin.extracted/squashfs-root/usr/sbin/pppd: binary file matches
./_firmware.bin.extracted/squashfs-root/usr/lib/opkg/info/busybox.list:3:/bin/login
./_firmware.bin.extracted/squashfs-root/usr/lib/opkg/info/base-files.list:19:/usr/libexec/login.sh
./_firmware.bin.extracted/squashfs-root/usr/lib/opkg/info/base-files.list:77:/lib/preinit/99_10_failsafe_login
./_firmware.bin.extracted/squashfs-root/usr/lib/lua/luci/model/cbi/admin_system/admin.lua:78:ra = s:option(Flag, "RootPasswordAuth", translate("Allow root logins with password"),
./_firmware.bin.extracted/squashfs-root/usr/lib/lua/luci/model/cbi/admin_system/admin.lua:79:   translate("Allow the <em>root</em> user to login with password"))
./_firmware.bin.extracted/squashfs-root/usr/libexec/login.sh:3:[ "$(uci get system.@system[0].ttylogin)" == 1 ] || exec /bin/ash --login
./_firmware.bin.extracted/squashfs-root/usr/libexec/login.sh:5:exec /bin/login
./_firmware.bin.extracted/squashfs-root/usr/share/rpcd/acl.d/unauthenticated.json:8:                                    "login"
grep: ./_firmware.bin.extracted/squashfs-root/bin/busybox: binary file matches
./_firmware.bin.extracted/squashfs-root/bin/config_generate:231:                set system.@system[-1].ttylogin='0'
```

#### Step 4 - Using find

Now we proceed to search for the name of this directory, using `find` to find the specific directory. We can see that it finds the address of the specific directory.

```bash
➜ find ./ -name sign
./_firmware.bin.extracted/squashfs-root/etc/config/sign
```

We proceed to analyze what this directory address contains, we observe that the file found is a text file, which possibly contains the content of a password.

```bash
➜ file ./_firmware.bin.extracted/squashfs-root/etc/config/sign            
./_firmware.bin.extracted/squashfs-root/etc/config/sign: ASCII text
➜ cat ./_firmware.bin.extracted/squashfs-root/etc/config/sign                     
qS6-X/n]u>fVfAt!
```

#### Step 5 - Access to the HTB system

You proceed to enter the system with the user and possible password `Device_Admin:qS6-X/n]u>fVfAt!`, when you enter you can see the `flag.txt`.

```ruby
➜ nc 83.136.255.10 48456

��������
ng-1591596-hwtheneedle-djvrn-6cfb57b64d-gljtm login: Device_Admin
Device_Admin
Password: qS6-X/n]u>fVfAt!    

ng-1591596-hwtheneedle-djvrn-6cfb57b64d-gljtm:~$ ls       
ls
flag.txt
ng-1591596-hwtheneedle-djvrn-6cfb57b64d-gljtm:~$
```

#### Step 6 - Flag

```ruby
ng-1591596-hwtheneedle-djvrn-6cfb57b64d-gljtm:~$ cat flag.txt
cat flag.txt

HTB{4_hug3_blund3r_d289a1_!!}
```

#### Step 5 - Conclusion

In this challenge, the importance of reading and finding hidden files was observed. Using the `binwalk` tool, a hidden directory was found, where several files could be seen that are used for firmware configurations. One of them contained the username and password to log into the system.
