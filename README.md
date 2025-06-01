# urve-pi-wg-kernel-support
Build and deploy URVE PI's Kernel with WireGuard support

## 🔍 References

- URVE BSP Repo: [urveboard3566bsp Git](https://urveboard.eveo.pl/urve/urveboard3566bsp)
- Docs: [URVE Pi | FAQ URVEBoard](https://faq.urveboard.com/shelves/urve-pi)
![[URVE_Board_User_Manual.pdf]]

## 🛠️ Build and deploy new URVE PI's Kernel with WireGuard support

### 📦 Prerequisites

- `urveboard3566bsp` repository
- Docker installed

### 🧪 Build Environment Setup (Docker)

The following script will start or create the required Docker container, then connect to it to install the necessary build environment

```
#!/bin/bash

BSP_PATH="/home/nementon/builds/urve-pi/urveboard3566bsp"
CONTAINER_NAME="kernel-builder"

if [ "$(docker ps -a --filter "name=^/${CONTAINER_NAME}$" --format '{{.Names}}')" = "${CONTAINER_NAME}" ]; then
  echo "🟢 Starting existing container: ${CONTAINER_NAME}"
  docker start "${CONTAINER_NAME}"
else
  echo "🌟 Creating new container: ${CONTAINER_NAME}"
  docker run -it --name "${CONTAINER_NAME}" -v "${BSP_PATH}:/build" ubuntu:22.04
fi

docker exec -it "${CONTAINER_NAME}" bash -c "cd /build && exec bash -l"
```

#### 📁 Install Container Build Dependencies

Run the following commands into the spawned container to install and configure the required build dependencies

```
apt update && apt upgrade -y
apt install -y \
  repo git ssh make gcc g++ libssl-dev liblz4-tool expect \
  patchelf chrpath gawk vim texinfo diffstat binfmt-support \
  qemu-user-static live-build bison flex fakeroot cmake \
  gcc-multilib g++-multilib unzip device-tree-compiler \
  python-pip ncurses-dev sudo python python2.7 tmux htop

ln -s /usr/bin/python2.7 /usr/bin/python

git config --global --add safe.directory /build
```

### ⚙️ Kernel Device Tree Update for eMMC (vccio2-supply)

To ensure proper voltage regulation for the eMMC memory, you must define the `vccio2-supply` property in the device tree. By default, this node may be missing, which can result in incorrect voltage (e.g., 3.3V) being supplied where 1.8V is expected, potentially damaging the eMMC. (Reference to the **IO Power Domain MAP** page 14 of the user documentation)

```
root@46e8ccd00568:/build# git diff kernel/arch/arm64/boot/dts/rockchip/rk3568-evb.dtsi  
diff --git a/kernel/arch/arm64/boot/dts/rockchip/rk3568-evb.dtsi b/kernel/arch/arm64/boot/dts/rockchip/rk3568-evb.dtsi  
index e9b027fd9..8098070e4 100755  
--- a/kernel/arch/arm64/boot/dts/rockchip/rk3568-evb.dtsi  
+++ b/kernel/arch/arm64/boot/dts/rockchip/rk3568-evb.dtsi  
@@ -1364,6 +1364,7 @@  
pmuio1-supply = <&vcc_3v3>; //<&vcc3v3_pmu>;  
pmuio2-supply = <&vcc_3v3>; //<&vcc3v3_pmu>;  
vccio1-supply = <&vcc_3v3>; //<&vccio_acodec>;  
+ vccio2-supply = <&vcc_1v8>; //<&vccio_flash>;  
vccio3-supply = <&vcc_3v3>; //<&vccio_sd>;  
vccio4-supply = <&vcc_1v8>;  
vccio5-supply = <&vcc_3v3>;
```
----
📝 **Note**: This change was applied with assistance from a LLM. While its effectiveness nor usefulness is not fully validated, the eMMC hardware has remained functional after applying this change. The eMMC has not been damaged due to incorrect voltage -- it has either no effect or the expected effect.
#### 🔎LLM proposed validation steps
After applying the patch and rebuilding the device tree:

1. **Recompile the Device Tree:**
```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- dtbs
```

2. **Confirm the property in the compiled `.dtb` file:**
Convert the DTB back to DTS to inspect:
```
dtc -I dtb -O dts -o output.dts kernel/arch/arm64/boot/dts/rockchip/rk3568-evb.dtb grep vccio2-supply output.dts
```
Expected output:
```
vccio2-supply = <&vcc_1v8>;
```
3. **Boot the system** and confirm that eMMC operates correctly without voltage-related errors in the kernel log (`dmesg`).
#### 📚 Reference

For further details, refer to the official Rockchip documentation on hardware and power domains:

- Rockchip Developer Documentation (Power Domains & IO Supplies):  
  https://opensource.rockchip.com

- URVEBoard documentation and BSP tools:  
  [https://faq.urveboard.com/shelves/urve-pi](https://faq.urveboard.com/shelves/urve-pi)  
  [https://urveboard.eveo.pl/urve/urveboard3566bsp](https://urveboard.eveo.pl/urve/urveboard3566bsp)

----
### 🧩 Kernel Configuration for WireGuard Support

The expected kernel configuration file is`./kernel/arch/arm64/configs/rockchip_linux_defconfig`, However, as a precaution, updates will also be applied to `./kernel/.config` in case this file is unexpectedly in use.

Ensure that the required options are set to `y` (built-in). While enabling them as modules (`m`) might theoretically work, it has been observed that using `lsmod` to list kernel modules sometimes causes the system to hang or freeze for unknown reasons. Using built-in options is ensure proper WireGuard functionality.

The following script checks whether the required kernel configuration options are present in the two expected configuration files.

> ⚠️**Note:**  
   The list of kernel configuration options required to enable **WireGuard**, **TUN**, and **full Netfilter support** was generated with the help of a LLM. While some entries may be redundant or not strictly necessary (i.e., potential hallucinations), the configuration has been successfully tested in practice and shown to work reliably.

```
#!/bin/bash

# Path to your kernel config (adjust as needed)

KCONFIG="./kernel/arch/arm64/configs/rockchip_linux_defconfig"
KCONFIG_BACKUP="./kernel/.config"

if [ ! -f "$KCONFIG" ]; then
    echo "❌ Kernel config not found at $KCONFIG"
    exit 1
fi

echo "🔍 Checking kernel config at: $KCONFIG"

REQUIRED_OPTIONS=(
  "CONFIG_WIREGUARD"
  "CONFIG_CRYPTO"
  "CONFIG_CRYPTO_AEAD"
  "CONFIG_CRYPTO_BLKCIPHER"
  "CONFIG_CRYPTO_CHACHA20"
  "CONFIG_CRYPTO_POLY1305"
  "CONFIG_CRYPTO_CHACHA20POLY1305"
  "CONFIG_CRYPTO_CURVE25519"
  "CONFIG_CRYPTO_X25519"
  "CONFIG_CRYPTO_HKDF"
  "CONFIG_CRYPTO_LIB_CURVE25519"
  "CONFIG_CRYPTO_LIB_BLAKE2S"
  "CONFIG_CRYPTO_LIB_CHACHA20POLY1305"
  "CONFIG_CRYPTO_USER_API"
  "CONFIG_CRYPTO_USER_API_HASH"
  "CONFIG_TUN"
  "CONFIG_NETFILTER"
  "CONFIG_NETFILTER_ADVANCED"
  "CONFIG_NF_CONNTRACK"
  "CONFIG_NF_NAT"
  "CONFIG_NF_NAT_REDIRECT"
  "CONFIG_NF_DEFRAG_IPV4"
  "CONFIG_NETFILTER_XTABLES"
  "CONFIG_NETFILTER_XT_TARGET_REDIRECT"
  "CONFIG_NETFILTER_XT_MATCH_TCP"
  "CONFIG_NETFILTER_XT_MATCH_UDP"
  "CONFIG_NETFILTER_XT_MATCH_CONNTRACK"
  "CONFIG_IP_NF_IPTABLES"
  "CONFIG_IP_NF_FILTER"
  "CONFIG_IP_NF_TARGET_REDIRECT"
  "CONFIG_IP_NF_NAT"
  "CONFIG_IP_NF_TARGET_MASQUERADE"
)

MISSING_KCONFIG=0
echo "[-] ${KCONFIG} configuration lookup:"
for opt in "${REQUIRED_OPTIONS[@]}"; do
    if grep -q "^$opt=" "$KCONFIG"; then
        val=$(grep "^$opt=" "$KCONFIG" | cut -d= -f2)
        echo "✅ $opt is set to $val"
    elif grep -q "^# $opt is not set" "$KCONFIG"; then
        echo "❌ $opt is NOT set"
        MISSING_KCONFIG=$((MISSING_KCONFIG + 1))
    else
        echo "⚠️  $opt is not found in config (may be renamed or missing)"
        MISSING_KCONFIG=$((MISSING_KCONFIG + 1))
    fi
done

MISSING_KCONFIG_BACKUP=0
echo "[-] ${KCONFIG_BACKUP} configuration lookup:"
for opt in "${REQUIRED_OPTIONS[@]}"; do
    if grep -q "^$opt=" "$KCONFIG_BACKUP"; then
        val=$(grep "^$opt=" "$KCONFIG" | cut -d= -f2)
        echo "✅ $opt is set to $val"
    elif grep -q "^# $opt is not set" "$KCONFIG"; then
        echo "❌ $opt is NOT set"
        MISSING_KCONFIG_BACKUP=$((MISSING_KCONFIG_BACKUP + 1))
    else
        echo "⚠️  $opt is not found in config (may be renamed or missing)"
        MISSING_KCONFIG_BACKUP=$((MISSING_KCONFIG_BACKUP + 1))
    fi
done

echo
if [ $MISSING_KCONFIG -eq 0 ]; then
    echo "✅ Your kernel ${KCONFIG} config supports WireGuard and TUN."
else
    echo "❌ $MISSING_KCONFIG required options are missing in ${KCONFIG}."
fi

if [ $MISSING_KCONFIG_BACKUP -eq 0 ]; then
    echo "✅ Your kernel ${KCONFIG_BACKUP} config supports WireGuard and TUN."
else
    echo "❌ $MISSING_KCONFIG_BACKUP required options are missing in ${KCONFIG_BACKUP}."
fi
```

### 📊 Build Process

All following commands has to be executed within your build environment, eg: docker container if in use.
#### 🔹 Build Kernel Only

It is recommended to first re-compile the Device Tree that may be needed, or to apply of full system build.

```
source ./envsetup.sh rockchip_rk3566
./build.sh kernel
```

In case of success, the build kernel image will be available at: `./kernel/boot.img`

#### 🔹 Full System Build (Recommended -- ensure everything is working as expected)

This will ensure to re-compile the device tree, that may be needed.

```
./urve_prepare.sh
```

In case of success, the build images (including the kernel `boot.img`) will be available at: `./rockdev/FlashTool/rockdev/Image/`

```
root@46e8ccd00568:/build# ls -lha rockdev/FlashTool/rockdev/Image/  
total 2.6G  
drwxr-xr-x 2 1000 1000 4.0K May 31 11:48 .  
drwxr-xr-x 3 1000 1000 4.0K May 17 2023 ..  
-rw-r--r-- 1 1000 1000 0 May 25 23:45 .gitkeep  
-rw-r--r-- 1 1000 1000 435K May 25 23:45 MiniLoaderAll.bin  
-rw-r--r-- 1 1000 1000 37M May 25 23:45 boot.img  
-rwxr-xr-x 1 1000 1000 48K May 25 23:45 misc.img  
-rw-r--r-- 1 1000 1000 17M May 25 23:45 oem.img  
-rw-r--r-- 1 1000 1000 488 May 25 23:45 parameter.txt  
-rw-r--r-- 1 1000 1000 47M May 25 23:45 recovery.img  
-rw-r--r-- 1 1000 1000 2.7G May 19 2023 rootfs-debian.img  
-rw-r--r-- 1 1000 1000 4.0M May 25 23:45 uboot.img  
-rw-r--r-- 1 1000 1000 4.3M May 25 23:45 userdata.img
```

### 🚀 Kernel Deployment

To deploy the kernel, the **RKDevTool** provided by URVE is required. This appears to be a Rockchip tool bundled with custom configurations for URVE PI memory mapping:  
[Download RKDevTool for URVE PI](https://oc.app.eveo.pl/s/SUYOaM4F6uJPV2Q/download)

To connect the URVE PI board in flashing mode:
1. Use a USB **A-to-A** cable to connect to the device's black USB port (1).
2. While powering on the device, **press and hold the recovery button** (2).

>⚠️**Note:**
>Without a USB **A-to-A**, the device won't be recognized 

![Pasted image 20250601165641](https://github.com/user-attachments/assets/b0868187-4d28-4a8f-af58-63280017a55c)

If the device is correctly detected, the RKDevTool’s status at the bottom should change from "**No Device Found**" to "**Found One MASKROM Device**".

![Pasted image 20250601165853](https://github.com/user-attachments/assets/f445438f-e0b1-4bcf-9deb-ffa64e3dec51)
![Pasted image 20250601165916](https://github.com/user-attachments/assets/c0c216af-48b1-4216-ad46-2da6e563415b)

Once the device is detected, go to the **Download Image** tab (used for pushing files to the device), and:

- **Uncheck all items** except for **boot.img**.
- Update the **Path** for `boot.img` to point to the `boot.img` file generated from your kernel build.
- Click the **Run** button to deploy the image.

The kernel image deployment should only take a few seconds. After completion, the URVE PI board will automatically reboot.
