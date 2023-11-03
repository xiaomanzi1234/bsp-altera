# Enclustra Build Environment - User Documentation

## Known Issues

### 1. SD Card access is not reliable

#### Affected hardware:

This issue was only observed on Mercury+ AA1 module equipped on Mercury ST1 base board.

#### Description

Linux does not boot or gets stuck while booting. Sometimes the SD Card stops working after Linux is up and running.

### 2. AA1 Linux is limited to 2 Gbyte DDR memory size

#### Affected hardware

All Arria 10 modules equipped with more than 2 Gbyte DDR of memory connected to the HPS:

- ME-AA1-480-2I3-D12E-NFX3

#### Description

The memory size is limited in U-Boot to max. 2 Gbyte. Therefore, only 2 Gbyte is available for U-Boot and Linux. However, the upper 2Gbyte can still be accessed by the FPGA fabric.

### 3. I2C frequency is wrong in U-Boot

#### Affected hardware

All Intel modules.

#### Description

In U-Boot, the I2C frequency is configured to be 100kHz. The frequency changes to 400kHz after "i2c probe" is executed. I2C gets reinitialized when probing fails (when "I2C probe" probes a non existent address), which changes the bus frequency.

### 4. USB host mode does not work in U-Boot

#### Affected hardware

All Mercury+ AA1 product models on PE1, ST1 and PE1 baseboards.

#### Description

USB host mode does not work in U-Boot because the U-Boot USB driver does not support overriding the mode dictated by the USB ID signal.

Last Page: [FAQ](./7_FAQ.md)