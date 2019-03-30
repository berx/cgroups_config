# cgroups_config

# purpose
cgc is a script which set cgroup limits for a specific set of disks and assigns processes to this cgroup.
It is designed for the content of Oracle RDBMS running on ASM, but probably can be extended to other purpose also. 

For each "groups"."name" in cgconfig.json, for each it's ."subystems"."subsystem" a dedicated cgroup is created. 
There each ."limits"."file" is filled with all the device-ids matching the "diskgroup" and translated into devices by the script GET_DEVICES.
All processes which matches any of the patterns in "processes" are added to the cgroups **tasks** file, so the limits apply to them. 

This implementation is absed on cgroups v1.

A description can be found (here)[https://berxblog.blogspot.com/2019/03/limit-ios-by-instances.html].

# requirements

## cgroups filesystem
* a proper cgroups mountpoint must exist. ("base" in cgconfig.json)
* for each **subsystem** a dedicated directory "myroot" must exist and writable by the user who is running cgc.

## jq
to parse the config file (jq)[https://stedolan.github.io/jq/] is required. 

## GET_DEVICES
a program which translates a disk as it's given by *select path from v$asm_disk* into real device names. 
In the current script a local copy of *get_real_device* from https://github.com/berx/asm_helpers is used. to replace it the variables GET_DEVICES and GET_DEVICES_PARAMETERS can be used.

## cgconfig.json
the config file must be a json which *jq* can parse. 
It contains 2 sections, a kind of "header" where global settings are defined, and a list of "groups" which define the diskgroup, cgroup-subsystems, limits and processes-patterns:
```json
{
  "base": "/sys/fs/cgroup",
  "myroot": "ORACLE",
  "subsystems": [
    "blkio"
  ],
  "groups": [
    {
      "name": "BX1",
      "diskgroup": "DG1",
      "subsystems": [
        {
          "subsystem": "blkio",
          "limits": [
            {
              "file": "blkio.throttle.read_bps_device",
              "limit": 5242880
            },
            {
              "file": "blkio.throttle.read_iops_device",
              "limit": 3000
            }
          ]
        }
      ],
      "processes": [
        "ora_p[[:digit:]].{2}_BXSR01",
        "BXSR01.*LOCAL=NO"
      ]
    },
    {
      "name": "BX2",
      "diskgroup": "DG_42",
      "subsystems": [
        {
          "subsystem": "blkio",
          "limits": [
            {
              "file": "blkio.throttle.read_bps_device",
              "limit": 10485760
            },
            {
              "file": "blkio.throttle.read_iops_device",
              "limit": 2000
            }
          ]
        }
      ],
      "processes": [
        "ora_p[[:digit:]].{2}_BXTST02",
        "BXTST02.*LOCAL=NO"
      ]
    }
  ]
}
``` 

## options

    cgc [-?] [-C config_file] [-G group_name]|[-a] [-d] [-p]
      -?            - this page
      -C            - specify the config_file to be used
      -G            - specify the group_name which wil lbe processed
      -a            - all groups
      -d            - process limits for the diskgroup(s) (depending on -G|-a)
      -p            - process processes filters (depending on -G|-a)

    cgc sets cgroups limits for a given "group" and assign processes 
    based on patterns to this cgroup. 
    A well formatted config file is required.
	
## crontab
an example crontab entry to check for changes in devices every hour and for new processes every minute

    # min   hour day_month month day_week   command
    1       *       *       *       *       /appl/oracle/cgroups_config/cgc -a -d
    *       *       *       *       *       /appl/oracle/cgroups_config/cgc -a -p
