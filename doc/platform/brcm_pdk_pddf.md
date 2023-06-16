 * [S3IP Standard Support](#s3ip-standard-support)
	  * [S3IP PDDF Requirements](#s3ip-pddf-requirements)
	  * [Implementation Details](#implementation-details)
		 * [PDDF and S3IP SysFS](#pddf-and-s3ip-sysfs)
		 * [S3IP SysFS Creation and Mapping](#s3ip-sysfs-creation-and-mapping)
		 * [Adding S3IP Support for a Platform](#adding-s3ip-support-for-a-platform)
| 0.8 | 03/17/2023  |  Fuzail Khan, Precy Lee     | S3IP SysFS support        |
## 7 S3IP Standard Support
S3IP sysfs specification defines a unified interface to access peripheral hardware on devices from different vendors, making it easier for SONiC to support different devices and platforms. The S3IP standard support is now available with PDDF. If the user wants, there is a provision to enable/create S3IP sysfs standards.

### 7.1 S3IP PDDF Requirements

- S3IP sysfs should be generated and could be removed on requirement
- Though S3IP can be clubbed with PDDF, PDDF should be independent of the S3IP
- If any attribute which cannot be read should have a value of 'NA' i.e. tools should not fail due to non existance of the attribute
- S3IP sysfs should be able to work with the existing PDDF common driver sysfs
- PDDF common driver attributes should be expanded, if required, to cover the left out attributes from S3IP specifications

### 7.2 Implementation Details

The S3IP specifications and framework are defined [here](https://github.com/sonic-net/SONiC/pull/1068). Both vendors and users are required to follow the S3IP spec. The platform vendors need to provide the implementation of the set/get attribute functions for the platforms which use S3IP sysfs framework. The attributes for each component are defined in the specificaitons. This effort is to combine the S3IP spec and PDDF framework. In other words, the platform which are using PDDF would be S3IP compliant too after this support is added.

#### 7.2.1 PDDF and S3IP SysFS

PDDF implements common kernel drivers for various components. These common drivers exposes a fixed set of sysfs attributes as per the HW support and current SONiC API requirements. Complying to S3IP spec requires the mapping of S3IP component attributes to PDDF exposed sysfs attributes and might even require adding new attributes to PDDF common driver code. Hence, S3IP spec sysfs attributes are divided into the following categories.

 - Platform Info Attributes: This includes the fixed information pertaining to the platform in entirity or any component. There is no need of reading this information from the component in run time. Further, these values will not change in the course of System running the SONiC image. Below are few examples of static info attributes.

     - /sys_switch/temp_sensor/number, /sys_switch/vol_sensor/number, /sys_switch/curr_sensor/number etc.

     - /sys_switch/cpld/cpld[n]/alias, /sys_switch/temp_sensor/temp[n]/alias, /sys_switch/temp_sensor/temp[n]/type etc.

   S3IP file system can be created and the information can be directly written from the PDDF JSON files to them. Since this is static information, there is no need of repeatedly updating it.

 - Component Attributes: These are the attributes are to be read from various HW components. These could be dynamically changing information like PSU voltage and temperature, or fixed like PSU serial number or FAN model etc.

     - Some such S3IP sysfs would match the sysfs exposed by the PDDF frameowrk and hence a proper mapping with softlink creation would suffice.

     - Some S3IP sysfs would not match directly with PDDF exposed sysfs. For such attributes, either PDDF common dirvers can be enhanced to provide the exact match or some other method can be used.

     - There are some S3IP attributes which can't and won't be mapped to common PDDF driver attributes because PDDF common code doesn't support these devices e.g. volt_sensors, curr_sensors etc. There are no PDDF common drivers for such devices. However, ODMs might extend the PDDF framework by providing the custom drivers. In such cases, ODMs need to take care of mapping the S3IP attributes to the attributes exposed by the custom drivers.

     - We faced a practical issue for some S3IP attributes e.g. Fan status. S3IP description says

     |Sysfs path|Permissions|Data type|Description|
     |-|-|-|-|
     |/sys_switch/fan/fan[n]/status |RO| enum| Fan states are defined as follows:<br>0: not present<br>1: present and normal<br>2: present and abnormal

     - This is a combination of 'presence' and 'running_status' informations of a fan unit. In SONiC we can handle this in the platform APIs but S3IP compels to performs this processing inside the kernel modules. Hence if ODM extends the PDDF driver and provide the kernel implementation of such sysfs, we can create the mapping. Otherwise we will map it to 'NA'.

#### 7.2.2 S3IP SysFS Creation and Mapping

![S3IP Support in PDDF](../../images/platform/s3ip_pddf.jpg "S3IP Support in PDDF")


If the S3IP sysfs is required on a PDDF platform, it can be represented using the field "enable_s3ip" in the PDDF JSON file. If this field is not mentioned or has a value "no", then the S3IP sysfs creation is disabled for that platform. The support for S3IP is controlled by a service "pddf-s3ip-init.service". This service is run at the end of PDDF platform initialization service. It is an standalone service which needs to have an 'after' dependency on PDDF platform init service.
```
    "PLATFORM":
    {
        "num_psus":2,
        "num_fantrays":4,
        "num_fans_pertray":2,
        "num_ports":64,
        "num_temps": 8,
        "enable_s3ip": "yes",
        "pddf_dev_types":
        {
            "description":"AS7816-64X - Below is the list of supported PDDF device types (chip names) for various components. If any component uses some other driver, we will create the client using 'echo <de
            "CPLD":
            [
                "i2c_cpld"
            ],
            "PSU":
            [
                "psu_eeprom",
                "psu_pmbus"
            ],
            "FAN":
            [
                "fan_ctrl",
                "fan_eeprom"
            ],
            "PORT_MODULE":
...
...
```

This pddf-s3ip service would create the sysfs as per the standards. It will also take care of linking the appropriate PDDF sysfs with the corrosponding S3IP sysfs.

In case the platform does not support some attributes present in the S3IP spec, 'NA' will be written to the attribute file so that the application does not fail.

Once this is done, users can run their S3IP compliant applicaitons and test scripts on the platform.

#### 7.2.3 Adding S3IP Support for a Platform

For adding support for S3IP on a platform which is already using PDDF for bringup, here are the steps.

- Add "enable_s3ip": "yes" into the pddf-device.json file for that platform
```
         "num_fans_pertray":1,
         "num_ports":54,
         "num_temps": 3,
+        "enable_s3ip": "yes",
         "pddf_dev_types":
         {
```

- Create a softlink for the 'pddf-s3ip-init.service' inside service folder for that platform.

```
diff --git a/platform/broadcom/sonic-platform-modules-<odm>/<platform>/service/pddf-s3ip-init.service b/platform/broadcom/sonic-platform-modules-<odm>/<platform>/service/pddf-s3ip-init.service
new file mode 120000
index 000000000..f1f7fe768
--- /dev/null
+++ b/platform/broadcom/sonic-platform-modules-<odm>/<platform>/service/pddf-s3ip-init.service
@@ -0,0 +1 @@
+../../../../pddf/i2c/service/pddf-s3ip-init.service
\ No newline at end of file


# ls -l platform/broadcom/sonic-platform-modules-<odm>/<platform>/service/pddf*
total 3
lrwxrwxrwx 1 fk410167 nwsoftusers  55 May  8 02:02 pddf-platform-init.service -> ../../../../pddf/i2c/service/pddf-platform-init.service
lrwxrwxrwx 1 fk410167 nwsoftusers  55 May  8 02:02 pddf-s3ip-init.service -> ../../../../pddf/i2c/service/pddf-s3ip-init.service
#

```

Build the platform and sonic-device-data debian packages and load the build on the respective platform.


## 8 Warm Boot Support
## 9 Scalability
## 10 Unit Test