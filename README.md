# .NET Memory Copy [![Build status](https://ci.appveyor.com/api/projects/status/5yu0qhae9nth2lur?svg=true)](https://ci.appveyor.com/project/tewarid/MemoryCopy) [![Codacy Badge](https://api.codacy.com/project/badge/Grade/4265958e03b84de6a62c3b91c2cc9111)](https://www.codacy.com/app/tewarid/net-memory-copy?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=tewarid/net-memory-copy&amp;utm_campaign=Badge_Grade)

.NET Memory Copy is a .NET library to serialize an object to byte stream and back.

## Overview

C programmers frequently use [Winsock helper functions](https://msdn.microsoft.com/en-us/library/ms741394.aspx) such as htonl to change the [byte ordering](https://msdn.microsoft.com/en-us/library/3thek09d.aspx) of elements of a struct, and memcpy to transfer the struct to an output buffer. Migrating that kind of code can be a pain, because succinct code in C translates to a lot of code in languages that don’t support pointers.

.NET has structs and reflection. These make for a really potent combination for reading and writing data to the network. [`System.Runtime.InteropServices.Marshal`](https://msdn.microsoft.com/en-us/library/System.Runtime.InteropServices.Marshal.aspx) provides methods such as `Copy` and `PtrToStructure` that can be very useful. This facility is used by .NET to access unmanaged code.

If all your custom types are based on struct, you probably should use Marshal, but it will not work if you have class based types. If you are creating new types, you should be aware that structs don’t support inheritance. They are extended by means of composition. This library works with structs and classes.

## Getting Started

You can add the latest version of this library to your .NET project using [NuGet](https://www.nuget.org/packages/MemoryCopy/).

Here's a short [example](/ProtocolHeaderExample) of how to use this library to annotate your types, and read from binary. See unit tests for more usage scenarios.

```c#
    class FixedHeader
    {
        [DataMember(Order = 1)]
        public ushort Id { get; set; }
    }

    class VariableHeader1 : FixedHeader
    {
        [DataMember(Order = 2)]
        private ushort Size
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
            MemoryStream stream = new MemoryStream(data);

            MemoryCopy copy = new MemoryCopy();
            copy.ByteOrder = ByteOrder.BigEndian; // default

            FixedHeader h;
            Task<object> t = copy.Read(typeof(FixedHeader), stream, true);
            t.Wait();
            h = (FixedHeader)t.Result;

            if (h.Id == 1)
            {
                t = copy.Read(typeof(VariableHeader1), stream, false);
                t.Wait();
                VariableHeader1 varh = (VariableHeader1)t.Result;
                varh.Id = h.Id;
                Console.WriteLine("{0:x} {1:x}", varh.Id, varh.Payload.Length, false);
            }
        }
    }
```

The `Read` method sets the properties of an object that are annotated using the `DataMember` attribute, using data extracted from a stream of bytes. The `Write` method writes out annotated properties into a byte stream.

`Read` uses [`GetProperties`](http://msdn.microsoft.com/en-us/library/kyaxdd3x.aspx) method of `Type`, that may return properties in any particular order. You should use the `Order` property of `DataMember` to enforce the order in which values are read. If the object inherits from another type that also has annotated properties, the inherited properties are read or ignored based on the `inherit` parameter. They can also be masked in subclasses, by specifying the same value for `Order`.
