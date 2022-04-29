## Experiments in Air Conditoner IR code recording

### Device: Insignia NS-AC08PWH1

IR signal duty is 38khz modulated, 50% duty cycle

Sampling from a Sensibo IR gateway, since I don't have access to the factory remote.

![waveform.png](waveform.png)

## Timings 

* packets start with a 4400us pulse, followed by 4400us wait
* Subsequent pulses are always 400us long
* A High bit is encoded as a 600us wait between pulses
* A low bit is encoded as a 1600us wait between pulses
* Packets end with another 4400us pulse, and always have at least a 5000us wait betwen them

Signals are 2 packets wide, but the 2nd packet seems to just be a 1's complement of the first...not sure if this is a compatibility feature of the Sensibo controller, or a feature of the factory remote (which I no longer have).

I've been choosing to ignore the first packet, as the second encodes things more intuitively (e.g. '1' is 'on', and higher temperatures are encoded as larger hex values). I've yet to confirm if both codes are necessary to operate the device.

![pulseview_decode_2.png](pulseview_decode_2.png)
![pulseview_decode.png](pulseview_decode.png)

## Packet Structure

Packets are always exactly six octets long.

Packets can be one of two different formats, and only one will be sent at a time.

* State packets begin with 0xa1, and encode power state, fan mode, HVAC mode, and target temperature. These are fully idempotent and declarative, representing the intended state of the device, rather than the changes to make to it.
* Command packets begin with 0xa2, and encode specific changes that are not included in the state.

![state_packet_structure.png](state_packet_structure.png)
![command_backet_structure.png](command_packet_structure.png)

## State Packet

	Octet 1: Packet Type
		0xa1 = state (normal packet, sends modes and target temperature)
		0xa2 = command (sends a command that does NOT include modes and target)
	Octet 2: Power State + Fan mode + HVAC Mode
	MSB	7: Power
		6: Always 0
		5: Fan Mode
		4: Fan Mode
		3: Fan Mode
		2: HVAC Mode
		1: HVAC Mode
	LSB	0: HVAC Mode 
	Octet 3: Target temperature (Degress F + 34)
		62 Farenheit -> 0x60
		86 Farenheit -> 0x78
		The Senisbo API doesn't allow trying to send values outside of this - I'll have to see what happens when we try
	Octet 4, 5: Always 0xff
	Octet 6: Checksum? 

### Byte 2 Encoding

|Power|Bit 7|
|---|---|
|On|1|
|Off|0|

|Fan Mode|Bit 5|Bit 4|Bit 3|
|---|---|---|---|
|Auto|1|0|0|
|High|0|1|1|
|Med |0|1|0|
|Low |0|0|1|

|HVAC Mode|Bit 2|Bit 1|Bit 0|
|---|---|---|---|
|Fan  |1|0|0|
|Heat |0|1|1|
|Auto |0|1|0|
|Dry  |0|0|1|
|Cool |0|0|0|

## Command Packet

Display control is sent as a toggle command packet
Oscillation control is sent as an idempotent on/off command packet 

	Octet 1: Packet Type
		0xa1 = state (normal packet, sends modes and target temperature)
		0xa2 = command (sends a command that does NOT include modes and target)
	Octet 2: Power State + Fan mode + HVAC Mode
	MSB	7: Always 0
		6: Always 0
		5: Always 0
		4: Always 0
		3: Toggle Display if set
		2: Always 0
		1: Enable Oscillation if set
	LSB	0: Disable Oscillation if set
	Octet 3-5: Always 0xff
	Octet 6: Checksum? 

## Sample Data

| Function | Data |
| --- | --- |
| (Off), Cool, Fan Auto, 63				| a1 20 61 ff ff cf |
| (Off), Cool, Fan Low, 63				| a1 08 61 ff ff e7 |
| (On),  Cool, Fan Auto, 63				| a1 a0 61 ff ff cf |
| (On), Cool, Fan Low, 63				| a1 88 61 ff ff 67 |
| On, (Heat), Fan Auto, 63				| a1 a3 61 ff ff 4c |
| On, (Cool), Fan Auto, 63				| a1 a0 61 ff ff 4f |
| On, (Fan), Fan Auto, --				| a1 a4 7e ff ff 58 |
| On, (Dry), Fan Auto, 63				| a1 81 61 ff ff 6e |
| On, (Auto), Fan Auto, 63				| a1 82 61 ff ff 6d |
| On, Cool, Fan High, (63)				| a1 98 61 ff ff 7b |
| On, Cool, (Fan Auto), 63				| a1 a0 61 ff ff 4f |
| On, Cool, (Fan Strong), 63			| a1 98 61 ff ff 7b |
| On, Cool, (Fan High), 63				| a1 98 61 ff ff 7b |
| On, Cool, (Fan Med), 63				| a1 90 61 ff ff 77 |
| On, Cool, (Fan Low), 63				| a1 88 61 ff ff 67 |
| On, Cool, (Fan Quiet), 63				| a1 88 61 ff ff 67 |
| Light Toggle							| a2 08 ff ff ff 75 |
| Swing on 								| a2 02 ff ff ff 7e |
| Swing off								| a2 01 ff ff ff 7c |

## Checksum calculation

TODO
