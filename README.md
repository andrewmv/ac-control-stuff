## Experiments in Air Conditoner IR code recording

### Device: Insignia NS-AC08PWH1

IR signal duty is 38khz modulated, 50% duty cycle

![waveform.png](waveform.png)

Timings (in microseconds) - inverted (high = no light)

* packets start with a 4400us pulse, followed by 4400us wait
* Subsequent pulse are always 400us long
* A High bit is encoded as a 600us wait between pulses
* A low bit is encoded as a 1600us wait between pulses
* Packets end with another 4400us pulse, and always have at least a 5000us wait betwen them

Signals are 2 packets wide, but the 2nd packet seems to just be a 1's complement of the first...not sure if this is a compatibility feature of the Sensibo controller, or a feature of the factory remote (which I no longer have).

I've been choosing to ignore the first packet as the "inverted" one for now, and use the second, as it encodes things more intuitively.

![pulseview_decode_2.png](pulseview_decode_2.png)

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
	MSB	7: 1=power on, 0=power off
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
