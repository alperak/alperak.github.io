---
title: "OTA (Over-The-Air) Updates"
date: 2024-07-03 13:35:47 +0300
categories: [Embedded Linux, OTA]
tags: [embedded-linux, ota, firmware]
---

![Line-art icon of a gear inside a circular arrow, captioned UPDATE](/assets/img/posts/ota-over-the-air-updates/1.jpg){: width="320" height="320" }

#### **What is OTA (Over-The-Air) update?**

Over-the-air (OTA) means of remotely updating the firmware, software, operating system of a device over the internet connection. OTA is particularly common in several fields such as:

✒ Embedded Systems

✒ Internet-of-Things (IoT) Devices

✒ Smartphones

✒ Automotive Industry

OTA typically categorized into two types:

✒ Software Over-The-Air (SOTA): It primarily involves updating the device’s user-space software applications.

✒ Firmware Over-The-Air (FOTA): Focused on updating low-level firmware components such as the bootloader, kernel, device drivers, root filesystem.

#### **Why are OTA updates important and what are the benefits?**

OTA updates are vital for various reasons:

✒ Security

Security vulnerabilities are constantly being discovered in software, including operating systems and applications. OTA updates allow to quickly push out patches to fix these vulnerabilities, helping to keep devices secure and protecting users from potential threats.

✒ Feature Enhancements

New features and improvements can be introduced to existing products, enhancing the user experience and extending the lifespan of devices through OTA updates.

✒ Bug Fixes

Software bugs are unavoidable, even with extensive testing. OTA updates allow developers to address bugs discovered after a product’s release and distribute fixes to users efficiently.

✒ Cost-Effectiveness

OTA updates can save manufacturers money by reducing the need for physical recalls or service center visits to address issues or deliver updates. They also reduce the support burden by allowing for remote fixes and updates.

✒ Convenience

OTA updates allow users to update their devices without needing to connect them to a device or take them to a service center. This convenience encourages users to keep their devices up to date with the latest features, bug fixes and security patches.

✒ User Experience

Keeping devices up to date ensures that users have access to the latest features and improvements, providing a better overall experience with the product.

✒ Future-Proofing

OTA updates can also future-proof devices by allowing them to adapt to changing technologies and standards over time. This helps extend the useful life of devices and ensures that they remain relevant in a rapidly evolving technological landscape.

Overall, OTA updates are vital for maintaining the security, performance, and usability of connected devices, providing benefits for both manufacturers and users alike.

#### **Several important considerations when designing OTA updates**

✒ Security

Encryption and authentication mechanisms, as well as the use of secure protocols are important. Secure communications are necessary for a secure update process.

✒ Reliability

OTA updates must be reliable, ensuring that updates are delivered and installed correctly without compromising device functionality.

✒ Fail-safe

In case of failed updates, a fallback or rollback mechanism should be in place to revert to the previous version and maintain device functionality. Logic that can be designed in bootloaders and hardware watchdogs can help with this. We never want devices to be left in an unusable state.

✒ Scalability

The OTA update system should be designed to scale efficiently with the growing number of deployed devices, ensuring that updates can be delivered reliably and promptly to all devices in the field. It is important to think about these questions:

- How will firmware managed for many devices and device types?
- How will firmware update deployed to considerable number of devices?
- What will be the update frequency? That may be affect the choice of update method.

✒ Device Management

- How will devices identified that are in need of an update?
- How will devices detected that have failed updates and require recovery?
- How will the status of devices tracked?

✒ Resource

The storage size of the device should be taken into consideration because it is a factor in determining the update mechanism.

✒ Network Types and Data Transfer

Considering the often limited bandwidth in IoT deployments, optimizing OTA updates to minimize data transfer while ensuring timely delivery requires careful consideration of factors such as bandwidth, latency, and network reliability, including how updates are delivered across various network types like Wi-Fi, LTE and understanding how these network characteristics can impact update efficiency.,

#### **OTA update strategies**

#### ✒ A/B Update (Dual Copy)

One of the safest ways to perform OTA updates. Also, this approach usually preferred for Android and Embedded Linux devices. A/B (Dual Copy) update is a strategy where two copies of the OS are maintained simultaneously on devices while a new firmware update is sent. While one copy is being updated, the other copy ensures the current system continues to operate. Once the update is completed, the device automatically switches to the updated copy.

A/B typically works like:

- Changes are made and new firmware is created.
- An OTA job is created that includes new firmware.
- The device runs from active copy and let’s call this partition A which is currently active.
- Device performs regular check in with OTA server and is notified of the availability of an update.
- The device performs integrity checks (e.g. checksums, hashes) to ensure the downloaded firmware is complete and not corrupted. Moreover, the package is also authenticated to ensure it has not been tampered with and is from a trusted source.
- After the new image has been successfully downloaded and It is written to partition B which is currently inactive.
- The system verifies the integrity of the new image to ensure that it has been successfully written to partition B.
- Bootloader configuration is updated to mark partition B as the active copy.
- Device is rebooted and boots the newly written image from partition B.
- If boot fails, the device will fallback to A partition and remains usable. In the event of a failure, fallbacks typically utilize the bootloader’s bootcount, bootlimit mechanisms, hardware watchdog.

This approach ensures that users can continue to use their devices uninterrupted during the update process and minimizes the risk of update failures or bricking the device. Also, keeping the previous old system image gives us the possibility to rollback to the system from before the update, in case the updated system crashes.

The most important drawback to consider is that twice as much storage space is needed to have two copies compared to the Single Copy scenario. In addition, downloading the full system image consumes more data than a delta update, which can be a issue for limited data plans.

#### ✒ Single Copy Update

This strategy based on a “Recovery/Rescue OS” which is consists of minimal kernel and rootfs which is just enough features to allow the device to boot and perform update. The most basic approach uses a simple partition layout like:

```text
| Bootloader | Recovery/Rescue OS (minimal kernel and rootfs) | Regular OS | Data |
```

This strategy usually follows these steps:

- Changes are made and new firmware is created.
- An OTA job is created that includes new firmware.
- The device runs from “Regular OS”.
- Device performs regular check in with OTA server and is notified of the availability of an update.
- The device performs integrity checks (e.g. checksums, hashes) to ensure the downloaded firmware is complete and not corrupted. Moreover, the package is also authenticated to ensure it has not been tampered with and is from a trusted source.
- New image is downloaded to the partition which is called “Data Partition”.
- Bootloader environment variable or an external GPIO should be adjusted to tell the bootloader to enter update mode.
- Device reboots into the “Recovery/Rescue OS” and that is load the recovery ramdisk image and boots using RAM.
- The new image is pulled from the “Data Partition” and used to update the system.
- Device is rebooted normally and boots the newly updated “Regular OS”.

If the device has limited storage space, this is a preferable strategy over A/B (Dual Copy). If space is not an issue, then it is recommended to use the A/B (Dual Copy) strategy because it will result in less down-time. Also, this strategy requires two reboots, one into the “Recovery/Rescue OS” and then another back into the “Regular OS”. The A/B (Dual Copy) only requires one reboot as update can be performed at any time.

One important thing to keep in mind is that this doesn’t guarantee a fallback! In the meantime, it can be guaranteed that the system goes automatically in update mode when the productivity software is not found or corrupted, as well as when the update process is interrupted for some reason. In fact, it is possible to consider the update procedure as a transaction and only after the successful update the new software is set as “bootable”. Also, the bootloader’s environment setting variables can be managed to indicate the start and end of a process and that the storage contains valid software.

#### ✒ Delta Update

Delta update involves sending only the differences between the currently running firmware on the device and the new firmware, rather than sending the entire new firmware image. This means that new firmware is largely made up of the old firmware with a few changes. It is very common that a device will be updated to a version that is similar to the running one but add new features and some bug fixes. Specifically in case of just fixes, the new version is pretty much equal as the original one. In addition, Delta updates can be used in conjunction with both A/B (Dual Copy) and Single Copy update strategies.

A basic outline of the steps of the Delta Update process might include the following steps:

✒ Changes are made and new firmware is created.

✒ The delta encoding (data differencing) algorithm is used to find the difference between firmwares.

There are several algorithms for delta encoding such as BsDiff, XDelta, librsync, casync, zchunk. Without going into too much detail about these algorithms, I will just mention a few things to consider when making a choice.

- File Type and Size

Some tools perform better on binary files while others might be optimized for text files. Also, worth to consider whether the tool can handle large files efficiently.

- Delta Size Efficiency

How well does the tool compress the delta files? Smaller deltas mean less data to transfer.

- Resource Usage

Some tools require loading entire files into memory, which might be a limitation for embedded systems with constrained resources.

- Integration Complexity

How straightforward is it to integrate the tool into your existing build and deployment pipeline? Also, does the tool provide robust APIs or libraries that can be easily used within your environment?

- Licensing

It should be checked whether there are any restrictions or requirements for commercial use.

- Security Features

Does the tool provide mechanisms to ensure the integrity of the delta and patched files? Besides, features like digital signatures and checksums to verify the authenticity of updates may useful.

- Supported Platforms

Ensure the tool supports the operating systems and architectures you are targeting. If you need to support multiple platforms, check the compatibility across these environments.

- Community and Support

Documentation is essential for efficient implementation and troubleshooting. Furthermore, strong community can provide assistance and contribute to the tool’s development.

- Specific Features

If I give an example of this, there are two things that come to my mind. Firstly, tools like zchunk offer advanced chunking mechanisms. Consider if this is a requirement for your use case. Secondly, tools like casync may introduce additional network overhead due to the large number of small chunk requests.

✒ The delta file created by one of the libraries that supports delta encoding. This package contains all the necessary instructions and data to update the old firmware to the new version.

✒ An OTA job is created that includes the delta package.

✒ Device performs regular check in with OTA servers and is notified of the availability of an update.

✒ Before applying the patch, the device performs integrity checks (e.g. checksums, hashes) to ensure the downloaded delta package is complete and not corrupted. Moreover, the package is also authenticated to ensure it has not been tampered with and is from a trusted source.

✒ For safety, the device may create a backup of the current firmware before applying the patch. This allows recovery in case the update fails.

✒ The device uses the patching library or tool to apply the delta package to the current firmware.

The patch library is responsible for applying the delta file to the old version to reconstruct the new version. This process is known as patching. Also, many delta libraries or tools provide both the delta encoding and the patching functionality.

✒ The patching library or tool reads the delta package and reconstructing the new firmware image by combining the old firmware(the existing firmware) with the changes specified in the delta package. The device might temporarily store the reconstructed firmware in a different location before replacing the old firmware to ensure atomic updates.

✒ After applying the patch, the device verifies that the new firmware has been correctly constructed. This can involve checking file integrity and ensuring that all changes have been applied as expected.

✒ The device may be rebooted to load the new firmware. The bootloader ensures the new firmware is correctly loaded. Additional verification steps might be performed during boot to ensure the firmware is functioning correctly.

✒ Once the device is running the new firmware, additional checks ensure that the device is operating as expected. If any issues are detected, the device may revert to the previous firmware using the backup.

If we talk about advantages, we can say that one of the most important advantages is the image size. Delta images are often smaller than full system images. Furthermore, Delta updates reduce the update time, data transfer and power consumption. Also, OTA updates become possible on low bandwidth networks such as LoRaWAN and NB-IoT.

On the other side, some things to keep in mind when dealing with Delta updates:

- Generating and applying delta files can be more complex than full updates.
- Major changes can result in delta files nearly as large as full updates.
- Delta updates rely on the exact base version, complicating version management.

#### ✒ Container-based Update

I initially intended to avoid from talk about this methodology, as I lacked direct experience with it. But then I decided that it might be beneficial to giving a brief overview of what it is. I thought this would help to explore the topic further.

A container-based OTA update is a method of delivering software updates to devices using containerization technology. Containers, like those managed by Docker, encapsulate an application and its dependencies into a single package that can be deployed consistently across different environments.

#### ✒ **OTA Update and Deployment Management Platforms**

There are multiple platforms that support OTA update and deployment management. I will share the ones I see the most, there may be more but these are the ones I have seen.

Updater:

- RAUC
- SWUpdate
- Mender
- OSTree
- Swupd
- Balena
- AWS IoT

Management:

- Eclipse hawkBit
- Mender
- Balena
- AWS IoT

#### ✒ **Conclusion**

Each strategy has its advantages and disadvantages, and the choice depends on factors such as device capabilities, network bandwidth, update frequency, and the criticality of maintaining system availability during the update process. It is also worth bearing in mind that there is no exact steps! As I mentioned in the article, many situation can change the implementation and steps of the strategies, but I hope the steps I have given above will help you at least get a basic knowledge of OTA and will help you to grasp the topic even more.I would like to add one last thing before I end this article, if there is anything that can be improved, anything that you see missing or wrong, please don’t hesitate to get in touch.

Finally, in future articles I will provide examples for each of the update strategies and look at them in detail, starting with A/B (Dual Copy) and continuing with Single Copy and Delta Update.

**References**

- <https://sbabic.github.io/swupdate/swupdate.html>
- <https://rauc.readthedocs.io/en/latest/>
- <https://docs.mender.io/>
- <https://www.thegoodpenguin.co.uk/blog/>
- <https://interrupt.memfault.com/blog/>
