---
layout: post
title:  "Debugging Linux device trees"
date:   2025-03-21 00:40:00 +0300
tags: Linux
---
Back with another debugging anecdote that kept me employed for a whole long day. This time it was not my usual platform, Windows but Linux. More specifically, an arm64 based board named Rock-5T running Ubuntu 24.04 Server Edition would'nt pick up an ethernet port and a Wi-Fi/Bluetooth adapter. The board was running an image intended for another very similiar board named Rock-5B-Plus and was picking up peripherals that were common between both. The previous board didn't have two ethernet ports and an intel Wifi/Bluetooth card and the kernel on this board wasn't picking it up either. After seeing the output from lshw and lspci and trying all the usual methods of fixing this problem like like checking and installing the drivers, looking at dmesg logs and restarting systemd services, it wasn't picking up the devices at all. I looked up on Google and immediately found the cause. Arm based devices such as this one don't have a standard for detecting and initializing hardware like ACPI on x86. Therefore they rely on device trees that provide them with the defination of different hardware devices and their configurations. Now this might seem a super basic concept to people that often work with embedded linux, but I don't much experience in this field. My experience with embedded linux has only involved cross-compiling a few linux utilities to my home router for some forgotten reason. So, after finding out that the problem lies with the device tree the kernel and u-boot were loading, I set out to solve this issue. First I searched the device for all the device trees it had and found out that while it had the Rock-5B-Plus device-tree it was lacking the device-tree for this exact board, i.e Rock-5T. So I downloaded the default linux iamge that came with this board and extracted the dtb file from it using 7z and then copied it back to the system. Then I looked for ways to load it into the device and I found out that this device was using extlinux.conf to specify a fdtdir from which u-boot automatically picked up the correct fdt based on the board model set at u-boot compile time. I changed it to a simple fdt instead of fdtdir and specified the path to the new dtb. I rebooted and boom, it now had two eth ports and wifi/bluetooth working. But this was a short lived joy as now the HDMI had stopped working. I looked at dmesg and the message indicated that the kernel was unable to register a clock for the rockchip-hdptx-phy-hdmi fed70000.hdmiphy node. Now this is where things get intresting, the HDMI was working with the wrong device tree but it refused to work with the correct device tree. To find out the differences between both device trees, I decompiled them using dtc to get a textual representation of the dtbs. Then I loaded both of the device tree source files into a really useful online site (diffchecker.com) that enabled me to find the differences between the 11k lines per each file. I found a lot of differences but none of them seemed to be the cause for the failure of the HDMI clock. I also found out that I was unfamilliar with the structure of a dts file and went on to Google to better educate myself regarding things like phandles, phys, regulators, clocks, rockchip-grfs and other properties in a dts. Now things started to make a little more sense. There was a difference in syntax between how clocks were declared for two hdmiphy nodes in the dts for rock-5t and rock-5b-plus:
```
Rock-5T:
	hdmiphy@fed60000 {
		compatible = "rockchip,rk3588-hdptx-phy-hdmi";
		reg = <0x00 0xfed60000 0x00 0x2000>;
		clocks = <0x02 0x2b5 0x02 0x267>;
		clock-names = "ref\0apb";
		resets = <0x02 0x48e 0x02 0x485 0x02 0xc003b 0x02 0xc003c 0x02 0xc003d 0x02 0x48c 0x02 0x48d>;
		reset-names = "phy\0apb\0init\0cmn\0lane\0ropll\0lcpll";
		rockchip,grf = <0x17c>;
		#phy-cells = <0x00>;
		status = "okay";
		phandle = <0xf5>;

		clk-port {
			#clock-cells = <0x00>;
			status = "okay";
			phandle = <0x33>;
		};
	};

	hdmiphy@fed70000 {
		compatible = "rockchip,rk3588-hdptx-phy-hdmi";
		reg = <0x00 0xfed70000 0x00 0x2000>;
		clocks = <0x02 0x2b5 0x02 0x268>;
		clock-names = "ref\0apb";
		resets = <0x02 0x491 0x02 0x486 0x02 0xc003f 0x02 0xc0040 0x02 0xc0041 0x02 0x48f 0x02 0x490>;
		reset-names = "phy\0apb\0init\0cmn\0lane\0ropll\0lcpll";
		rockchip,grf = <0x1b8>;
		#phy-cells = <0x00>;
		status = "okay";
		phandle = <0x19f>;

		clk-port {
			#clock-cells = <0x00>;
			status = "okay";
			phandle = <0x34>;
		};
	};
```
Rock-5B-Plus:
```
    hdmiphy@fed60000 {
		compatible = "rockchip,rk3588-hdptx-phy-hdmi";
		reg = <0x00 0xfed60000 0x00 0x2000>;
		clocks = <0x02 0x2b5 0x02 0x267>;
		clock-names = "ref\0apb";
		clock-output-names = "clk_hdmiphy_pixel0";
		#clock-cells = <0x00>;
		resets = <0x02 0x48e 0x02 0x485 0x02 0xc003b 0x02 0xc003c 0x02 0xc003d 0x02 0x48c 0x02 0x48d>;
		reset-names = "phy\0apb\0init\0cmn\0lane\0ropll\0lcpll";
		rockchip,grf = <0x17c>;
		#phy-cells = <0x00>;
		status = "okay";
		phandle = <0x33>;
	};

    	hdmiphy@fed70000 {
		compatible = "rockchip,rk3588-hdptx-phy-hdmi";
		reg = <0x00 0xfed70000 0x00 0x2000>;
		clocks = <0x02 0x2b5 0x02 0x268>;
		clock-names = "ref\0apb";
		clock-output-names = "clk_hdmiphy_pixel1";
		#clock-cells = <0x00>;
		resets = <0x02 0x491 0x02 0x486 0x02 0xc003f 0x02 0xc0040 0x02 0xc0041 0x02 0x48f 0x02 0x490>;
		reset-names = "phy\0apb\0init\0cmn\0lane\0ropll\0lcpll";
		rockchip,grf = <0x1b8>;
		#phy-cells = <0x00>;
		status = "okay";
		phandle = <0x34>;
	};
```
As you can see, the dts for Rock-5T was defining the clocks under a sub node named clk-port and Rock-5B-Plus was directly defining them in the node defination for hdmiphy. Now, I looked at the dts to see how and where the clocks and hdmiphys where being refrenced and found out that if I changed the defination of the hdmiphys to follow the Rock-5B-Plus synntax, I would have to edit the following nodes as well:
```
	hdmi@fde80000 {
		compatible = "rockchip,rk3588-dw-hdmi";
		reg = <0x00 0xfde80000 0x00 0x10000 0x00 0xfde90000 0x00 0x10000>;
		interrupts = <0x00 0xa9 0x04 0x00 0xaa 0x04 0x00 0xab 0x04 0x00 0xac 0x04 0x00 0x168 0x04>;
		clocks = <0x02 0x221 0x02 0x265 0x02 0x222 0x02 0x223 0x02 0x246 0x02 0x274 0x02 0x275 0x02 0x276 0x02 0x277 0x05 0x33>;
		clock-names = "pclk\0hpd\0earc\0hdmitx_ref\0aud\0dclk_vp0\0dclk_vp1\0dclk_vp2\0dclk_vp3\0hclk_vo1\0link_clk";
		resets = <0x02 0x3d0 0x02 0x49c>;
		reset-names = "ref\0hdp";
		power-domains = <0x5a 0x1a>;
		pinctrl-names = "default";
		pinctrl-0 = <0xf1 0xf2 0xf3 0xf4>;
		reg-io-width = <0x04>;
		rockchip,grf = <0xbf>;
		rockchip,vo1_grf = <0xd0>;
		phys = <0x33>;                             # changed from <0xf5>
		phy-names = "hdmi";
		#sound-dai-cells = <0x00>;
		status = "okay";
		cec-enable = "true";
		enable-gpios = <0xf6 0x09 0x00>;
		phandle = <0x1c6>;

		ports {
			#address-cells = <0x01>;
			#size-cells = <0x00>;

			port@0 {
				reg = <0x00>;
				#address-cells = <0x01>;
				#size-cells = <0x00>;
				phandle = <0x272>;

				endpoint@0 {
					reg = <0x00>;
					remote-endpoint = <0x39>;
					status = "okay";
					phandle = <0xd4>;
				};

				endpoint@1 {
					reg = <0x01>;
					remote-endpoint = <0xf7>;
					status = "disabled";
					phandle = <0xda>;
				};

				endpoint@2 {
					reg = <0x02>;
					remote-endpoint = <0xf8>;
					status = "disabled";
					phandle = <0xe0>;
				};
			};
		};
	};

    hdmi@fdea0000 {
		compatible = "rockchip,rk3588-dw-hdmi";
		reg = <0x00 0xfdea0000 0x00 0x10000 0x00 0xfdeb0000 0x00 0x10000>;
		interrupts = <0x00 0xad 0x04 0x00 0xae 0x04 0x00 0xaf 0x04 0x00 0xb0 0x04 0x00 0x169 0x04>;
		clocks = <0x02 0x224 0x02 0x266 0x02 0x225 0x02 0x226 0x02 0x24c 0x02 0x274 0x02 0x275 0x02 0x276 0x02 0x277 0x05 0x34>;
		clock-names = "pclk\0hpd\0earc\0hdmitx_ref\0aud\0dclk_vp0\0dclk_vp1\0dclk_vp2\0dclk_vp3\0hclk_vo1\0link_clk";
		resets = <0x02 0x3d7 0x02 0x49d>;
		reset-names = "ref\0hdp";
		power-domains = <0x5a 0x1a>;
		pinctrl-names = "default";
		pinctrl-0 = <0x19b 0x19c 0x19d 0x19e>;
		reg-io-width = <0x04>;
		rockchip,grf = <0xbf>;
		rockchip,vo1_grf = <0xd0>;
		phys = <0x34>;                                      # changed from 0x19f
		phy-names = "hdmi";
		#sound-dai-cells = <0x00>;
		status = "okay";
		cec-enable = "true";
		enable-gpios = <0x150 0x0f 0x00>;
		phandle = <0x1c8>;

		ports {
			#address-cells = <0x01>;
			#size-cells = <0x00>;

			port@0 {
				reg = <0x00>;
				#address-cells = <0x01>;
				#size-cells = <0x00>;
				phandle = <0x47b>;

				endpoint@0 {
					reg = <0x00>;
					remote-endpoint = <0x1a0>;
					status = "disabled";
					phandle = <0xd7>;
				};

				endpoint@1 {
					reg = <0x01>;
					remote-endpoint = <0x3c>;
					status = "okay";
					phandle = <0xdd>;
				};

				endpoint@2 {
					reg = <0x02>;
					remote-endpoint = <0x1a1>;
					status = "disabled";
					phandle = <0xe5>;
				};
			};
		};
	};
```
After making the above changes and rebooting the device, the dmesg errors were gone and the display was back. Now, to find the cause of why this was happening, I went to radxa's source for the linux kernel and found out through their commit history that this was a change they made due to some internal parsing logic change in the kernel. And I was using the dtb intended for a previous kernel version on a newer one. So in short, while device-trees are mostly portable between different kernel versions, sometimes a breaking change can lead to them becoming faulty between kernel versions. And while there were many ways to solve this problem instead of directly debugging the device trees, I am confident that it still took less time then compiling U-Boot, the Linux Kernel and the dtbs from source again and again especially considering that the kernel version in which Radxa introduced the rock-5t dts and the kernel version being used by my board (https://github.com/Joshua-Riek/ubuntu-rockchip.git) was different. Also, the linux compilation process used by the source of the running linux image (https://github.com/Joshua-Riek/ubuntu-rockchip.git) was erroring out for me and was using outdated linux kernel source. So all in all, I had a great experience debugging this and learnt a lot of new things about the linux kernel, u-boot and hardware, also I made a github repo to document this hack and solve this issue for others using the same linux image.