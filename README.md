# NetMemoryCopy [![Build status](https://ci.appveyor.com/api/projects/status/5yu0qhae9nth2lur?svg=true)](https://ci.appveyor.com/project/tewarid/netmemorycopy)

NetMemoryCopy is a .NET library to serialize an object to byte array and back.

## Overview

C programmers frequently use [Winsock helper functions](https://msdn.microsoft.com/en-us/library/ms741394.aspx) such as htonl to change the [byte ordering](https://msdn.microsoft.com/en-us/library/3thek09d.aspx) of elements of a struct, and memcpy to transfer the struct to an output buffer. Migrating that kind of code can be a pain, because succinct code in C translates to a lot of code in languages that don’t support pointers.

.NET has structs and reflection. These make for a really potent combination for reading and writing data to the network. [System.Runtime.InteropServices.Marshal](https://msdn.microsoft.com/en-us/library/System.Runtime.InteropServices.Marshal.aspx) provides methods such as Copy and PtrToStructure that can be very useful. This facility is used by .NET to access unmanaged code.

If all your custom types are based on struct, you probably should use Marshal, but it will not work if you have class based types. If you are creating new types, you should be aware that structs don’t support inheritance. They are extended by means of composition. This library works with structs and classes.

## Getting Started

You can add the latest version of this library to your .NET project using [NuGet](https://www.nuget.org/packages/NetMemoryCopy/).

Here's a short example of how to use this library to annotate your types, and read from binary. See unit tests for more usage scenarios.

```csharp
using System;
using System.Runtime.Serialization;

class FixedHeader
{
    [DataMember(Order = 1)]
    public ushort Id { get; set; }
}

class VariableHeader : FixedHeader
{
    [DataMember(Order = 2)]
    public ushort Size
    {
        get { return Payload == null? (ushort)0 : (ushort)Payload.Length; }
        set { Payload = new byte[value]; }
    }
    [DataMember(Order = 3)]
    public byte[] Payload { get; set; }
}

class Program
{
    public static void Main()
    {
        byte[] data = { 0x0, 0x1, 0x0, 0x5, 0x01, 0x02, 0x03, 0x04, 0x05 };

        MemoryCopy.MemoryCopy copy = new MemoryCopy.MemoryCopy();
        copy.ByteOrder = ByteOrder.BigEndian; // default

        FixedHeader h;
        int startIndex = 0;
        h = copy.Read(typeof(FixedHeader), data, ref startIndex, true);

        VariableHeader varh;
        varh = copy.Read(typeof(VariableHeader), data, ref startIndex, false);
        Console.WriteLine("{0:x} {1:x}", h.Id, varh.Size);
        Console.ReadLine();
    }
}
```
