---
layout: post
title:  "Hacking IOBluetooth"
date:   2018-03-25
---

# Hacking `IOBluetooth`

I recently did a little research project to see if I could control the Bluetooth controller on my Mac via HCI (Host Controller Interface) commands in order to replicate the functionality of [PureSwift/BluetoothLinux](https://github.com/PureSwift/BluetoothLinux) on macOS. Since I have my [PureSwift/Bluetooth](https://github.com/PureSwift/Bluetooth) library with some HCI commands implemented, I assumed the endeavor was only a matter of opening a socket to the Bluetooth adapter and sending bytes, which is how its done on Linux. It turns out that Apple's Darwin kernet is more secure and doesn't allow userland code to open a socket and talk directly to the hardware, instead CoreBlueooth acts as a proxy for `bluetoothd` which use the `IOBluetooth` framework, which in turn opens a Mach port to a Kernel Extension (Device driver) which is loaded with the Kernel. In theory, this should be much more secure than Linux becuase it would impossible to talk to the Bluetooth hardware directly even with elevated permissions like `sudo`. Here is a breakdown on how the CoreBluetooth API works:

App code (loads dynamic library) 

-> (via ObjC) **CoreBluetooth** (`CBCentralManager` is a proxy for `CBXpcConnection`) 

-> (via XPC) **com.apple.bluetoothd** deamon with root permissions (Objective-C `IOBluetooth.IOBluetoothHostController`) 

-> (via Mach Port) **Kernel Extension** / .kext (C++ `IOKit.IOBluetoothHostController`)

Thankfully, the Objetive-C `IOBluetoothHostController` API is a public API that has been around since macOS 10.2, despite it being the code that runs on the `bluetoothd` service. This Objective-C class allows for some basic operations with your Bluetooth controller according the official APIs. I used Hopper to decompile `bluetoothd` and discovered that it was using `IOBluetoothHostController`, which led me to believe it was using private APIs I could exploit. Using `class-dump`, I discovered that Apple implemented all the HCI commands in the Bluetooth 4.2 specification via [private methods](https://github.com/colemancda/HCITool/blob/42afa45337781405897135ec11d86ddf6934ebe2/HCITool/IOBluetoothHostController.h) on `IOBluetoothHostController`. Changing the local name of a Mac's Bluetooth adapter via these private APIs is pretty easy and requires no root access or special entitlements.

```objective-c
IOBluetoothHostController *hciController = [IOBluetoothHostController defaultController]; // public API
unsigned char name[256];
name[0] = 'C';
name[1] = 'D';
name[2] = 'A';
[hciController BluetoothHCIWriteLocalName:&name]; // private API
```

If you check in *Bluetooth Preferences* or discover your Bluetooth adapter with an app like [LightBlue Explorer](https://itunes.apple.com/us/app/lightblue-explorer/id557428110?mt=8), its unlikely you will see your Bluetooth local name change since the values are cached. Apple provides [PacketLogger and BluetoothExplorer](https://robservatory.com/debugging-bluetooth-issues-in-macos-sierra/) which allow you to debug your hardware (and requires elevated permissions to run). 

![Image]({{ site.url }}/images/IOBLuetoothChangeLocalNamePacketLogger.png)

![Image]({{ site.url }}/images/BluetoothExplorerRefreshLocalName.gif)

Not content with letting Apple build and send the HCI commands, I decided to use my favorite decompiler, [Hopper](https://www.hopperapp.com), to see if there was a way to control the hardware directly. 

```bash
open /System/Library/Frameworks/IOBluetooth.framework/Versions/A/IOBluetooth -a /Applications/Hopper\ Disassembler\ v4.app
```

![Image]({{ site.url }}/images/HopperIOBluetoothHostControllerBluetoothHCIWriteLocalName.png)

```c
int -[IOBluetoothHostController BluetoothHCIWriteLocalName:](void * self, void * _cmd, unsigned char[248] arg2) {
    var_18 = arg2;
    var_A0 = 0x4;
    var_A4 = 0x10e1;
    var_90 = &var_A4;
    var_AC = _BluetoothHCIDispatchUserClientRoutine(&var_90, &var_94, &var_A0);
    if (sign_extend_64((0x0 != var_AC ? 0x1 : 0x0) & 0x1 & 0xff) == 0x0) {
            var_90 = &var_94;
            var_AC = _BluetoothHCIDispatchUserClientRoutine(&var_90, 0x0, 0x0);
            var_90 = &var_94;
            _BluetoothHCIDispatchUserClientRoutine(&var_90, 0x0, 0x0);
    }
    rax = var_AC;
    return rax;
}
```

Most of the private methods for HCI commands follow a pattern which requires sending data via a Mach port to the IOKit service to forward to the HCI command via `BluetoothHCIDispatchUserClientRoutine()`. 

```c
int _BluetoothHCIDispatchUserClientRoutine(int arg0, int arg1, int arg2) {
    var_8 = arg0;
    var_10 = arg1;
    var_18 = arg2;
    var_1C = _BluetoothHCISetupUserClient();
    if (var_1C == 0x0) {
            if ((var_18 != 0x0) && (*var_18 > 0x100000)) {
                    var_1C = 0xe00002c2;
            }
            else {
                    var_1C = IOConnectCallStructMethod(*(int32_t *)0x12da64, 0x0, var_8, 0x74, var_10, var_18);
                    if (var_1C != 0x0) {
                            if (var_1C > 0x64) {
                                    _BluetoothHCIDecodeIOReturnError(var_1C);
                            }
                            else {
                                    _BluetoothHCIDecodeError(var_1C);
                            }
                    }
            }
    }
    rax = var_1C;
    return rax;
}

```

After a while of reading assembly and psuedocode, I discovered that `IOBluetoothHostController` has subclasses for specific vendors. 

- `AtherosHostController`
- `BroadcomHostController`
- `CSRBlueICEHostController`
- `CSRHostController`

I assumed that Apple engineers had done this to provide vendor specific HCI commands that are not part of the Bluetooth standard. Since these classes are implemented in the Objective-C library, I assumed they do not want to modify the C++ IOKit Kernel Extension library since its need to be extremely stable and any changes could potentially cause kernel crashes, not to mention that C++ suffers from class fragility and a Kext compiled for an older macOS version would not be able to link with a newer IOKit release if they made ABI breaking changes in the C++ Kernel Extensions layer. I was able to find the needle in the haystack in one of the vendor HCI commands that Apple seemed to be using for testing Mac hardware before shipping.

```c
int _IOBluetoothCSRLibHCISendBCCMDMessage(int arg0) {
    var_8 = **___stack_chk_guard;
    var_18 = arg0;
    var_21 = _IOBluetoothCSRLibGetBCCMDMessageLen(var_18);
    var_30 = &stack[-136] - ((var_21 & 0xff) + 0x13 & 0x1f0);
    var_32 = 0xfc00;
    __memmove_chk(var_30, &var_32, 0x2, 0xffffffffffffffff);
    *(int8_t *)(var_30 + 0x2) = (var_21 & 0xff) + 0x1;
    *(int8_t *)(var_30 + 0x3) = 0xc2;
    if ((var_21 & 0xff) > 0x0) {
            __memmove_chk(var_30 + 0x4, var_18, var_21 & 0xff, 0xffffffffffffffff);
    }
    var_1C = _BluetoothHCIRequestCreate(&var_20, 0x1770, 0x0, 0x0);
    if (var_1C == 0x0) {
            var_1C = _BluetoothHCISendRawCommand(var_20, var_30, sign_extend_64((var_21 & 0xff) + 0x4));
            sleep(0x1);
            _BluetoothHCIRequestDelete(var_20);
    }
    var_74 = var_1C;
    if (**___stack_chk_guard == var_8) {
            rax = var_74;
    }
    else {
            rax = __stack_chk_fail();
    }
    return rax;
}
```

To send a raw HCI command, a userland executable has to perform the following procedure:

1. Allocate an HCI request buffer on the IOKit service driver and get its identifier.
2. Send a struct whose values will be used as arguments to call a C++ function on the IOKit driver. In this case, we include the request identifier, raw HCI command data, and the size of the command data that will be sent to the controller. In the Kernel Extension, the `IOBluetoothHCIController` service (a C++ object) will extract these values from the sent struct and call the once of its corresponging C++ methods on the actual driver code loaded with the kernel (e.g. `IOBluetoothHCIUserClient::DispatchHCIReadLocalName()`, `IOBluetoothHostController::SendRawHCICommand`). 
3. Finally, we cleanup our operation in the Kext by requesting the driver to delete its HCI command request buffer.

While this procedure is typically done by the `com.apple.bluetoothd` deamon with elevated permissions, it seems that any executable can use this procedure without special permissions or entitlements. I updated my method to write the Bluetooth controller's local name with the Objective-C API and using `BluetoothHCISendRawCommand()` directly.

```objective-c

struct HCIRequest {
    uint32 identifier;
};

- (void)writeName:(id)sender {
    
    IOBluetoothHostController *hciController = [IOBluetoothHostController defaultController];
    
    unsigned char name[256];
    
    name[0] = 'C';
    name[1] = 'D';
    name[2] = 'A';
    
    int setNameError = [hciController BluetoothHCIWriteLocalName:&name];
    
    if (setNameError) {
        
        NSLog(@"Error %@", @(setNameError));
        return;
    }
    
    NSLog(@"BluetoothHCIWriteLocalName");
    
    // manually via C function
    
    struct HCIRequest request;
    int error = BluetoothHCIRequestCreate(&request, 1000, nil, 0);
    
    NSLog(@"Created request: %lu", request.identifier);
    
    if (error) {
        
        BluetoothHCIRequestDelete(request);
        printf("Couldnt create error: %08x\n", error);
    }
    
    size_t commandSize = 248 + 3;
    unsigned char command[commandSize];
    memset(&command, '\0', commandSize);
    command[0] = 0x13;
    command[1] = 0x0C;
    command[2] = 248;
    command[3] = 'A';
    command[4] = 'B';
    command[5] = 'C';
    
    error = BluetoothHCISendRawCommand(request, &command, commandSize);
    
    if (error) {
        
        BluetoothHCIRequestDelete(request);
        printf("Send HCI command Error: %08x\n", error);
    }
    
    sleep(0x1);
    
    BluetoothHCIRequestDelete(request);
}
```

I attempted to read the `Accept Connection Timeout` parameter with the ObjC private API and with the C functions, but `BluetoothHCISendRawCommand()` doesn't allow for HCI command return parameters, 

```objective-c

- (void)readConnectionTimeout:(id)sender {
    
    IOBluetoothHostController *hciController = [IOBluetoothHostController defaultController];
    
    uint16 connectionAcceptTimeout = 0;
    
    int readTimeout = [hciController BluetoothHCIReadConnectionAcceptTimeout:&connectionAcceptTimeout];
    
    if (readTimeout) {
        
        NSLog(@"Error %@", @(readTimeout));
        return;
    }
    
    // BluetoothHCIReadConnectionAcceptTimeout 2440
    NSLog(@"BluetoothHCIReadConnectionAcceptTimeout %@", @(connectionAcceptTimeout));
    
    // manually
    
    struct HCIRequest request;
    
    // No funcion argument of `BluetoothHCISendRawCommand()`
    // allows for writing output values.
    uint16 output;
    size_t outputSize = sizeof(output);
    
    int error = BluetoothHCIRequestCreate(&request, 1000, nil, 0);
    
    NSLog(@"Created request: %u", request.identifier);
    
    if (error) {
        
        BluetoothHCIRequestDelete(request);
        
        printf("Couldnt create error: %08x\n", error);
    }
    
    size_t commandSize = 3;
    uint8 * command = malloc(commandSize);
    command[0] = 0x15;
    command[1] = 0x0C;
    command[2] = 0;
    
    error = BluetoothHCISendRawCommand(request, command, 3);
    
    if (error) {
        
        BluetoothHCIRequestDelete(request);
        printf("Send HCI command Error: %08x\n", error);
    }
    
    sleep(0x1);
    
    BluetoothHCIRequestDelete(request);
    
    // BluetoothHCIReadConnectionAcceptTimeout 0
    NSLog(@"BluetoothHCIReadConnectionAcceptTimeout %@", @(output));
}
```

Upon verifying with `PacketLogger` that the command was being sent to the controller, I needed to find a way had to way to retrieve the HCI command return parameter value. My solution was to creating another C function that imitates `BluetoothHCISendRawCommand()` and would accept a data pointer for output.

```c
int _BluetoothHCISendRawCommand(int arg0, int arg1, int arg2) {
    var_8 = arg0;
    var_10 = arg1;
    var_18 = arg2;
    memset(&var_90, 0x0, 0x74);
    if ((var_10 != 0x0) && (var_18 > 0x0)) {
            var_90 = &var_8;
            var_4 = _BluetoothHCIDispatchUserClientRoutine(&var_90, 0x0, 0x0);
    }
    else {
            var_4 = 0xe00002c2;
    }
    rax = var_4;
    return rax;
}
```

Despite reverse engineering the procedure, the decompiled psuedo code only indicated a value on the stack that is passed to `BluetoothHCIDispatchUserClientRoutine()`, not its size or what kind of struct. I was able to deduce the size of struct / data being sent is `0x74` or 116 bytes since is it standard practice to properly initialize a struct or data pointer in C by settings its contents to 0 with `memset()`. I could also verify that the data provided to `BluetoothHCIDispatchUserClientRoutine()` was always 116 bytes since that value (`0x74`) was  provided as the length of the data sent to the IOKit service.

```c
int _BluetoothHCIDispatchUserClientRoutine(int arg0, int arg1, int arg2) {
    var_8 = arg0;
    var_10 = arg1;
    var_18 = arg2;
    var_1C = _BluetoothHCISetupUserClient();
    if (var_1C == 0x0) {
            if ((var_18 != 0x0) && (*var_18 > 0x100000)) {
                    var_1C = 0xe00002c2;
            }
            else {
                    var_1C = IOConnectCallStructMethod(*(int32_t *)0x12da64, 0x0, var_8, 0x74, var_10, var_18);
                    if (var_1C != 0x0) {
                            if (var_1C > 0x64) {
                                    _BluetoothHCIDecodeIOReturnError(var_1C);
                            }
                            else {
                                    _BluetoothHCIDecodeError(var_1C);
                            }
                    }
            }
    }
    rax = var_1C;
    return rax;
}

```

`IOConnectCallStructMethod()` is used to call the C++ method of the Bluetooth kernel extension (also called `IOBluetoothHostController`, but implemented in C++, not Objective-C) and requires the C++ methods' arguments and return value to be provided as data pointers. After searching online for IOBluetooth vulnerabilites I found an [example](http://roberto.greyhats.it/2015/01/osx-bluetooth-lpe.html) of calling a C++ member, `IOConnectCallStructMethod` of `IOBluetoothHostController`.

```obejctive-c
struct BluetoothCall { // 120 bytes
  uint64_t args[7];
  uint64_t sizes[7];
  uint64_t index;
};

	struct BluetoothCall a;
  int i;
  
  int main(void) {
  /* Finding vuln service */
  io_service_t service =
    IOServiceGetMatchingService(kIOMasterPortDefault,
				IOServiceMatching("IOBluetoothHCIController"));

  if (!service) {
    return -1;
  }

  /* Connect to vuln service */
  io_connect_t port = (io_connect_t) 0;
  kern_return_t kr = IOServiceOpen(service, mach_task_self(), 0, &port);
  IOObjectRelease(service);
  if (kr != kIOReturnSuccess) {
    return kr;
  }

  kr = IOConnectCallMethod((mach_port_t) port, /* Connection */
			   (uint32_t) 0,       /* Selector */
			   NULL, 0,            /* input, inputCnt */
			   (const void*) &a,   /* inputStruct */
			   120,                /* inputStructCnt */
			   NULL, NULL, NULL, NULL); /* Output stuff */
  printf("kr: %08x\n", kr);

  return IOServiceClose(port);
 }
```

I assumed the 116 bytes sent in `BluetoothHCIDispatchUserClientRoutine()` is a `BluetoothCall` struct and the last member is `uint32` instead of `uint64`. With this information I tried to reimplement `BluetoothHCISendRawCommand()` in order to add arguments to allow for HCI commands return parameters. While Hopper didnt specify which values of the struct were being sent I was able to deduce this information by dumping the value via a symbolic breakpoint for `BluetoothHCIDispatchUserClientRoutine()` in LLDB:

```
(lldb) memory read --size 8 --format 'A' --count 15 $arg1

0x7ffeefbfe9b0: 0x00007ffeefbfea38 -> 0x000000010000002c HCITool`_mh_execute_header + 44 // HCI command request ID pointer
0x7ffeefbfe9b8: 0x000060c000000640 // HCI command data pointer
0x7ffeefbfe9c0: 0x00007ffeefbfea28 // HCI command data size pointer
0x7ffeefbfe9c8: 0x0000000000000000
0x7ffeefbfe9d0: 0x0000000000000000
0x7ffeefbfe9d8: 0x0000000000000000
0x7ffeefbfe9e0: 0x0000000000000000
0x7ffeefbfe9e8: 0x0000000000000004 // size of uint32 (HCI request identifier)
0x7ffeefbfe9f0: 0x0000000000000003 // size of HCI command data
0x7ffeefbfe9f8: 0x0000000000000008 // size of HCI command data size value (`size_t`)
0x7ffeefbfea00: 0x0000000000000000
0x7ffeefbfea08: 0x0000000000000000
0x7ffeefbfea10: 0x0000000000000000
0x7ffeefbfea18: 0x0000000000000000
0x7ffeefbfea20: 0x000060c000000062 // IOKit Service Method Index
```

With this information I was able to reimplement `BluetoothHCISendRawCommand()`

```c
int _BluetoothHCISendRawCommand(struct HCIRequest request, void *commandData, size_t commmandSize) {
    
    int errorCode = 0;
    
    struct BluetoothCall call;
    size_t size = 0x74;
    memset(&call, 0x0, size);
    
    if ((commandData != 0x0) && (commmandSize > 0x0)) {
        
        call.args[0] = (uintptr_t)&request.identifier;
        call.args[1] = (uintptr_t)commandData;
        call.args[2] = (uintptr_t)&commmandSize;
        call.sizes[0] = 0x04;
        call.sizes[1] = 0x03;
        call.sizes[2] = 0x08;
        call.index = 0x000060c000000062;
        
        // IOBluetoothHostController::SendRawHCICommand(unsigned int, char*, unsigned int, unsigned char*, unsigned int)
        errorCode = BluetoothHCIDispatchUserClientRoutine(&call, 0x0, 0x0);
    }
    else {
        errorCode = 0xe00002c2;
    }
    
    return errorCode;
}
```

You can find the source code on GitHub at [ColemanCDA/HCITool](https://github.com/ColemanCDA/HCITool).


