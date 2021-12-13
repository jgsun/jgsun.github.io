---
layout: post
title: "Linux GPIO driver"
category: system
tags: GPIO gpiolib libgpiod gpio-toos
author: jgsun
---

* content
{:toc}

[TOC]
# Overview
下图来自文章 [Linux kernel GPIO user space interface](https://embeddedbits.org/new-linux-kernel-gpio-user-space-interface/) ，可概括这篇文章的内容。本文介绍 Linux GPIO driver的初始化，用户态访问 GPIO 和内核态访问 GPIO。

![image](/images/posts/gpio/gpio_overview.png)




















# GPIO init
![image](/images/posts/gpio/gpio_init.png)





## DTS
```
        gpio0: gpio@GPIO0_FULL_ADDR {
            compatible = "snps,dw-apb-gpio";
            reg = <GPIO0_REGS GPIO_REGS_SIZE>;
            #address-cells = <1>;
            #size-cells = <0>;
            gpio0p0: gpio-controller@0 {
                compatible = "snps,dw-apb-gpio-port";
                gpio-controller;
                #gpio-cells = <2>;
                snps,nr-gpios = <8>;
                snps,gpio-base = <0>;
                reg = <0>;
                interrupt-controller;
                #interrupt-cells = <2>;
                interrupt-parent = <&gic>;
                interrupts = <GIC_SPI GPIO_0_SPI IRQ_TYPE_LEVEL_HIGH>;
            };
        };
```
## core_initcall(gpiolib_dev_init)
```
gpiolib_dev_init
    bus_register(&gpio_bus_type)
    alloc_chrdev_region(&gpio_devt, 0, GPIO_DEV_MAX, "gpiochip")
    gpiolib_initialized = true
    gpiochip_setup_devs() //traverse gpio_devices 
        gpiochip_setup_dev(gdev)
            cdev_add(&gdev->chrdev, gdev->dev.devt, 1)
            device_add(&gdev->dev)
            gpiochip_sysfs_register(gdev)
                device_create_with_groups(&gpio_class, parent
```

## postcore_initcall(gpiolib_sysfs_init)
```
class_register(&gpio_class)
gpiochip_sysfs_register
```
导出 sysfs 文件 /sys/class/gpio/export 和 /sys/class/gpio/unexport 


## subsys_initcall(gpiolib_debugfs_init)
```
debugfs_create_file("gpio",  // create "/sys/kernel/debug/gpio"
```
## module_platform_driver(bgpio_driver)
Generic driver for memory-mapped GPIO controllers.
## module_platform_driver(dwapb_gpio_driver)
```
dwapb_gpio_probe
    dwapb_gpio_add_port
        bgpio_init
        dwapb_configure_irqs(gpio, port, pp)
            irq_domain_create_linear
            irq_alloc_domain_generic_chips
            irq_get_domain_generic_chip
            irq_set_chained_handler_and_data or
            devm_request_irq or irq_set_chained_handler_and_data
            irq_create_mapping(gpio->domain, hwirq)
            port->gc.to_irq = dwapb_gpio_to_irq
        gpiochip_add_data(&port->gc, port)
            gpiodev_add_to_list //add to gpio_devices
            gpiochip_set_desc_names
            gpiochip_irqchip_init_valid_mask
            of_gpiochip_add
            gpiochip_setup_dev
```

## user drivers
### UIO platform driver
```
uio_irq_platform_probe
    uio_irq_parse
    uio_register_device
```
### builtin_platform_driver(gpio_clk_driver)
Gpio controlled clock implementation
### module_platform_driver(mdio_gpio_driver)
GPIO based MDIO bitbang driver.
### module_platform_driver(gpio_led_driver)
LEDs driver for GPIOs
## user interface
### Char device

    ~ # ls /dev/gpiochip* -l
    crw-rw----    1 root     root      254,   0 Jan  1  1970 /dev/gpiochip0
    crw-rw----    1 root     root      254,   1 Jan  1  1970 /dev/gpiochip1
    crw-rw----    1 root     root      254,   2 Jan  1  1970 /dev/gpiochip2

### sysfs

    /sys/class/gpio # ls -l
    --w-------    1 root     root          4096 Dec  3 19:07 export
    lrwxrwxrwx    1 root     root             0 Dec  3 19:07 gpiochip0 -> ../../devices/platform/axi/4f200000.gpio/gpio/gpiochip0
    lrwxrwxrwx    1 root     root             0 Dec  3 19:07 gpiochip16 -> ../../devices/platform/axi/4f600000.gpio/gpio/gpiochip16
    lrwxrwxrwx    1 root     root             0 Dec  3 19:07 gpiochip8 -> ../../devices/platform/axi/4f400000.gpio/gpio/gpiochip8
    --w-------    1 root     root          4096 Dec  3 19:07 unexport



# 内核态访问 GPIO
![image](/images/posts/gpio/gpio_kernel.png)




# 用户态访问 GPIO
![image](/images/posts/gpio/gpio_user.png)

## libgpiod 
buildroot 选择 package libgpiod 和 gpio tools：
![image](/images/posts/gpio/br_libgpiod.png)

libgpiod - C library and tools for interacting with the linux GPIO
            character device (gpiod stands for GPIO device)

(https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git/)

Since linux 4.8 the GPIO sysfs interface is deprecated. User space should use
the character device instead. This library encapsulates the ioctl calls and
data structures behind a straightforward API.

## gpio tools

There are currently six command-line tools available:

* gpiodetect - list all gpiochips present on the system, their names, labels
               and number of GPIO lines

* gpioinfo   - list all lines of specified gpiochips, their names, consumers,
               direction, active state and additional flags

* gpioget    - read values of specified GPIO lines

* gpioset    - set values of specified GPIO lines, potentially keep the lines
               exported and wait until timeout, user input or signal

* gpiofind   - find the gpiochip name and line offset given the line name

* gpiomon    - wait for events on GPIO lines, specify which events to watch,
               how many events to process before exiting or if the events
               should be reported to the console

# 参考文档
* [Linux kernel GPIO user space interface](https://embeddedbits.org/new-linux-kernel-gpio-user-space-interface/)
* [How to control a GPIO in userspace](https://wiki.stmicroelectronics.cn/stm32mpu/wiki/How_to_control_a_GPIO_in_userspace)
* [How to control a GPIO in kernel space](https://wiki.stmicroelectronics.cn/stm32mpu/wiki/How_to_control_a_GPIO_in_kernel_space)
