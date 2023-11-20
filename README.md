# Protoplaster

Copyright (c) 2022-2023 [Antmicro](https://www.antmicro.com)

An automated framework for platform testing (Hardware and BSPs).

Currently includes tests for:

* I2C
* GPIO
* Camera
* FPGA

## Installation
```bash
pip install git+https://github.com/antmicro/protoplaster.git
```

## Usage

```
usage: protoplaster [-h] [-t TEST_FILE] [-g GROUP] [--list-groups] [-o OUTPUT] [--csv CSV] [--csv-columns CSV_COLUMNS] [--generate-docs] [-c CUSTOM_TESTS]

options:
  -h, --help            show this help message and exit
  -t TEST_FILE, --test-file TEST_FILE
                        Path to the test yaml description
  -g GROUP, --group GROUP
                        Group to execute
  --list-groups         List possible groups to execute
  -o OUTPUT, --output OUTPUT
                        A junit-xml style report of the tests results
  --csv CSV             Generate a CSV report of the tests results
  --csv-columns CSV_COLUMNS
                        Comma-separated list of columns to be included in generated CSV
  --generate-docs       Generate documentation
  -c CUSTOM_TESTS, --custom-tests CUSTOM_TESTS
                        Path to the custom tests sources
```

Protoplaster expects a yaml file describing tests as an input. That yaml file should have a specified structure.

<!-- name="example" -->
```yaml
base:                # A group specifier
  i2c:               # A module specifier
  - bus: 1           # An interface specifier
    devices:         # Multiple instances of devices can be defined in one module
    - name: "Sensor name"
      address: 0x3c  # The given device parameters determine which tests will be run for the module
  - bus: 2
    devices:
    - name: "I2C-bus multiplexer"
      address: 0x70
  camera:
  - device: "/dev/video0"
    camera_name: "Camera name"
    driver_name: "Driver name"
  - device: "/dev/video2"
    camera_name: "Camera2 name"
    driver_name: "Driver2 name"
    save_file: "frame.raw"
additional:
  gpio:
  - number: 20
    value: 1
```

### Groups
In the yaml file, there is a way to define different groups of tests to run them for different purposes. In the example yaml file, there are two groups defined: base and additional. Protoplaster when run without a defined group, will execute every test in each group. When the group is specified with a parameter `-g` or `--group`, only tests in the specified group are going to be run. There is a possibility to list existing groups in the yaml file, simply run `protoplaster --list-groups test.yaml`.

## Base modules parameters
Each base module has some parameters that are needed for test initialization. Those parameters describe the tests and are passed to the test class as its attributes.

### I2C
Required parameters:

* `bus` - i2c bus to check
* `name` - name of device to be detected
* `address` - address of the device to be detected on the indicated bus

### GPIO
Required parameters:

* `number` - number of the gpio pin
* `value` - the value written to that pin

Optional parameters:

* `gpio_name` - name of the sysfs gpio interface after exporting

### Cameras
Required parameters:

* `device` - path to the camera device (eg. /dev/video0)
* `camera_name` - expected camera name
* `driver_name` - expected driver name

Optional parameters:

* `save_file` - a path which the tested frame is saved to (the frame is saved only if this parameter is present)

### FPGA
Required parameters:

* `sysfs_interface` - path to a sysfs interface for flashing the bitstream to the FPGA
* `bitstream_path` - path to a test bitstream that is going to be flashed

## Writing additional modules
Apart from base modules available in Protoplaster, you can provide your own extended modules. The module should contain a `test.py` file in the root path. That file should contain a test class that is decorated with `ModuleName("")` from `protoplaster.conf.module` package. That decorator tells Protoplaster what the name of the module is. With that information Protoplaster can then correctly initialize the test parameters. The test class should contain a `name()` method. Its return value is used for the `device_name` field in CSV output.

The description of the external module should be added to the yaml file as for other tests. By default, external modules are searched in the `/etc/protoplaster` directory. If you want to store them in a different path, use the `--custom-tests` argument to set your own path. The individual tests run by Protoplaster should be present in the main class in the `test.py` file. The class's name should start with `Test`, and every test's name in that class should also start with `test`. An example of the extended module's test:

```python
from protoplaster.conf.module import ModuleName

@ModuleName("additional_camera")
class TestAdditionalCamera:
    """
    {% macro TestAdditionalCamera(prefix) -%}
    Additional camera tests
    -----------------------
    {% do prefix.append('') %}
    This module provides tests dedicated to camera sensors on specific video node:
    {%- endmacro %}
    """

    def test_exists(self):
        """
        {% macro test_exists(device) -%}
          check if the path exists
        {%- endmacro %}
        """
        assert self.path == "/dev/video0"
```

And a yaml definition:

```yaml
---
base:
  additional_camera:
    - path: "/dev/video0"
    - path: "/dev/video1"
```
## System report
Protoplaster provides `protoplaster-system-report`, a tool to obtain information about system state and configuration. It executes list of provided scripts and saves output of each one, optionally generating summaries. The outputs are stored in single zip archive together with html summary.

### Usage
```
usage: protoplaster-system-report [-h] [-o OUTPUT_FILE] [-c CONFIG]

options:
  -h, --help            show this help message and exit
  -o OUTPUT_FILE, --output-file OUTPUT_FILE
                        Path to the output file
  -c CONFIG, --config CONFIG
                        Path to the yaml config file
```

The yaml config contains list of actions to perform. The single action is described as follows:

```yaml
report_item_name:
  run: script
  summary:
    - name: summary_name
      run: summary_script
  output: script_output_file
  superuser: required | preferred
  on-fail: ...
```

* `run` defines script to run.
* `summary` provides list of summary generators. Each one consisting of `name` that defines name of the summary and `run` with summary script. The summary script can read report script output from its stdin. This field is optional.
* `output` defines output file that will contain script output, its optional if at least one summary is provided.
* `superuser` defines whether script requires elevated privileges to run or not. If its set to `required` the `protoplaster-system-report` will terminate if the privilege requirement is not met. In case of `preferred` it will issue a warning and this particular report item won't be included in the report. This field is optional.
* `on-fail` contains description of the item to run in case of failure. It can be used to run some alternative script if the first one is not available. This field is optional.

Example yaml config file:
```yaml
uname:
  run: uname -a
  summary:
    - name: os info
      run: cat
dmesg:
  run: dmesg
  summary: 
    - name: usb
      run: grep usb
    - name: v4l
      run: grep v4l
  output: dmesg.out
  superuser: required
ip:
  run: ip a
  output: ip.out
  on-fail:
    run: ifconfig -a
    output: ifconfig.out
```
