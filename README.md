# NACS (SAE J3400) charging with Foccci and Clara — brainstorming

Status: brainstorming / concept. No changes to hardware or software yet.
Statements marked **[verify]** come from memory or assumptions and must be checked against the standards (SAE J3400, Tesla NACS specification, SAE J1772, DIN SPEC 70121) or by measurements.

References:
- Foccci hardware: https://openinverter.org/wiki/Foccci and https://github.com/uhi22/foccci (schematic analysis based on commit [b86fd34](https://github.com/uhi22/foccci/tree/b86fd341241138769f60162c21d3ca200bdf4482), "version 5")
- Clara software: https://github.com/uhi22/ccs32clara (code review based on commit [94de233](https://github.com/uhi22/ccs32clara/tree/94de233da14b674889938c4aa63063d4b4e65c90), version 1.00)
- ZombieVerter VCU: https://github.com/damienmaguire/Stm32-vcu (code review based on commit [b061f84](https://github.com/damienmaguire/Stm32-vcu/tree/b061f840ea05aa3e6c761953e746dabed9d9a375))

## 1. Goal

Use Foccci with Clara as the charge controller behind a NACS (SAE J3400) vehicle inlet, for
- DC charging (e.g. Tesla Supercharger V3/V4, other NACS DC chargers), and
- AC charging (e.g. Tesla Wall Connector, Mobile Connector, J3400 AC chargers, and J1772 chargers via adapter).

## 2. What NACS differs from CCS

| Topic | CCS2 / CCS1 (what Foccci does today) | NACS / J3400 |
|---|---|---|
| Contacts | separate AC pins and DC pins (DC+, DC−) | 5 contacts: 2 power pins **shared by AC and DC**, PE, CP, PP |
| Control Pilot | IEC 61851 / J1772, 1 kHz PWM, 5 % = digital communication | same electrical signal **[verify]** |
| DC protocol | HomePlug Green PHY (PLC) on CP, SLAC, DIN 70121 / ISO 15118-2 | same PLC stack, DIN 70121 / ISO 15118-2 **[verify]** |
| Proximity Pilot | CCS2: IEC 62196 resistor coding (cable current); CCS1: J1772 latch circuit | J1772-style latch circuit (150 Ω / 330 Ω + button S3, 330 Ω pull-up to 5 V and 2.7 kΩ pull-down in the vehicle) **[verify]** |
| AC | CCS2: 1 or 3 phases. CCS1: 1 phase / split phase | 1 phase / split phase, up to 80 A **[verify]** |
| Connector lock | lock actuator in inlet | lock actuator in inlet, depends on the inlet part **[verify]** |
| Charge port door | – | Tesla handle button sends an RF signal (315 MHz in North America) to open the door. Optional. **[verify]** |

The main consequence: **the same two power pins carry either up to ~1000 V DC or 120/240 V AC.** This is the main topic of the hardware concept.

## 3. Overall concept

Inlet side (signals from the NACS inlet; the inlet itself has no CAN):
```
   NACS inlet
   CP ───────────────────────────────── Foccci CP (PLC + PWM measurement + 1k3 state C)
   PP ───────────────────────────────── Foccci PP (ppvariant J1772 style)
   PE ───────────────────────────────── chassis / Foccci GND
   Power A ──┬── DC contactor 1 ──────── battery +
             └── AC contactor 1 ──────── OBC L1
   Power B ──┬── DC contactor 2 ──────── battery −
             └── AC contactor 2 ──────── OBC L2/N
   inlet temperature sensors ─────────── Foccci TEMP1..3
   lock motor + feedback ─────────────── Foccci LOCK_MOT1/2, LOCKFB
   HV sense (inlet side of contactors) ─ Foccci IN_U_HV (muehlpower board)
```
Vehicle side (Foccci interfaces into the car):
```
   Foccci SW1/SW2 ───── contactor coils (DC or AC path, depending on the option in 4.1)
   Foccci CAN ───────── VCU / BMS / OBC (charge current, voltages, AcObcState, PortState, ...)
   Foccci 12V, GND, WAKEUP, BUTTON, LEDs
```

- Clara detects the mode from the CP signal, as it does today:
  - 5 % PWM → DC (PLC, SLAC, DIN 70121 or ISO 15118-2), close DC contactors after pre-charge.
  - 8–97 % PWM → basic AC charging: lock, close AC relay, set state C, tell the OBC the current limit via CAN.
- DC contactors and AC relay are never closed at the same time (software interlock in Clara plus a hardware interlock).

## 4. Hardware concept

### 4.1 Shared power pins: AC/DC switching (main open point)

Requirements (see the note on Tesla OBCs below):
1. During DC charging the OBC input must be disconnected: the charger applies a cable check (insulation test) voltage and then up to the battery voltage onto the power pins. The OBC input is not rated for that, and its EMI filter / Y capacitors would disturb the insulation test.
2. During AC charging the DC contactors must stay open: the battery must never be connected to the mains.
3. Both paths must be checked for welded contacts before the other path is used.

Note: In the forum thread [NACS Inlet Port options](https://openinverter.org/forum/viewtopic.php?t=3978), asavage states that Tesla Gen1 OBCs tolerate HVDC on their AC input, and that Tesla cars have no contactors between charge port and OBC. This is based on the service manual, not on a measurement, and others in the thread disagree. If true, requirement 1 could be relaxed for such OBCs (the insulation test still needs checking). **[verify]**

Foccci today has two contactor drivers (SW1, SW2). Clara uses them for the two DC contactors (switched on one after the other, 300 ms apart, with economizer PWM). There is no third output for an AC relay.

Options:
- **A:** SW1 drives both DC contactors in parallel, SW2 drives the AC relay (2-pole). Needs a check: current capability of one driver for two contactor coils, loss of the 300 ms sequential start, loss of the separate welding check per DC contactor.
- **B:** The OBC switches its own input relay, commanded via CAN. Clara only tells "AC allowed / not allowed". *Usually not possible: OBCs normally have no input relays.*
- **C:** The VCU switches the AC relay, based on Clara's PortState via CAN.
- **D:** New Foccci hardware revision with a third driver (or a small add-on board).
- **E: Path selector relay.** A spare output (of Foccci or of another control unit) drives a small relay with two changeover contacts (2× NC+NO). The relay routes the two Foccci contactor outputs SW1/SW2 either to the two DC contactors or to the AC contactors (see 4.1.1).
- In options A, C and D: a hardware interlock, e.g. the coil of the AC relay is fed through a normally-closed auxiliary contact of the DC contactors and the other way round. Option E has this exclusive-OR built in.

#### 4.1.1 Option E in detail

```
                    selector relay K_SEL (2 changeover contacts)
Foccci SW1 ──── COM1 ──┬── NO1 ──── DC contactor 1 coil
                       └── NC1 ──── AC contactor 1 coil (L1)
Foccci SW2 ──── COM2 ──┬── NO2 ──── DC contactor 2 coil
                       └── NC2 ──── AC contactor 2 coil (L2/N)
spare output ────────────────────── K_SEL coil      (energized = DC path)
```

Advantages:
- Foccci keeps the on/off control of the power path in both modes, with the existing features: sequential switch-on, economizer PWM, welding detection per contactor.
- Exclusive-OR between the AC and DC path at coil level, without software. A software bug cannot energize both paths at the same time.
- No change to the Foccci hardware needed if a spare output is available.

Points to clarify:
1. **Who drives the selector?** See 4.1.2.
2. **Default position.** Proposal: not energized = AC path (NC), energized = DC path. DC sessions are short, so the selector coil is only powered for a short time. AC sessions can last many hours with no selector coil current. If Foccci resets, SW1/SW2 also turn off, so the default position is not a risk by itself.
3. **Switch only without load.** The selector must only change position when SW1 and SW2 are off (plus a settling time of some 10 ms). Otherwise the contacts break the inductive coil current and arc. Rule for Clara: outputs off → change selector → wait → outputs on.
4. **Contact rating.** The selector contacts carry the full coil current (100 % for about 200 ms, then economizer PWM). Check: inrush current of the contactor coils, 18 kHz PWM through relay contacts, and the turn-off clamp of the Foccci driver (about 40 V) acting through the selector.
5. **Stuck or welded selector contact.** This is the main remaining risk. Example: in AC mode the selector hangs in the DC position. Then "close AC contactors" closes the DC contactors, and the battery is connected to the inlet. Ways to detect it:
   - Feedback: use a third contact set, or an auxiliary contact of the contactors, on a Foccci input (e.g. LOCKFB is in use; a spare input would be needed).
   - Plausibility check with the inlet voltage (IN_U_HV): in AC mode, after closing the contactors but **before** setting state C, the inlet voltage must be about 0 V (the EVSE relay is still open). If it shows battery voltage, the DC path is closed by mistake → open everything, error. In DC mode, wrong routing would connect the OBC instead of the battery, and the charger's current would not reach the battery. Detection via the BMS current (CAN) or the voltage on the battery side.
6. **Different coils.** AC and DC contactors may need different economizer duty. Proposal: one economizer parameter per path.
7. **Welded main contacts** (not the selector) are not covered by the exclusive-OR. A welded DC contactor plus an AC session still puts the battery onto the mains. Welding detection of both paths is needed anyway (see requirement 3).
8. AC contactor: two single-pole contactors on SW1/SW2, or one 2-pole contactor on SW1 only (then SW2/NC2 stays free).

#### 4.1.2 Who drives the selector relay

Common rule for all variants: **Clara decides** the path (from the CP signal) and only switches SW1/SW2 on after the selector position is confirmed. New Clara values, e.g. `PowerPathRequest` (0 = AC / default, 1 = DC) and `PowerPathFeedback`.

**Variant 1: additional output driver on Foccci**
- **Connector pin:** In the Foccci v5 schematic ([pcb/connectors.kicad_sch](https://github.com/uhi22/foccci/blob/b86fd341241138769f60162c21d3ca200bdf4482/pcb/connectors.kicad_sch), J20 = DTM13-12PA-12PB), the GND pins are **B3, B5, B9 and B12**. All other 20 pins are used. The only way to get a new output is to re-use one of the four GND pins, e.g. **B9**. (The netlist [pcb/qca7000.xml](https://github.com/uhi22/foccci/blob/b86fd341241138769f60162c21d3ca200bdf4482/pcb/qca7000.xml) in the repo is from March 2024 and still shows the old 4.5a pinout, where B9 = LED_RED. The wiki's 4.5b column does not list B9 at all.)
- What to consider when re-using B9 as an output:
  - **Ground current:** three GND pins remain. Through them flow the board supply, the lock motor (H-bridge, several A peak), the WAKEUP output (up to about 1 A), and the LED currents. DTM size-20 contacts are rated about 7.5 A each [verify], so three are probably enough. Check whether the GND pins have different roles (e.g. a short path to PE as the reference for CP/PP, compared with power return).
  - **Harness compatibility:** no issue. Harnesses are built for each installation, so a new board version comes with its own harness. Still good practice: a short-circuit-proof driver (like the NCV8402 types used for SW1/SW2), the output off by default, and Clara enables it via a parameter (e.g. `B9Function = Unused, PowerPathSelector`) plus a new hardware-variant ID.
- **Free MCU pins (verified in [pcb/controller.kicad_sch](https://github.com/uhi22/foccci/blob/b86fd341241138769f60162c21d3ca200bdf4482/pcb/controller.kicad_sch) v5, U3 = STM32F103RET6):** not connected are PC13, PC14, PC15, PB12, PB13, PB15, PA8, PA9, PA10, PC12, PD2, PB5, PB6, PB8, PB9. PA8 (TIM1_CH1) could also do PWM, e.g. for an economizer. These pins are only reachable at the LQFP64 pins (0.5 mm pitch). A hand-wired modification is possible for a prototype, not for a product. The same goes for using pins already in use: UART4 (PC10/PC11) is used for logging, and PB3 is SWO.
- **Driver:** the same type as for SW1/SW2 (high-side, with a turn-off clamp of about 40 V, short-circuit and thermal protection), so that the selector coil, or even an AC contactor directly, can be driven.
- Pro: no additional control unit, no CAN dependency. Con: a new Foccci hardware revision, and one GND pin less.
- Short-term workaround without a new board: use one LED output (12 V high-side, current rating not stated in the wiki) to drive the selector coil, or a small relay that drives the selector. Lose one LED color.

**Variant 2: small IO-extender control unit, controlled by Foccci via CAN**
- Possible functions beyond the selector relay:
  - relay driver outputs (selector, or the AC contactors directly),
  - feedback inputs (auxiliary contacts of the contactors and of the selector) → detects a stuck selector and welded contactors,
  - AC voltage measurement on the inlet → "AC present" and AC welding check,
  - receiver for the Tesla handle button (RF, 315 MHz in North America [verify]) and charge port door actuator,
  - additional temperature inputs, charge port LED.
- CAN: uses the openinverter CAN mapping like the rest of the system. Foccci sends `PowerPathRequest`, the extender sends back its inputs.
- Safety: on CAN timeout the extender switches the selector to the default position (AC, not energized). Because SW1/SW2 stay with Foccci, a CAN failure cannot close a contactor by itself.
- Pro: no change of the Foccci board, extendable, also usable for CCS installations. Con: one more unit, more wiring, a CAN dependency in the safety chain.

**Variant 3: existing VCU** (e.g. ZombieVerter or another openinverter VCU)
- The VCU reads `PowerPathRequest` from Clara via CAN and drives the selector with one of its spare outputs, and maybe returns a feedback.
- Pro: no new hardware if the VCU has a free output. Con: depends on the VCU used, and each VCU software needs this function.

Proposal: specify the CAN interface (`PowerPathRequest` / `PowerPathFeedback`) independently of the variant, so that variants 2 and 3 are the same for Clara, and variant 1 maps the same values onto a local pin.

### 4.2 Proximity Pilot

- Foccci 5.x with `ppvariant = 4` (330 Ω pull-up to 5 V via diode, 2.7 kΩ pull-down in the inlet) already matches the J1772 vehicle-side circuit. If the inlet has no 2.7 kΩ pull-down, it has to be added (or use the internal R60 3.3 kΩ option, `ppvariant = 5`, close to the J1772 value **[verify]**).
- Expected PP resistances: about 150 Ω (plug latched), about 480 Ω (150 Ω + 330 Ω, button pressed), open (no plug) **[verify]**.
- Possible conflict: wakeup via PP (jumper JP3) needs "no pull-down in the charge port" according to the wiki, but the J1772 circuit has a 2.7 kΩ pull-down. To be checked: is PP wakeup possible, or do we use CP or the WAKEUP pin for wakeup?

### 4.3 Control Pilot

The CP circuit (1k3 resistor for state C, diode, PWM measurement, PLC coupling) is the same as for CCS. No change expected.

### 4.4 Inlet voltage measurement (IN_U_HV)

- The HV voltage sensor (e.g. muehlpower board) on the inlet side of the contactors sees DC in DC mode and 120/240 V AC in AC mode. Check that the sensor is rated and safe for AC.
- Could be used in AC mode to check that AC voltage is present, and that the DC contactors are not welded.

### 4.5 Temperature sensors

NACS inlets have temperature sensors at the power pins (type NTC or PT1000 depends on the inlet **[verify]**). Clara supports 3 sensors, NTC or PT1000. As the pins are shared, the same sensors protect AC and DC.

### 4.6 Connector lock

Depends on the chosen inlet: motor voltage and current, end-position switch or resistor feedback. Clara supports time-based and feedback-based lock control (`LockControlVariant`).

### 4.7 Charge port door (optional)

Optional RF receiver to open the door with the button on the Tesla handle. Not part of Foccci today.

### 4.8 CP and PP between Foccci and the OBC

Only one vehicle-side CP circuit and one vehicle-side PP circuit may be connected to the inlet at a time. How AC and DC are combined is a per-vehicle architecture decision. The known solutions (separate inlets, changeover, CP spoofing by a VCU, OBC owns the state-B/C resistors) and their CP and PP handling are described in [Q1 in section 10](#q1-what-do-we-do-with-cp-and-pp-when-both-foccci-and-the-obc-want-to-use-them).

## 5. Software concept (Clara)

### 5.1 What already works (from code review of Clara 1.00, commit 94de233)

- DC: HomePlug/SLAC, SDP, TCP, cable check, pre-charge, welding detection. Charging on Tesla Superchargers in Europe (CCS2) has been shown in 2023.
- DC protocols: DIN 70121 and ISO 15118-2 (DC, external payment only; no TLS, no Plug & Charge), selectable with the parameter `V2gProtocols` (DinOnly, IsoThenDin, DinThenIso, IsoOnly; default DinOnly). Details: [doc/iso15118-2.md](https://github.com/uhi22/ccs32clara/blob/94de233da14b674889938c4aa63063d4b4e65c90/doc/iso15118-2.md).
- AC basic charging: CP duty → `EvseAcCurrentLimit` up to 80 A ([ccs/acOBC.cpp](https://github.com/uhi22/ccs32clara/blob/94de233da14b674889938c4aa63063d4b4e65c90/ccs/acOBC.cpp)), state C via 1k3, lock handling, OBC state via CAN (`AcObcState`, `AcChargeControl` CAN or standalone). With `ExternalStateCResistor = 1`, Clara does not switch state C in AC mode and leaves it to the OBC (see Q1, option CP-D).
- PP resistance measurement with several hardware variants (`ppvariant` 0–5), `PlugPresent` (R < 5 kΩ).
- Two contactor outputs with sequential switch-on and economizer.

### 5.2 What is missing or has to be changed

1. **PP decoding for J1772/J3400.** Today `CableCurrentLimit` uses IEC coding (1.5 kΩ = 13 A, 680 Ω = 20 A, 220 Ω = 32 A, 100 Ω = 63 A). For NACS we need a mode which decodes "latched" (≈150 Ω) and "button pressed" (≈480 Ω), and which uses the cable current limit only from CP. With the current code, 150 Ω gives `CableCurrentLimit = 0`.
2. **Reaction to the PP button.** When the button is pressed (the user wants to unplug), AC and DC current must be ramped down and charging stopped fast **[verify the required timing in J1772/J3400]**.
3. **AC/DC path switching and interlock.** New logic: AC relay output (or CAN signal for option C), mutual exclusion with the DC contactors, welding check of both paths. For option E: control of the selector output, switching only with SW1/SW2 off, plausibility check of the selector position (inlet voltage before state C, or a feedback input), and economizer duty per path.
4. **AC temperature derating.** For DC, Clara reduces the requested current via `TempLimitedCurrent`. For AC, the reduced current must go to the OBC via CAN **[check if this already exists]**.
5. **Wakeup strategy** depending on the PP circuit (see 4.2).
6. **Configuration:** a new parameter like `InletType = CCS2 / CCS1 / NACS` which selects the PP decoding and the AC relay handling.

### 5.3 Not planned for the first step

- Plug & Charge (ISO 15118-2 with TLS and certificates). Clara supports ISO 15118-2 with external payment only.
- ISO 15118-2 AC high-level communication (Clara's ISO 15118-2 support covers DC only).
- Tesla's legacy proprietary protocol (single-wire CAN on CP, used by older Tesla vehicles on V2 Superchargers) **[verify]**.

## 6. Charger compatibility and authorization (to clarify)

- Which protocol do Tesla Superchargers (V3/V4, North America) choose at a native NACS cable, DIN 70121 or ISO 15118-2 (external payment)? Clara can offer both (`V2gProtocols`). **[verify]**
- V2 Superchargers are not opened for non-Tesla vehicles **[verify]**.
- Session start for a non-Tesla vehicle: Tesla app (choose the stall) or Plug & Charge (needs ISO 15118-2 and certificates). Does Autocharge (EVCCID = MAC) play a role?
- Voltage range: V3 up to about 500 V, V4 cabinets up to about 1000 V **[verify]**. Relevant for the battery voltage of the target vehicle.

## 7. Alternative without new development

A CCS1 inlet with a NACS→CCS1 adapter works with today's Foccci/Clara (DC). A J1772 AC charger with a NACS-to-J1772 adapter works today on a CCS1 inlet. A native NACS inlet saves the adapter but needs the points above.

## 8. Result of the concept: next Foccci hardware version

The new Foccci hardware version is an output of this concept. The requirements collected so far (baseline: Foccci v5 schematic):

| # | Requirement | Reason | Status |
|---|---|---|---|
| HW1 | Additional protected high-side output (driver like SW1/SW2), driven by a free MCU pin (e.g. PA8, PWM-capable) | Path selector relay (option E, 4.1.2 variant 1) | proposal |
| HW2 | Route the new output to connector pin B9 (today GND; B3, B5, B12 stay GND) | No free connector pin left | proposal; check the GND roles |
| HW3 | New hardware-variant resistor (new ID for Clara) | Clara must know that the new output exists | proposal |
| HW4 | PP circuit suitable for J1772/J3400 (330 Ω pull-up, 2.7 kΩ pull-down on board or in the inlet), compatible with PP wakeup | NACS proximity | open (4.2) |
| HW5 | Feedback input for the selector / contactor auxiliary contacts | Detect a stuck selector or welded contacts | open |
| HW6 | State-B resistor R26 (2k7) as an assembly option or solder jumper | Q1 option CP-D: the OBC owns the state-B/C resistors; today R26 must be removed by hand | proposal |

## 9. Open questions

1. Which vehicles and OBCs shall the concept support (battery voltage range, OBC control via CAN or via its own CP input)? This defines the requirements, not one specific car.
2. Requirements on the inlet: which NACS inlets exist on the market (Tesla OEM, aftermarket), and what do they offer (lock type, lock feedback, temperature sensors, is the 2.7 kΩ PP pull-down inside)?
3. AC/DC switching: preferred option (A, C, D or E in 4.1)? OBCs usually have no input relays, so B drops out. For E: who drives the selector relay (4.1.2: Foccci output, IO extender, or VCU), and is a feedback input available? B9 is GND on v5 (one of four GND pins: B3, B5, B9, B12) and is the candidate for a new output in a new board revision (harnesses are built for each installation, so this is no issue). Open: do the GND pins have different roles (PE reference compared with power return)?
4. Shall the new Foccci version still support CCS2/CCS1 with the same board (configuration via parameter), or is it a NACS-only variant?
5. Test possibilities: access to NACS chargers (North America), or only simulation (pyPLC as charger, AC EVSE simulator)?
6. Scope: DIN 70121 and ISO 15118-2 with external payment are available in Clara. Is Plug & Charge (TLS, certificates) needed for NACS?
7. Wakeup: via CP, WAKEUP pin, or PP?

## 10. Questions and answers from the community

Questions from the forum thread [East Coast clusters want better DC fast charging on conversions](https://openinverter.org/forum/viewtopic.php?p=95806#p95806), with the answers maintained here.

### Q1: What do we do with CP and PP when both Foccci and the OBC want to use them?

Question by ChazFisher ([forum post](https://openinverter.org/forum/viewtopic.php?p=95826#p95826)):
> What do we do with the CP signal when both FOCCI and the AC charger are expecting to interact with it? For example, the wiring for the Mitsubishi/Outlander OBC shown in the Wiki shows the CP signal to be connected directly to it. Or is that charger the outlier, and it's a reasonable constraint to proceed with FOCCI as the only interface to CP?

Answer:

**Two vehicle-side circuits in parallel are not possible, neither for CP nor for PP.**
- CP: each vehicle-side CP circuit has the diode and the 2.74 kΩ state-B resistor. Two in parallel give about 1.37 kΩ, which is out of spec: the EVSE sees roughly state C without anybody requesting charging. arber333 saw this in practice when he tapped the CP for a second device: the Outlander OBC no longer pulled in the EVSE ([Outlander charger thread, page 4](https://openinverter.org/forum/viewtopic.php?t=628&start=75)).
- PP: the same applies. Two pull-up / pull-down networks in parallel give wrong voltages, so neither device detects the plug and the cable coding correctly.

**How AC and DC are combined is a per-vehicle architecture decision.** There is no general rule. Known solutions (prefix "CP-" to distinguish them from the power path options A–E in section 4.1):

**CP-A) Separate inlets for AC and DC**
- CP and PP: each inlet has its own CP and PP. The OBC uses the AC inlet, Foccci the DC inlet. No sharing.
- For NACS: AC is then not charged via the NACS inlet, but via a second inlet (e.g. J1772).

**CP-B) Changeover relay or manual switch**, which connects the inlet either to Foccci or to the OBC. Also mentioned in the [Foccci wiki, "Using Foccci as a charge coordinator"](https://openinverter.org/wiki/Foccci).
- CP: is switched over.
- PP: **unclear, depends on the OBC.** If the OBC has its own PP input, PP has to be switched over too. Then Foccci loses plug detection (and maybe the connector lock control) while the OBC owns the inlet. The Outlander OBC has no PP pin ([wiki: Mitsubishi Outlander DCDC OBC](https://openinverter.org/wiki/Mitsubishi_Outlander_DCDC_OBC)), so for it PP can stay with Foccci.
- Open points: who decides the position (a manual switch can be forgotten; an automatic changeover needs a decision, e.g. from the PWM duty measured by Foccci); how the EVSE reacts to the short CP interruption; johu observed that the Outlander OBC only resumes after about 3 CP cycles ([Outlander charger thread, page 12](https://openinverter.org/forum/viewtopic.php?t=628&start=275)).

**CP-C) Foccci sits on the inlet, the VCU generates the CP PWM for the OBC.** This is already implemented in the ZombieVerter VCU (code at commit [b061f84](https://github.com/damienmaguire/Stm32-vcu/tree/b061f840ea05aa3e6c761953e746dabed9d9a375)):
- Foccci measures the EVSE's PWM and sends the AC current limit, the port status, the plug state and the lock state in CAN message 0x357. ZombieVerter takes the AC limit as `PilotLim` ([Foccci.cpp, handle357](https://github.com/damienmaguire/Stm32-vcu/blob/b061f840ea05aa3e6c761953e746dabed9d9a375/src/Foccci.cpp#L122-L202)).
- A ZombieVerter PWM output configured as `CpSpoof` generates a 1 kHz PWM with duty = `PilotLim` / 0.6 A per %, i.e. the same current limit as the EVSE signals ([utils.cpp, CpSpoofOutput](https://github.com/damienmaguire/Stm32-vcu/blob/b061f840ea05aa3e6c761953e746dabed9d9a375/src/utils.cpp#L622-L642)). The OBC's CP input is wired to this output.
- ZombieVerter sends CAN message 0x358 to Foccci with `enable`, `AcObcState` and `AllowUnlock` ([Foccci.cpp, Task100Ms](https://github.com/damienmaguire/Stm32-vcu/blob/b061f840ea05aa3e6c761953e746dabed9d9a375/src/Foccci.cpp#L242-L281)). Clara then locks the connector and switches state C at the inlet ([Clara acOBC.cpp](https://github.com/uhi22/ccs32clara/blob/94de233da14b674889938c4aa63063d4b4e65c90/ccs/acOBC.cpp)).
- For the Outlander OBC, ZombieVerter starts charging when Foccci is the charge interface and AC is requested ([outlanderCharger.cpp](https://github.com/damienmaguire/Stm32-vcu/blob/b061f840ea05aa3e6c761953e746dabed9d9a375/src/outlanderCharger.cpp#L106-L116)).
- PP: Foccci measures PP at the inlet and reports the plug state via CAN. The OBC needs no PP (Outlander: no PP pin). For OBCs with a mandatory PP input: **unclear**, it would need a spoofed PP as well.
- Open points:
  - Electrical circuit of the spoofed CP: the ZombieVerter PWM outputs are 12 V high-side outputs ([wiki: Zombieverter Wiring](https://openinverter.org/wiki/Zombieverter_Wiring)), so the spoofed CP is a 0/12 V square wave, not ±12 V through 1 kΩ like a real EVSE. The Outlander OBC is reported to work with a simple 12 V square wave ([wiki: Mitsubishi Outlander DCDC OBC](https://openinverter.org/wiki/Mitsubishi_Outlander_DCDC_OBC)). Other OBCs: **unclear**. No circuit is documented in the wiki.
  - Handshake: in the current ZombieVerter code, `AcObcState` is always sent as 2 (Charging), and the `enable` bit decides whether Foccci switches state C. The OBC's own "pull in EVSE" is not fed back to Foccci. To be checked whether this is the intended behavior.
  - For NACS: Foccci must decode the J1772-style PP (latched / button pressed) and stop on the button. Not yet implemented in Clara (section 5.2).

**CP-D) Foccci without its state-B resistor, the OBC controls state B/C in AC mode.** Proposed by johu as "option D" ([forum post](https://openinverter.org/forum/viewtopic.php?p=95832#p95832)).
- CP: Foccci and the OBC are both connected to CP, but only the OBC has the state-B resistor. On Foccci v5, the CP circuit is the diode D14, then R26 = 2k7 to GND (state B), R27 = 1k2 switched by Q1 (state C) and R58 = 100k to the CP measurement ([pcb/outputdrivers.kicad_sch](https://github.com/uhi22/foccci/blob/b86fd341241138769f60162c21d3ca200bdf4482/pcb/outputdrivers.kicad_sch)). For option CP-D, R26 is removed.
- AC: the OBC sees the real EVSE signal and switches state C itself. Clara does not switch state C in AC mode when the parameter `ExternalStateCResistor = 1` ([PR #84](https://github.com/uhi22/ccs32clara/pull/84), merged; [acOBC.cpp](https://github.com/uhi22/ccs32clara/blob/94de233da14b674889938c4aa63063d4b4e65c90/ccs/acOBC.cpp#L326-L333)). Foccci still measures the PWM and reports the AC current limit via CAN.
- DC: unchanged. Clara switches state C with Foccci's R27, in parallel to the OBC's state-B resistor.
- PP: not covered by option CP-D. If the OBC has no PP input (Outlander), PP stays with Foccci. For OBCs with their own PP input: **unclear**.
- Hints:
  - Check the CP input capacitance of the OBC. It adds to Foccci's own capacitance on CP (C76 = 1 nF) and could weaken the PLC signal needed for DC.
  - Hardware: today R26 has to be removed by hand. For the next Foccci version, see HW6 in section 8.

Summary:

| Solution | CP | PP | Usable for AC via the NACS inlet |
|---|---|---|---|
| CP-A) Separate inlets | separate | separate | no |
| CP-B) Changeover | switched | depends on the OBC: stays with Foccci if the OBC has no PP input, otherwise switched (unclear how) | yes |
| CP-C) CP spoofing by VCU | Foccci on the inlet, VCU feeds the OBC | Foccci; for OBCs that need PP: unclear | yes |
| CP-D) OBC owns state B/C | both on CP; Foccci without R26, OBC switches state C in AC mode | Foccci; for OBCs that need PP: unclear | yes |
