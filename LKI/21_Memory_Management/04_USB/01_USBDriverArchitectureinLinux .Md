🔧 USB Driver Architecture in Linux (In-Depth for Interview)
=====================================================================================================

🧠 What Interviewers Expect You to Cover:
--------------------------------------------------------------------
Understanding of how USB works in Linux
Roles of USB Core, Host Controller Driver (HCD), and Device Drivers
Real kernel paths and code flow
How enumeration, probing, and URBs work
Debugging USB issues at the kernel level

🧱 Layered Architecture:
=====================================================================================================

+--------------------------+       User Space
| libusb / udev / sysfs    |
| user-space applications  |
+--------------------------+       Kernel Space
| USB Device Driver        |
| (e.g., usb-storage.c)    |
+--------------------------+
| USB Core Subsystem       |
| (usbcore.ko, hub.c)      |
+--------------------------+
| Host Controller Driver   |
| (ehci-hcd, ohci-hcd)     |
+--------------------------+
| Hardware (USB Controller)|
+--------------------------+


+-----------------------------+
|   User Space Applications  |
|  (libusb, udev, etc.)      |
+-----------------------------+
|     USB Class Drivers      |  ← Example: usb-storage, usbhid
+-----------------------------+
|       USB Core             |  ← Common APIs, enumeration logic
+-----------------------------+
| Host Controller Driver (HCD) | ← e.g., EHCI, OHCI, XHCI
+-----------------------------+
|     USB Host Controller    |
+-----------------------------+
|         USB Bus            |
+-----------------------------+
|      USB Device (Peripheral)|
+-----------------------------+



🔹 1. USB Host Controller Driver (HCD)
-----------------------------------------
This layer directly interacts with the USB host controller hardware.

✅ Examples:
            ehci-hcd.c → USB 2.0
            xhci-hcd.c → USB 3.x
            ohci-hcd.c → USB 1.1

✅ Responsibilities:
-----------------------------
Scheduling transfers (interrupt, bulk, isochronous, control)
Managing root hubs
Handling DMA or PIO access to endpoints
Issuing URBs (USB Request Blocks) to devices
Communicating with hardware using MMIO, interrupts

🔹 2. USB Core Subsystem
====================================================================================================
Located at drivers/usb/core/, this acts as the bridge between the HCD and device drivers.

📂 Key Files:
                usb.c, hub.c, driver.c, file.c

✅ Responsibilities:
    Enumeration and device descriptor parsing
    Managing USB device hierarchy (topology)
    Matching devices to drivers via id_table
    Managing device reference count and power states
    
    Registration APIs for drivers:
                                    usb_register()
                                    usb_deregister()

🔹 3. USB Device Drivers
=====================================================================================================
These drivers manage specific types of USB devices, such as mass storage, keyboard, camera, etc.

✅ Location:
            drivers/usb/ or drivers/usb/class/

✅ Structure:

            struct usb_driver {
                const char *name;
                const struct usb_device_id *id_table;
                int (*probe)(struct usb_interface *, const struct usb_device_id *);
                void (*disconnect)(struct usb_interface *);
                ...
            };

✅ Registration:
                usb_register(&my_usb_driver);

✅ Probe Function:
                    Called when a device matches the id_table.
                    Initializes the device, allocates buffers, submits URBs.



🔹 4. URBs (USB Request Blocks)
====================================================================================================
URB is a structure used to send data to/from a USB device asynchronously.

✅ Fields:

struct urb {
    void *transfer_buffer;
    int transfer_buffer_length;
    unsigned int pipe;
    ...
};


✅ Submission & Handling:

usb_submit_urb(urb, GFP_KERNEL);
usb_kill_urb(urb); // to cancel

URBs are completed via callbacks in interrupt context.

🔹 5. Hotplugging and Enumeration Process
====================================================================================================

Device Inserted: 
                    Detected by HCD via interrupt.

Device Enumeration:
                    USB core assigns an address.
                    Reads device descriptors.
                    Loads appropriate driver using idVendor and idProduct.

Driver Matching:
                    Core matches device to usb_device_id[].
                    Calls the probe() of the matched driver.

URB Submission:
                    Driver sends/receives data using usb_submit_urb.

Device Removal:
                    Core notifies driver by calling disconnect().


🔍 Debugging & Tools
====================
✅ Sysfs:
            /sys/bus/usb/devices/
            /sys/class/usb_device/

✅ Tools:
            lsusb, usb-devices
            dmesg, usbmon
            usbcore.autosuspend, usbcore.nousb


📦 Sample Code Snippet
====================================================================================================
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/usb.h>

/*
 * Table of USB device IDs this driver supports.
 * Format: USB_DEVICE(Vendor ID, Product ID)
 * This helps the USB core match the driver to connected devices.
 */
static struct usb_device_id my_table[] = {
    { USB_DEVICE(0x1234, 0x5678) }, // Example Vendor/Product ID
    {}  // Terminating entry
};
MODULE_DEVICE_TABLE(usb, my_table); // Required for hotplug/autoload

/*
 * Probe function - called when a matching USB device is connected.
 * @iface: USB interface descriptor
 * @id: Pointer to the matched device ID from the table
 */
static int my_probe(struct usb_interface *iface, const struct usb_device_id *id)
{
    pr_info("My USB device connected\n");

    // Device initialization and resource allocation can go here

    return 0; // Success
}

/*
 * Disconnect function - called when the device is removed.
 * @iface: USB interface descriptor
 */
static void my_disconnect(struct usb_interface *iface)
{
    pr_info("My USB device removed\n");

    // Cleanup and resource deallocation can go here
}

/*
 * USB driver definition structure
 */
static struct usb_driver my_usb_driver = {
    .name       = "my_usb",        // Driver name
    .id_table   = my_table,        // Supported device ID table
    .probe      = my_probe,        // Called on device connect
    .disconnect = my_disconnect,   // Called on device disconnect
};

/*
 * Register the USB driver with the kernel
 * module_usb_driver is a helper macro that registers the driver
 * in init and unregisters in exit function.
 */
module_usb_driver(my_usb_driver);

MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Sample USB driver with probe/disconnect");
MODULE_LICENSE("GPL");


✅ Key Additions:
#include headers required for kernel module and USB APIs.
MODULE_DEVICE_TABLE() so udev/modprobe can auto-load your module.
MODULE_* macros for metadata.
Clear and concise comments per function and structure.


