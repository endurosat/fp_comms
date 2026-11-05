# fp-comms

Python library for EnduroSat's Function Protocol (FP), for use with
macchiato-generated Gen3 Python clients. It provides:

- FP serialization and deserialization (`fp_comms.fp_serdes`)
- `CspCanCommAgent`, a communication agent that carries FP over CSP v2 on a
  CAN bus, talking to the CAN adapter through
  [python-can](https://python-can.readthedocs.io). It needs no compiled CSP
  library and no other service.

## Install

Download the wheel from the GitHub release and install it:

```
pip install fp_comms-<version>+public-py3-none-any.whl
```

`<version>` is the release number, e.g. 2.3.0. This also installs
python-can and pydantic from PyPI.

### Windows: PEAK PCAN-USB

Install PEAK's PCAN-Basic driver from https://www.peak-system.com, plug the
adapter in, and use the `pcan` interface. The agent sets the bitrate
(default 1 Mbit/s), as it does for every python-can interface except
`socketcan`:

```python
agent = CspCanCommAgent("pcan://PCAN_USBBUS1", local_node=0x2A, bitrate=1_000_000)
```

### Linux: SocketCAN

The mainline kernel driver supports PEAK adapters; no PEAK software is
needed. Set the bitrate on the interface and bring it up:

```
sudo ip link set can0 down
sudo ip link set can0 up type can bitrate 1000000
sudo ip link set can0 txqueuelen 1000
```

The longer transmit queue matters for FP messages above about 80 bytes,
which span more CAN frames than the kernel's default queue of 10 holds.

Then use the `socketcan` interface:

```python
agent = CspCanCommAgent("socketcan://can0", local_node=0x2A)
```

## Quick start

`fp_ping` is the example interface from the examples archive published
next to the wheel: `ping.fidl` and the client and server generated from it.

```python
from fp_comms import CspCanAddress, CspCanCommAgent
from fp_ping import EchoRequest, fp_pingClient  # generated from ping.fidl

agent = CspCanCommAgent("socketcan://can0", local_node=0x2A)  # this PC's CSP node
agent.start()
try:
    client = fp_pingClient(comm_agent=agent, default_destination=CspCanAddress(node=0x2B))
    reply = client.send_req_echo(EchoRequest(seq=1, text="hello"), timeout_ms=2000)
    print(reply.echoed_seq, reply.echoed_text)
finally:
    agent.stop()
```

- FP travels as connectionless CSP packets to port 12 by default, with a
  CRC-32C trailer (`use_crc=True`, overridable per call).
- `CspCanAddress(node, port=12)` addresses a CSP node; ports are 0-63.
- The board can call the PC as well: register a handler with
  `agent.register_handler(iface_id, handler)` and answer with
  `agent.send_response(...)`.
- `send_broadcast` without a destination sends to the highest address in
  the local node's subnet (the `netmask` argument, default 8).
- Wrap the agent in `RetryingCommAgent(agent, max_attempts=3)` to resend
  requests that time out.
- Handlers run on the agent's receive thread. They may call `send_response`,
  `send_broadcast` and `stop`, but not `send_request`.

## Examples

The examples archive (`fp_comms-examples-<version>+public.zip`) holds an FP echo
client and server and a CSP ping, with a README that lists every command
and its expected output. They run without hardware on python-can's virtual
bus, on a Linux `vcan` interface, or on a real CAN bus.

## License

The fp_comms wheel and the examples are available under the EnduroSat Software
License, version 1.0, see `LICENSE`. It allows use, modification and
distribution, including commercial use, but not publishing a rebranded copy of
the software. python-can is a separate package under the LGPL-3.0.
