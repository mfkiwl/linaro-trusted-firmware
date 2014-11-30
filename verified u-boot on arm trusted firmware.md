#Tools   
1. FVP : Foundation_v8p. The simulator used to test.  
It can be downloaded from [ARM][] .
[ARM]: http://www.arm.com/zh/products/tools/models/fast-models/foundation-model.php      "ARM"

2. Linux server  
Host working environment, my environment is ubuntu 12.04.04

3. Cross compiler: gcc-linaro-aarch64-none-elf-4.8-2013.11_linux.tar.xz  

#Prepare Software  
1. RAM-disk initrd
Test version can be getted from linaro, Details can be found on ARM-Trusted-Firmware's [Guide][] .   
[Guide]: https://github.com/xiaoqiangdu/arm-trusted-firmware/blob/master/docs/user-guide.md   "user guide"
2. Linux kernel

3. Arm Trusted Firmware

4. Uboot: Universal boot loader
The latest version of Uboot can be downloaded from [UBoot][], or    
git clone git://git.denx.de/u-boot.git

[uboot]: http://git.denx.de/cgi-bin/gitweb.cgi?p=u-boot.git;a=summary  "UBoot"

#Boot system   

###1)Build Uboot  
We need to run system on Foundation, So my target board is vexpress_aemv8a.   
vexpress_aemv8a_semi_config can be selected when you run on FVP platform.  
Modify macro CONFIG_SYS_TEXT_BASE which locate in include/configs/vexpress_aemv8a.h file,   
for BL31 will jump to address 0x88000000, CONFIG_SYS_TEXT_BASE should be modified to this value.  

Compile Uboot as bellow:  
    $cd uboot  
    $make CROSS_COMPILE=<path>/bin/aarch64-none-elf- distclean  
    
    $make vexpress_aemv8a_config    
    The latest version you should use vexpress_aemv8a_defconfig   

    $make CROSS_COMPILE=<path>/bin/aarch64-none-elf- all

###2)Make uImage  
Supposed that linux image has created   

    $cd  <linux_kernel_path>/arch/arm64/boot   
    $/<uboot_path>/tools/mkimage -A arm64 -O linux -T kernel -C none -a 0x80080000 -e 0x80080000  -n 'linux-3.15' -d           Image uImage   
    
Both load address and link address are 0x80080000.

###3)Make firmware Package   
Firmware package includes: bl1.bin/ bl2.bin/ bl31.bin/ bl33.bin(uboot.bin)
    Set variable BL33 as <path_to_uboot_directory>/uboot.bin  
    $cd <firmware_path>  
    $make CROSS_COMPILE=<path>/bin/aarch64-none-elf- PLAT=fvp BL33=<path_to_uboot_directory>/uboot.bin all fip.bin

###4)Running system   
+ Copy all of these images including bl1.bin/fip.bin/uImage/ramdisk/fdt blob to work directory,then enter.   
+ Starting Foundation as bellow:  
$/<path_to_fvp>/Foundation_v8       \  
 --cores=4                 \  
--no-secure-memory        \  
--visualization             \  
--gicv3                   \  
--data=bl1.bin@0x0        \  
--data=fip.bin@0x8000000  \  
--data=uImage@0x90000000 \  
--data=ramdisk_file@0xa1000000 \  
--data=fdt.dtb@0xa0000000   
PS: --data command can be used to load image into FVP’s memory
Boot kernel. Once firmware successfully started, System will stop at uboot’s shell environment. 
+ We can use UBOOT’s bootm command to start linux kernel as below:  

    $bootm 0x90000000 0xa1000000:size 0xa0000000.  

0x90000000 is kernel’s address, 0xa0000000 is device tree dtb’s address,   
and 0xa1000000 is ramdisk’s address, we also need to fill in ramdisk size.  

###5. Verified U-boot  
+ Since I verified it on Foundation platform, So I choice vexpress_aemv8a as the target board for U-Boot.
Edit vexpress_aemv8a.h file, add macros as below:   

    CONFIG_OF_CONTROL  
    CONFIG_RSA  
    CONFIG_FIT_SIGNATURE  
    CONFIG_FIT  
    CONFIG_OF_SEPARATE    

There may exit some compile problem for lack of gpio.h file, this is caused by Uboot's dependency design.  
We need to add a empty gpio.h file to the path arch\arm\include\asm\arch-armv8, just like other boards.  

+ Generate RSA Key pairs with openssl tools  

    key_dir="/work/keys/"  
    key_name="dev"  

Generate the private signing key as:  

    $ openssl genrsa -F4 -out "${key_dir}"/"${key_name}".key 2048  

Generate the certificate containing public key as:   

    $ openssl req -batch -new -x509 -key "${key_dir}"/"${key_name}".key \
       -out "${key_dir}"/"${key_name}".crt  
+ Build control device tree for U-Boot used to contain public key information  
For Uboot need to verify images with public key, so there must be a place used to store public information.  
Here we setup a simple device tree that used to store public key, sample Uboot device tree <u-boot.dts> as:

    /dts-v1/;  
    
    / {  
            model = "Keys";  
            compatible = "linaroSwg FVP";  
            signature {  
                key-dev {  
                        required = "conf";  
                        algo = "sha1,rsa2048";  
                        key-name-hint = "my_key";  
                };  
        };  
    };

+ Edit FIT descripte file.   
Verified boot is based on new U-boot image format FIT, so we need to create a device tree file  
that describes the information about images, including kernel image, FDT blob and RAMDISK.   
FIT file <kernel.its> sample as bellow   
  
/ {  
　　description = "Verified boot";    
　　\#address-cells = <1>;   
　　images {  
　　　　　　kernel@1 {  
　　　　　　　　description = “kernel image”;  
　　　　　　　　data = /incbin/("Image");  
　　　　　　　　type = "kernel_noload";  
　　　　　　　　arch = "arm64";  
　　　　　　　　os = "linux";  
　　　　　　　　compression = "none";  
　　　　　　　　load = < 0x80080000 >;  
　　　　　　　　entry = < 0x80080000 >;  
　　　　　　　　kernel-version = <1>;  
　　　　　　　　signature@1 {  
　　　　　　　　　　algo = "sha1,rsa2048";  
　　　　　　　　　　key-name-hint = "dev";  
　　　　　　　　};  
　　　　　　};  
　　　　　　fdt@1 {  
　　　　　　　　description = "fdb blob";  
　　　　　　　　data = /incbin/("atf_psci.dtb");  
　　　　　　　　type = "flat_dt";  
　　　　　　　　arch = "arm64";  
　　　　　　　　compression = "none";  
　　　　　　　　fdt-version = <1>;  
　　　　　　　　signature@1 {  
　　　　　　　　　　algo = "sha1,rsa2048";  
　　　　　　　　　　key-name-hint = "dev";  
　　　　　　　　};  
　　　　　　};  
　　　　　　ramsidk@1 {  
　　　　　　　　description=”ramdisk”;  
　　　　　　　　data = /incbin/(“filesystem.cpio.gz”);  
　　　　　　　　type = “ramdisk”;  
　　　　　　　　arch = "arm64";  
　　　　　　　　os = "linux";  
　　　　　　　　compression = "gzip";  
　　　　　　　　load = < 0xaa000000 >;  
　　　　　　　　entry = < 0xaa000000 >;  
　　　　　　　　signature@1 {  
　　　　　　　　　　algo = "sha1,rsa2048";  
　　　　　　　　　　key-name-hint = "dev";  
　　　　　　　　};  
　　　　　　};  
　　};  
　　configurations {  
　　　　　　default = "conf@1";  
　　　　　　conf@1 {  
　　　　　　　　kernel = "kernel@1";  
　　　　　　　　fdt = "fdt@1";  
　　　　　　　　ramdisk = “ramdisk@1”;  
　　　　　　};  
　　};  
};   
Pay attention to section key-name-hint, This point to the path of key  generated before.  
Before we build FIT image, kernel image , FDT blob and ramdisk should be prepared.  
Details configure information depends on your own board, you need to select load address   
and entry address for each sub image. Build FIT image and signed the dtb file for Uboot as below:  

    $ dtc -p 0x1000 u-boot.dts -O dtb -o u-boot.dtb  
    $ mkimage -D "-I dts -O dtb -p 2000" -f kernel.its image.fit
    $ mkimage -D "-I dts -O dtb -p 2000" -F -k <path_to_key> -K u-boot.dtb -r image.fit  

Both images are signed with private key and public key has been stored in u-boot.dtb file.  
After the last command line, we can verify that there are new nodes added to u-boot.dtb file, as:  
    $ dtc -I dtb -O dts u-boot.dtb


+ Build FDT U-boot   

    $ make distclean  
    $ make vexpress_aemb8a_config  
    $ make CROSS_COMPILE=<> DEVICE_TREE=foundation all  
    $ make CROSS_COMPILE=<> EXT_DTB=<dtb file>  

Note that, I copied device tree file foundation.dts to Uboot's arch/arm/dts file,   
and made corresponding modifications to the Makefile. Or there will be compile error occurs.  
This is the object what DEVICE_TREE point to.  

EXT_DTB is the dtb file that we signed with public key before in make FIT image step: u-boot.dtb.   
U-Boot can use this to verify the image that signed with private key.  
U-boot-dtb.bin is the file that we need.  
Then Build ATF, and BL33 is point to u-boot-dtb.bin

+ Run the FIT image as bellow  

$/<path_to_fvp>/Foundation_v8       \  
--cores=4                 \  
--no-secure-memory        \  
--visualization             \  
--gicv3                   \  
--data=bl1.bin@0x0        \  
--data=fip.bin@0x8000000  \  
--data=image.fit@0xa2000004  

Note: There exist an alignment problem in U-boot’s mkimage, I traced the code and found  
sometimes may encounter the problem. I avoid this problem by modify the address that FIT image be loaded.   
IE, I loaded image.fit to 0xa2000004 and it is OK, when I load it to 0xa0000000 there would be a abortion exception. 
This problem is mainly related to mkimage tools and FIT format image’s parse. 

After loaded images are verified, Use bootm command to boot kernel as :  

    $bootm 0xa0000004  

PS: There exist some problems in he latest version of Uboot(v2014.10),  
You may need to roll back gic_v64.S file as the older version when you open CONFIG_GICV2 macro.  
Although I have feedbacked the problem to Uboot's maintainer, still I recommend you   
to use the older version of Uboot if you want to verified on Foundation platform currently.  
Or you can substitude gic_v64.S file with the older version and add CONFIG_GICV2 macro in vexpress_aemv8a.h.
