---
theme:  ../../template
transition: slide-left
mdc: true
layout: cover
hideInToc: true
---

# gpio-i2c
## An introduction to GPIO-I2C
### Meena Murthy

---
layout: default
hideInToc: true
---

# Table of contents

<Toc minDepth="1" maxDepth="2"/>

---
layout: default
---
# i2c-gpio: Key Capabilities
<div class="center">

- Linux kernel driver for bitbanging I2C bus using the GPIO API.
- It provides functionality for toggling SDA and SCL lines, handling incomplete I2C transfers, and injecting faults for testing purposes.
- The driver supports platform data configuration and works with ACPI and Device Tree setups.

</div>
---
layout: default
---

# GPIO API
<div class="center">

GPIO API provides functions(all functions starting as gpiod_) to configure, read and write to GPIO pins.

</div>
<!--The GPIO API in the I2C GPIO driver context allows the driver to control the GPIO pins that simulate the I2C bus signals. By using functions from this API, the driver can bitbang the I2C protocol, providing a software implementation of I2C communication using general-purpose pins. --> 

---
layout: default
---

# What is bit banging?

Bit banging is a method of using software to send and receive signals which would otherwise be done by hardware which in this case would be UART, SPI, I2C

## Why we need it here?

- Makes platform independent as long as hardware supports GPIO pins
- Flexible to work with custom hardware
- Flexible to work with systems lacking dedicated hardware controllers

---

# Let's dive into the code

<div class="center">

Entry point

```c
static struct platform_driver i2c_gpio_driver = {
	.driver		= {
		.name	= "i2c-gpio",
		.of_match_table	= i2c_gpio_dt_ids, //Device Tree Match Table
		.acpi_match_table = i2c_gpio_acpi_match, //ACPI Match Table
	},
	.probe		= i2c_gpio_probe,
	.remove_new	= i2c_gpio_remove,
};

static int __init i2c_gpio_init(void)
{
	int ret;

	ret = platform_driver_register(&i2c_gpio_driver);
	if (ret)
		printk(KERN_ERR "i2c-gpio: probe failed: %d\n", ret);

	return ret;
}
subsys_initcall(i2c_gpio_init);

```
</div>
<!--
Populating data inside i2c_gpio_driver struct
Initialization and registration of a platform driver for an I2C GPIO driver in the Linux kernel
__init is to reclaim memory after initialization (especially valuable on embedded or memory-constrained systems).
-->
---

# I2C GPIO driver code

Start condition for claiming the bus

Step 1
```c
i2c_gpio_setsda_val(void *data, int state)
```

Step 2
```c
i2c_gpio_setscl_val(void *data, int state)
```

<!--
state - The state to set the SDA pin to (0 for low, 1 for high)
(show image of slight delay scl starts with)

What is data here?

SDA
data being written is any byte-level communication from master to slave, such as:
Device address
Register index
Value to write

SCL
desired state of the SCL line
-->

---

```c
i2c_gpio_getsda(void *data)

Helper function

static int i2c_gpio_getsda(void *data)
{
	struct i2c_gpio_private_data *priv = data;

	return gpiod_get_value_cansleep(priv->sda);
}
```
<!--
gpiod_get_value_cansleep() reads the current value (high or low) of the SDA GPIO line.

used as a callback when the i2c bit-banging driver needs to know the state of the data line.

priv->sda is the GPIO descriptor for the SDA (data) line.
-->
---

```c
i2c_gpio_getscl(void *data)

Helper function

static int i2c_gpio_getscl(void *data)
{
	struct i2c_gpio_private_data *priv = data;

	return gpiod_get_value_cansleep(priv->scl);
}

```
<!--
Purpose of reading the current state of the SCL line:

Detect clock stretching

Ensure the line is actually high before toggling
-->
---

# i2c_gpio_get_properties

```c
static void i2c_gpio_get_properties(struct device *dev,
				    struct i2c_gpio_platform_data *pdata)
{
	u32 reg;

	device_property_read_u32(dev, "i2c-gpio,delay-us", &pdata->udelay);

	if (!device_property_read_u32(dev, "i2c-gpio,timeout-ms", &reg))
		pdata->timeout = msecs_to_jiffies(reg);

	pdata->sda_is_open_drain =
		device_property_read_bool(dev, "i2c-gpio,sda-open-drain");
	pdata->scl_is_open_drain =
		device_property_read_bool(dev, "i2c-gpio,scl-open-drain");
	pdata->scl_is_output_only =
		device_property_read_bool(dev, "i2c-gpio,scl-output-only");
	pdata->sda_is_output_only =
		device_property_read_bool(dev, "i2c-gpio,sda-output-only");
	pdata->sda_has_no_pullup =
		device_property_read_bool(dev, "i2c-gpio,sda-has-no-pullup");
	pdata->scl_has_no_pullup =
		device_property_read_bool(dev, "i2c-gpio,scl-has-no-pullup");
}
```
<!--
hardware properties (or characteristics) of GPIO pins used for I²C or other digital signals

Typically ARM architecture uses device tree and x86 uses ACPI

device_property_read_bool does not have to worry about if it is DT or ACPI

1. Hardware properties (or characteristics) of GPIO pins used for i2c are read
2. Then, the necessary configuration properties from the firmware (DT or ACPI) are assigned to the driver's platform data structure, enabling the I2C GPIO driver to be properly configured based on the hardware description provided by the system.
-->
---

# i2c_gpio_get_desc

```c
	struct gpio_desc *retdesc;
	int ret;

	retdesc = devm_gpiod_get(dev, con_id, gflags);
	if (!IS_ERR(retdesc)) {
		dev_dbg(dev, "got GPIO from name %s\n", con_id);
		return retdesc;
	}
```
<!--
To obtain a GPIO descriptor that represents a specific GPIO line associated with a device. This descriptor is then used to perform GPIO-related operations such as setting or getting the GPIO value, configuring the GPIO direction, etc

 %-ENOENT if no GPIO has been assigned to the requested function, or
 IS_ERR() code if an error occurred while trying to acquire the GPIO.

-->
---

```c
	retdesc = devm_gpiod_get_index(dev, NULL, index, gflags);
	if (!IS_ERR(retdesc)) {
		dev_dbg(dev, "got GPIO from index %u\n", index);
		return retdesc;
	}

	ret = PTR_ERR(retdesc);
```
<!--
Why are both functions needed, when inside devm_gpiod_get(), devm_gpiod_get_index() is going to be called anyway by assigning index as 0?
Because some platforms specify GPIOs by name, and others just by index.

Name-based first - more descriptive, preferred when available.
				 - supports older bindings and is user-friendly for devices that specify GPIOs by label.

Fallback to index — catches unnamed bindings, generic configurations, or incomplete device trees.
-->
---

```c
	/* FIXME: hack in the old code, is this really necessary? */
	if (ret == -EINVAL)
		retdesc = ERR_PTR(-EPROBE_DEFER);

	/* This happens if the GPIO driver is not yet probed, let's defer */
	if (ret == -ENOENT)
		retdesc = ERR_PTR(-EPROBE_DEFER);

	if (PTR_ERR(retdesc) != -EPROBE_DEFER)
		dev_err(dev, "error trying to get descriptor: %d\n", ret);

	return retdesc;
```

---

# i2c_gpio_probe

Function that gets called by the kernel to ask the driver to initialize and prepare everything about the device and make it ready to work

```c
	priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;


	adap = &priv->adap;
	bit_data = &priv->bit_data;
	pdata = &priv->pdata;
```
<!--
  allocating mem for driver specific information
Devm functions basically allocates memory in a order resources are allocated and deallocates those memory automatically in a reverse order. The most commonly used allocation flag is GFP_KERNEL means that allocation is performed on behalf of a process running in the kernel space. This means that the calling function is executing a system call on behalf of a process.

priv private data needed for managing the I2C GPIO driver. The primary purpose of these assignments is to make it easier to reference these members without repeatedly dereferencing priv
-->

---

#
```c
	if (fwnode) {
		i2c_gpio_get_properties(dev, pdata);
	} else {
		/*
		 * If all platform data settings are zero it is OK
		 * to not provide any platform data from the board.
		 */
		if (dev_get_platdata(dev))
			memcpy(pdata, dev_get_platdata(dev), sizeof(*pdata));
	}
```
<!--
if firmware node available `i2c_gpio_get_properties` is called to retrieve and set the platform-specific data (pdata) for the I2C GPIO driver.
if no fwnode and platform data associated with the device represented by the dev no platform data provided from the board
if device platform data available and no fw node device specific platform data is provided
-->
---

#

```
if (pdata->sda/scl_is_open_drain || pdata->sda/scl_has_no_pullup)
	gflags = GPIOD_OUT_HIGH;
	else
		gflags = GPIOD_OUT_HIGH_OPEN_DRAIN;
	priv->sda/scl = i2c_gpio_get_desc(dev, "sda/scl", 1, gflags);
	if (IS_ERR(priv->sda/scl))
		return PTR_ERR(priv->sda/scl)

	if (gpiod_cansleep(priv->sda) || gpiod_cansleep(priv->scl))
		dev_warn(dev, "Slow GPIO pins might wreak havoc into I2C/SMBus bus timing");
	else
		bit_data->can_do_atomic = true;

	bit_data->setsda = i2c_gpio_setsda_val;
	bit_data->setscl = i2c_gpio_setscl_val;

	if (!pdata->scl_is_output_only)
		bit_data->getscl = i2c_gpio_getscl;
	if (!pdata->sda_is_output_only)
		bit_data->getsda = i2c_gpio_getsda;
```
<!--
Configure i2c atomicity based on gpiod is atomic or not
-->
---

```c
	if (pdata->udelay)
		bit_data->udelay = pdata->udelay;
	else if (pdata->scl_is_output_only)
		bit_data->udelay = 50;			/* 10 kHz */
	else
		bit_data->udelay = 5;			/* 100 kHz */

	if (pdata->timeout)
		bit_data->timeout = pdata->timeout;
	else
		bit_data->timeout = HZ / 10;		/* 100 ms */

	bit_data->data = priv;
```
<!--
scl output only means slower communication
The line ```bit_data->data = priv;``` is crucial for linking the specific private data structure with the bit-banging algorithm's configuration structure. This allows the algorithm to access any necessary additional data or state information specific to this particular instance of the I2C adapter.
i2c_adapter is the structure used to identify a physical i2c bus along with the access algorithms necessary to access it.

If fwnode is present, the adapter's name is set to the device's name.
If fwnode is absent, the adapter's name is generated using the platform device's ID to ensure it is unique.
-->
---

#

  Configuring and initializing an I2C adapter in the Linux kernel, ensuring that the adapter is properly linked with its bit-banging algorithm data, classified correctly, associated with its parent device, and linked with the appropriate firmware node. In this setup, the parent device is the platform device representing the hardware providing the GPIOs for the I2C bit-banging bus. This association helps manage the device hierarchy and resource dependencies in the kernel.

---

#

  the pdev->id provides a unique identifier for the I2C bus, and the adapter is registered with the I2C core, making the bus available for use by I2C client devices in the system. If registration fails, the error is returned and can be handled appropriately.

  The platform_set_drvdata function is used in the Linux kernel to associate driver-specific data with a platform device. This association allows the driver to retrieve its data structure later, typically in other callback functions, using the platform device structure.

---

#

  Providing information on SDA and SCL lines used and if clock stretching is used
  (clock stretching)

---

# i2c_gpio_remove

Cleanup resources

```c
 static void i2c_gpio_remove(struct platform_device *pdev)
{
	struct i2c_gpio_private_data *priv;
	struct i2c_adapter *adap;

	priv = platform_get_drvdata(pdev);
	adap = &priv->adap;

	i2c_del_adapter(adap);
}
```
<!--
Cleanup is performed by
1. Retrieving priv through platform_get_drvdata(pdev)
2. Deleting the adapter
-->
---

# i2c_gpio_exit

```c
static void __exit i2c_gpio_exit(void)
{
	platform_driver_unregister(&i2c_gpio_driver);
}
```
<!--
The __exit macro is used by the kernel to optimize code placement — it hints that this function will only be used at module unload time (and can be discarded in a built-in kernel).
Tells the kernel to clean up resources, and stop matching devices to it.
-->
