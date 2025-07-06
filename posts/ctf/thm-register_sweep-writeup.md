# THM: Register Sweep (Industrial Intrusion CTF 2025)

"During a recent audit of legacy OT systems, engineers identified an undocumented Modbus TCP device
still active on the network. It's believed this device was once used for 
temporary configuration storage during early system deployment, but its documentation has since been lost.

You've been tasked with manually inspecting the device's register space 
to ensure nothing sensitive remains embedded within it. Most data appears normal, but 
a specific holding register contains deliberately stored **ASCII-encoded 
information** that must be retrieved and reviewed.

Start the machine by clicking the start machine button. 
You may access the Modbus service on **port 502**."

The only relevant port here is port 502, which holds the Modbus servuce that contains our flag.
Let's start msfconsole to use the modbus client.

```bash
msf6 auxiliary(scanner/scada/modbusclient) > search modbus

Matching Modules
================

   #   Name                                            Disclosure Date  Rank    Check  Description
   -   ----                                            ---------------  ----    -----  -----------
   0   auxiliary/analyze/modbus_zip                    .                normal  No     Extract zip from Modbus communication
   1   auxiliary/scanner/scada/modbus_banner_grabbing  .                normal  No     Modbus Banner Grabbing
   2   auxiliary/scanner/scada/modbusclient            .                normal  No     Modbus Client Utility
```

We will use the READ_HOLDING_REGISTERS action to gather information.
These are my options to read from the Modbus service:

```bash
Module options (auxiliary/scanner/scada/modbusclient):

   Name            Current Setting  Required  Description
   ----            ---------------  --------  -----------
   DATA                             no        Data to write (WRITE_COIL and WRITE_REGISTER modes only)
   DATA_ADDRESS    0                yes       Modbus data address
   DATA_COILS                       no        Data in binary to write (WRITE_COILS mode only) e.g. 0110
   DATA_REGISTERS                   no        Words to write to each register separated with a comma (WRITE_REGISTERS mode only) e.g. 1,2,3
                                              ,4
   HEXDUMP         true             no        Print hex dump of response
   NUMBER          125              no        Number of coils/registers to read (READ_COILS, READ_DISCRETE_INPUTS, READ_HOLDING_REGISTERS,
                                              READ_INPUT_REGISTERS modes only)
   RHOSTS          10.10.223.52     yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasp
                                              loit.html
   RPORT           502              yes       The target port (TCP)
   UNIT_NUMBER     1                no        Modbus unit number


Auxiliary action:

   Name                    Description
   ----                    -----------
   READ_HOLDING_REGISTERS  Read words from several HOLDING registers
```

Running this returns the following:
```bash
[+] 10.10.223.52:502 - 125 register values from address 0 : 
[+] 10.10.223.52:502 - [15885, 27951, [...], 37439, 32120]
```
This doesn't reveal ASCII information, let's try using the hexdump options to get more information:
```bash
[+] 10.10.223.52:502 - response: 0000000000fd0103fa3e0d6d2f3a402700efddc696f267eddeb89423c8c2aad20d509a4dcad63263b6f09e64f7d595cde2672fa737c3a <SNIP>
[+] 10.10.223.52:502 - 125 register values from address 0 : 
[+] 10.10.223.52:502 - [15885, 27951, 14912, 9984, 61405, 50838, 62055, 60894, 47252, 9160, 49834, 53773, 20634, 19914 <SNIP>
```

This hexdump didn't return anything useful when using Cyberchef...
Let's try to only focus on the numerical register values.

We will use this Python script to turn the numerical data into ASCII characters:
```python
# List of register values from the Modbus scan
registers = [15123, 48101, 64631, <SNIP>]

# Function to convert two consecutive registers to ASCII
def convert_to_ascii(registers):
    ascii_chars = []
    
    for i in range(0, len(registers) - 1, 2):
        high = registers[i]
        low = registers[i + 1]
        
        # Combine high and low 16-bit register values into a 32-bit integer
        combined = (high << 16) + low
        
        # Convert this 32-bit integer to bytes and decode as ASCII
        ascii_str = combined.to_bytes(4, byteorder='big').decode('ascii', errors='ignore')
        ascii_chars.append(ascii_str)
    
    return ''.join(ascii_chars)

# Convert the registers and print the result
ascii_output = convert_to_ascii(registers)
print(ascii_output)
```

Let's run this using our first list of numerical values:
```bash
└─$ python script1.py                                 
PM2cdg/7▒EMa8;#{d<8+Kwh
                       7k&w[vH%KzKCQ;d4)s%KQin3-{gwI4\XAoGaM`!JLw/$qlJfBW<GI> ?
```
We need more data, let's try getting it from registers with higher addressess.

We can use the DATA_ADDRESS option on Metasploit to move further in hopes of finding our flag.
```bash
msf6 auxiliary(scanner/scada/modbusclient) > set DATA_ADDRESS 50
msf6 auxiliary(scanner/scada/modbusclient) > run
[Paste the register values in the register variable in our script]
└─$ python script1.py
h
 7k&w[vH%KzKCQ;d4)s%KQin3-{gwI4\XAoGaM`!JLw/$qlJfBW<GI> ?}xZJ▒1!O6~`]x~]▒G~2Ew>D>=aQp>jg▒e▒U9#&"(BTHM{FL
```

We can see our flag now!
Let's continue moving our DATA_ADDRESS
```bash
msf6 auxiliary(scanner/scada/modbusclient) > set DATA_ADDRESS 100
msf6 auxiliary(scanner/scada/modbusclient) > run
[Paste the register values in the register variable in our script]
└─$ python script1.py
;w/$qlJfBW<GI> ?}xZJ▒1!O6~`]x~]▒G~2Ew>D>=aQp>jg▒e▒U9#&"(B*THM{FLAG}*F|m@j}upFjZN{h|4+73Ig*kw [HMt;X5
```

Our flag is located within the Modbus registers!

