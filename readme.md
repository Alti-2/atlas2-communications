<img src="https://github.com/Alti-2/atlas2-communications/blob/main/Alti2splash.png"
     alt="Alti-2 splash"
     style="width: 400px;" />
# Atlas 2 Device Communication
### **WARNING**! Altering your device software may cause unreliable operation and lead to severe injury or death. Use of this information is at your own risk.

## Configuration
Establish a serial connection to the device with the following settings
- Baud:  57600
- Data bits: 8
- Parity: None
- Stop bits: 1
- Flow Control: CTS/RTS
- DTR: enabled

## Communication with the device
Many serial libraries do not correctly handle flow control. Care must be taken to not send data over the serial line while the CTS line is lowered. To communicate with the device you will first need to get the Info message then parse the necessary information for the encryption key. All packets from the device require an ack except the Info message. Note that byte order for device communication is **little endian**. 

*The device expects all packets to be 32 bytes long and encrypted. If your packet is less than 32 bytes, pad the packet to 32 bytes, encrypt, then send to the device.*

The device will respond with the follow codes after receiving a packet:
- 0x30:	Abort communication.	
- 0x31:	Ack: Encrypted Packet Received Successfully	
- 0x32:	Packet Length error: Decrypted length exceeds max allowed. Could indicate the keys used for encoding and decoding are not identical.
- 0x33:	Checksum Error	Typically from a malformed packet. Any command within the packet is ignored. 
- 0x34:	Overflow Error. Typically from flow control failures.

## Info Message (aka Type Zero Message)
After opening a connection to the device send 6 bytes (doesn't matter what) and the device will respond with an unencrypted 32 byte Info message. Do not send an ack for the Info message, doing so will cause the device to disconnect. The Info message will be sent as ascii representation of hex values. An example message is 

`1E 00 05 10 09 32 33 30 30 37 34 32 33 20 01 0C 01 00 20 01 00 03 00 BF 00 00 00 20 05 00 00 E9`

| Position | Contents                                               |
| -------- | ------------------------------------------------------ |
| 0        | Length of the packet. should be	0x1E                   |
| 1        | Record type	0x00                                       |
| 2        | Internal use                                           |
| 3        | Software Rev & Major Software Version                  |
| 4        | Software Minor Version                                 |
| 5 – 13   | 9 Bytes of serial number. Example: “200221245“         |
| 14       | Hardware Id                                            |
| 15       | Product ID                                             |
| 16       | Fram Configuration	0x00, 0x01                          |
| 17 – 20  | Fram Detailed Log Start Address Example: “00 00 20 01” |
| 21 – 22  | Total Number of Jumps                                  |
| 23 – 26  | Total Jump In Seconds                                  |
| 27 – 30  | Fram Summary Log Start Address	Example “20 05 00 00”   |
| 31       | Checksum                                               |


## Encryption
All messages other than the Info message is encrypted using the [XTEA cipher](https://en.wikipedia.org/wiki/XTEA). One notable exception is the Atlas uses 16 rounds instead of the typical 32 or 64. The encryption key is built using product specific key codes and information from the info message.

| Product | Codes              |
| ------- | ------------------ |
| Atlas   | `[0x4E,0x75,0x7E]` |
| Atlas 2 | `[0xAA,0x69,0x44]` |
| Juno    | `[0x8D,0xAF,0x11]` |

The Key
| Position | Index of value in Type Zero Message | Value                 |
| -------- | ----------------------------------- | --------------------- |
| 0        |                                     | Product Codes[0]      |
| 1        | Type Zero Message[23]               | Total Jump Seconds[0] |
| 2        | Type Zero Message[6]                | Serial Number[1]      |
| 3        | Type Zero Message[13]               | Serial Number[8]      |
| 4        | Type Zero Message[24]               | Total Jump Seconds[1] |
| 5        | Type Zero Message[22]               | Total Jumps[1]        |
| 6        | Type Zero Message[12]               | Serial Number[7]      |
| 7        |                                     | Product Codes[1]      |
| 8        | Type Zero Message[7]                | SerialNum[2]          |
| 9        | Type Zero Message[8]                | SerialNum[3]          |
| 10       | Type Zero Message[10]               | SerialNum[5]          |
| 11       |                                     | Product Codes[2]      |
| 12       | Type Zero Message[9]                | Serial Number[4]      |
| 13       | Type Zero Message[11]               | Serial Number[6]      |
| 14       | Type Zero Message[26]               | Total Jump Seconds[3] |
| 15       | Type Zero Message[25]               | Total Jump Seconds[2] |


Example C Encryption:

```c
#define Delta 0x9E3779B9

void encode(uint32_t *v, uint32_t *const w)
{
  uint32_t long y = v[0], z = v[1];
  uint32_t long sum = 0;
  int n = 16;

  while (n-- > 0)
  {
    y += (((z << 4) ^ (z >> 5)) + z) ^ (pKey[(sum & 3)] + sum);
    sum += Delta;
    z += (((y << 4) ^ (y >> 5)) + y) ^ (pKey[((sum >> 11) & 3)] + sum);
  }

  w[0] = y;
  w[1] = z;
}
```
Example C Decryption:
```c
#define Delta 0x9E3779B9

void decode(const uint32_t *const v, uint32_t *const w)
{
  uint32_t y = v[0];
  uint32_t z = v[1];
  uint32_t sum = (uint32_t)Delta * n;
  int n = 16;

  while (n-- > 0)
  {
    z -= (((y << 4) ^ (y >> 5)) + y) ^ (pKey[((sum >> 11) & 3)] + sum);
    sum -= Delta;

    y -= (((z << 4) ^ (z >> 5)) + z) ^ (pKey[(sum & 3)] + sum);
  }

  w[0] = y;
  w[1] = z;
}
```

## A Commands
- A0: Read EEprom Min to read is 2 bytes, maximum to read is 64K bytes
- A1: Read Information memory. Min to read is 2 bytes, maximum to read is 128 bytes
- A2: Read Time & Date
- A4:	Ack Command:  This command should be used as a “keep alive” message.  Neptune responds with a one character code. 
- A5:	Send encrypted Info message.
- AF:	Exit Command

A0 and A1 commands have the same format:  4 Byte Address, followed by 2 byte read length. The address must be sent “little endian”. If the packet is less than 32 bytes, pad the packet to 32 bytes, then encrypt before sending to the device.

### A0/1 Response packet structure
The response to a read command has the following structure. The four bytes of the address in the response are the same as the four bytes sent in the A0 or A1 command. A simple read test of address 0000 will return the serial number.

| Index | Contents                                        |
| ----- | ----------------------------------------------- |
| 0     | Packet length                                   |
| 1     | Command  (A1 / A2)                              |
| 2     | Source Address LSB                              |
| 3     | Source Address                                  |
| 4     | Source Address                                  |
| 5     | Source Address MSB                              |
| 6     | Length LSB                                      |
| 7     | Length MSB                                      |
| 8-30  | Data                                            |
| 31    | Checksum or Data if another packet is following |

## B Commands
- B0:	Write to EEProm
- B1:	Write to Info Memory. Starting Address is at 0x1000 and max. of 128 bytes can be written.
- B2:	Set Time & Date
- B3:	Erase Info Mem.
- B4:	Reset to Bootloader mode. **Warning** Attempting to write new firmware to the device may overwrite existing firmware and prevent normal operation of the device.

### Write Packet Structure
	B0 and B1 commands have a similar format as the A0/1 commands. Address (4 bytes) sent “little endian” followed by the data to be written. The maximum size of packets is 32 bytes. 3 bytes are taken by the length, command, and checksum. Another 4 bytes are used for the destination address. This leaves a total of 25 bytes of payload data per packet.

| Index | Contents                |
| ----- | ----------------------- |
| 0     | Packet length           |
| 1     | Command  (B1 / B2)      |
| 2     | Destination Address LSB |
| 3     | Destination Address     |
| 4     | Destination Address     |
| 5     | Destination Address MSB |
| 6-30  | Payload                 |
| 31    | Checksum                |

  ## Device Responses

## Command Packet Responses
When the device receives a command packet it will respond with with an ACK and one of the following codes.

- 0x35:	Command executed properly.
- 0x36:	Unrecognized Command
- 0x37:	Invalid Syntax for the command
- 0x38:	EEPRom/Fram write Error
- 0x39:	Flash Erase Error
- 0x41:	Info Mem Address is out of bounds.
- 0x42:	Flash write error

## Date and Time
Both encoding and decoding use the same data structure on the device side.  The C structures are shown below, VB code that deals with transforming a PC date and time into a device structured date and time, and the reverse as well. 
```c
        typedef struct                                
             short year; 
             unsigned char month;
             unsigned char day;
         Date;
         typedef struct                               
             BYTE Chksum; 
             BYTE hour; 
             BYTE minute; 
             BYTE second; 
         Timer;
```
```vba
Public Function decodedatetime(ByVal pInstring As String) As DateTime
        ' this decodes the Neptune 3 style date/time send
        Dim rval As DateTime
        ' D7 07 02 06 E7 00 01 16 A4 07 09 26 88 87 7E 23 A4 07 09 26 88 87 7E 23 A4 07 09 26 88 87 7E 23
        ' 1  2  3  4  5  6  7  8  9  10 11 12
        ' 1  4  7  1  1  1  1  2  2  2  3
        ' 1234567890123456789012345678901234567890 
        ' D7 07 YEAR
        ' 02 month
        ' 06 day
        ' E7 checksum
        ' 00 hour
        ' 01 minute
        ' 16 second
        ‘ from “A4” on, after “16”, the data in the packet is irrelevant to date and time
        Dim iYear As Integer = (HexChar2Byte(Mid(pInstring, 5, 1)) * 256) + (HexChar2Byte(Mid(pInstring, 1, 1)) * 16) + HexChar2Byte(Mid(pInstring, 2, 1))
        Dim iMonth As Integer = (HexChar2Byte(Mid(pInstring, 7, 1)) * 16) + HexChar2Byte(Mid(pInstring, 8, 1))
        Dim iDay As Integer = (HexChar2Byte(Mid(pInstring, 10, 1)) * 16) + HexChar2Byte(Mid(pInstring, 11, 1))
        Dim iHour As Integer = (HexChar2Byte(Mid(pInstring, 16, 1)) * 16) + HexChar2Byte(Mid(pInstring, 17, 1))
        Dim iMinute As Integer = (HexChar2Byte(Mid(pInstring, 19, 1)) * 16) + HexChar2Byte(Mid(pInstring, 20, 1))
        Dim iSecond As Integer = (HexChar2Byte(Mid(pInstring, 22, 1)) * 16) + HexChar2Byte(Mid(pInstring, 23, 1))
        On Error Resume Next  
        rval = CDate(iMonth & "/" & iDay & "/" & iYear & " " & iHour.ToString & ":" & iMinute.ToString & ":" & iSecond.ToString)

        Return rval
    End Function

    Public Function encodedatetime(ByVal pDateTime As DateTime) As Byte()
        Dim rval As Byte()
        Dim i As Integer
        i = ((CByte(pDateTime.Year / 256) - 1) * 256)
        ' number of years after we get rid of the count given in byte 1
        ' the first byte is the hardest: the lower nibble
        ReDim rval(7)
        rval(0) = (pDateTime.Year - i)
        rval(1) = CByte(pDateTime.Year / 256) - 1
        rval(2) = pDateTime.Month
        rval(3) = pDateTime.Day
        rval(4) = 0 ' checksum
        rval(5) = pDateTime.Hour                     ' always 24 hour clock
        rval(6) = pDateTime.Minute
        rval(7) = pDateTime.Second
        Return rval
    End Function
```
