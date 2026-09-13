---
name: omsi-system-reference
description: Complete self-contained dictionary of all OMSI 1 & 2 system variables, predefined local variables, system macros, system triggers, stack/math/string operators, and model/bus CFG keywords.
---

# OMSI System & API Reference

Authoritative, complete technical reference for OMSI 1.x and OMSI 2.x script and configuration engines. Contains all variable names, system macros, system triggers, stack operators, and CFG keywords without external dependencies.

## 1. Stack & Language Operators

### Registers & Stack Operations
- `s0` .. `s7` : Store float TOS (Top-of-Stack) into register 0..7. Retains value on stack.
- `l0` .. `l7` : Load float from register 0..7 onto TOS.
- `d` : Duplicate float TOS (`[a] -> [a, a]`).
- `$d` : Duplicate string TOS.
- `%stackdump%` : Debug popup displaying entire float stack. For diagnostic use only.
- `$msg` : Write string TOS to OMSI on-screen debug bar and logfile.

### Float Math & Logic Operators
- `+` : `stack1 + stack0`
- `-` : `stack1 - stack0` (earlier push minus later push)
- `*` : `stack1 * stack0`
- `/` : `stack1 / stack0` (earlier push divided by later push)
- `%` : Float modulo: `stack0 - trunc(stack1 / stack0) * stack1`
- `/-/` : Negate sign (`-x`)
- `min` / `max` : Minimum / maximum of `stack1` and `stack0`
- `abs` : Absolute value of TOS
- `trunc` : Truncate float towards 0
- `round` : Round float to nearest integer
- `sqrt` / `sqr` : Square root / square of TOS
- `exp` / `ln` : Natural exponential / logarithm
- `sin` / `cos` / `tan` : Trigonometric in radians
- `arcsin` / `arccos` / `arctan` : Inverse trigonometric (OMSI 2 only)
- `random` : Uniform pseudo-random float in range `[0.0, 1.0)`
- `&&` : Logical AND: `1` if both != 0, else `0`
- `||` : Logical OR: `1` if either != 0, else `0`
- `!` : Logical NOT: `1` if TOS == 0, else `0`
- `=` : Float equality (`1` if equal, else `0`)
- `<` : Less than: `1` if `stack1 < stack0`, else `0`
- `>` : Greater than: `1` if `stack1 > stack0`, else `0`
- `<=` : Less or equal: `1` if `stack1 <= stack0`, else `0`
- `>=` : Greater or equal: `1` if `stack1 >= stack0`, else `0`

### String Operators
- `"string"` : Push string literal to string stack.
- `$loadcount` : Push number of strings loaded onto float stack.
- `$msg` : Display TOS string in OMSI UI message banner.
- `$+` : String concatenation (`stringstack1 + stringstack0`).
- `$*` : Repeat `stringstack0` until length reaches float `stack0`.
- `$=` : String equality comparison.
- `$<` / `$> ` / `$<=` / `$>=` : String lexicographical comparison.
- `$length` : String character length of `stringstack0` -> pushes float.
- `$cutBegin` : Cut float `stack0` characters from start of `stringstack0`.
- `$cutEnd` : Cut float `stack0` characters from end of `stringstack0`.
- `$SetLengthR` : Right-align / pad / trim `stringstack0` to float `stack0` width. (Float stays on stack).
- `$SetLengthL` : Left-align / pad / trim `stringstack0` to float `stack0` width. (Float stays on stack).
- `$SetLengthC` : Center / pad / trim `stringstack0` to float `stack0` width. (OMSI 2 only; float stays on stack).
- `$IntToStr` : Truncate float `stack0` to integer and push as string.
- `$StrToFloat` : Parse `stringstack0` to float TOS.

### Memory & File Access Syntax
- `(L.S.var)` : Read system variable `var` (read-only).
- `(L.L.var)` : Read local float variable `var`.
- `(S.L.var)` : Store float TOS into local variable `var`.
- `(L.$.var)` : Read local string variable `var`.
- `(S.$.var)` : Store string TOS into local string variable `var`.
- `(C.L.const)` : Read numeric constant from constfile.
- `(C.$.const)` : Read string constant from constfile.
- `(M.L.macro)` : Invoke local script macro `macro`.
- `(M.V.macro)` : Invoke OMSI native C++ system macro `macro`.
- `(T.L.trigger)` : Fire local sound trigger.
- `"sound.wav" (T.F.trigger)` : Fire sound trigger passing explicit filename.

---

## 2. System Variables `(L.S.varname)`

All system variables are OMSI-wide and strictly read-only from scripts. Access syntax: `(L.S.varname)`.

- `(L.S.Time gap)` [seconds] (from OMSI 1.00): Time step since the last iteration (frame)
- `(L.S.GetTime)` [seconds] (from OMSI 1.00): Contains an absolute time value that counts from the start of OMSI's execution.
- `(L.S.NoSound)` [0 = Sounds enabled, 1 = Sounds disabled] (from OMSI 1.00): Sounds disabled?
- `(L.S.Pause)` [0 = Simulation running, 1 = Simulation paused] (from OMSI 1.00): Simulation Pause active?
- `(L.S.Time)` [seconds] (from OMSI 1.00): Time calculated from midnight of the current day
- `(L.S.Day)` [Take] (from OMSI 1.00): Day counted from the beginning of the month
- `(L.S.Month)` [Sweet] (from OMSI 1.00): Month
- `(L.S.Year)` [Years] (from OMSI 1.00): Year
- `(L.S.DayOfYear)` [Take] (from OMSI 1.00): Day counted from the beginning of the year
- `(L.S.mouse_x)` [Pixel] (from OMSI 1.00): X-coordinate of the mouse pointer on the screen
- `(L.S.mouse_y)` [Pixel] (from OMSI 1.00): Y-coordinate of the mouse pointer on the screen
- `(L.S.PrecipType)` [0 = none, 1 = rain, 2 = snow] (from OMSI 1.00): Precipitation type
- `(L.S.PrecipRate)` [0 = no precipitation, 1 = maximum precipitation] (from OMSI 1.00): Precipitation type
- `(L.S.coll_pos_x)` [Meter] (from OMSI 1.00): Position of the collision, x-direction relative to the vehicle origin (only when the collision trigger is activated)
- `(L.S.coll_pos_y)` [Meter] (from OMSI 1.00): Position of the collision, y-direction relative to the vehicle origin (only when the collision trigger is activated)
- `(L.S.coll_pos_z)` [Meter] (from OMSI 1.00): Position of the collision, z-direction relative to the vehicle origin (only when the collision trigger is activated)
- `(L.S.coll_energy)` [Nm] (from OMSI 1.00): Collision energy (only when the collision trigger is activated)
- `(L.S.Weather_Temperature)` [°C] (from OMSI 1.00): Outside temperature
- `(L.S.Weather_AbsHum)` [g/m³] (from OMSI 1.00): Absolute humidity
- `(L.S.AutoClutch)` [0 = off, 1 = on] (from OMSI 1.03): "Automatic clutch" option activated
- `(L.S.wearlifespan)` (from OMSI 2.00): Do not use!
- `(L.S.SunAlt)` [°] (from OMSI 2.00): Höhenwinkel der Sonne

---

## 3. Predefined Local Variables `(L.L.varname)` / `(S.L.varname)`

Vehicle-scoped and scenery-scoped variables bound to OMSI core simulation. Access with `(L.L.varname)` to read, `(S.L.varname)` to write.
Legend: `[RO]` = Read-Only by script (OMSI core writes). `[On-Demand]` = Must be explicitly declared in varlist file before OMSI will update it. `[Bidirectional]` = Handshake between OMSI and script.

### 3.1 Vehicle Predefined Variables

- `(L.L.Refresh_Strings)` : Diese Variable muss auf "1" gesetzt werden, damit die  Text-Texturen aktualisiert werden! Wird nach erfolgreicher Abarbeitung  von OMSI auf 0 gesetzt.
- `(L.L.Envir_Brightness)` [RO]  [0 = dunkel, 1 = hell]: Umgebungshelligkeit in Fahrzeugnähe
- `(L.L.StreetCond)` [RO]  [0: Trocken, 0..1: zunehmende Feuchtigkeit, 1: komplett feucht, 1..2: teilweise Pfützen, 2: Fahrzeug komplett in Pfützen]: Information über die Oberflächenbedingungen
- `(L.L.Spot_Select)`  [-1: kein Spotlight, 0: Spotlight Nr. 0, 1: Spotlight Nr. 1 usw.]: Hierüber kann gesteuert werden, welches fürs Fahrzeug vordefinierte Spotlight aktiv sein soll.
- `(L.L.Colorscheme)` [RO] : Index des gewählten Farbschemas/Anstriches
- `(L.L.M_Wheel)`  [kNm = 1000Nm]: Hierüber kann das Script einDrehmomentauf die Räder übertragen. Hierbei die Summe anzugeben, welche auf alle als angetrieben ausgewiesene Achsen wirkensoll.
- `(L.L.n_Wheel)` [RO]  [Umdrehungen pro Minute]: Mittlere Raddrehzahl des Fahrzeuges
- `(L.L.Throttle)` [RO]  [0..1]: Stellung des Gaspedals
- `(L.L.Brake)` [RO]  [0..1]: Stellung des Bremspedals
- `(L.L.Clutch)` [RO]  [0..1]: Stellung des Kupplungspedals
- `(L.L.Brakeforce)`  [N]: Hierüber kann das Script die Gesamtbremskraft des  Fahrzeuges vorgeben, welche auf die Räder wirkt. Sollte nicht  gleichzeitig mitAxle_Brakeforce...verwendet werden.
- `(L.L.Velocity)` [RO]  [km/h]: Geschwindigkeit, welche sich über die Raddrehung ergibt, entspricht der Tachoanzeige
- `(L.L.Velocity_Ground)` [RO]  [km/h]: Geschwindigkeit, welche direkt relativ zum Grund  gemessen wird, berücksichtigt kein Radschleudern oder -blockieren,  entspricht der Anzeige auf einem GPS
- `(L.L.tank_percent)`  [0..1]: Tankinhalt, nur für die Anzeige auf der roten  Informationsleiste. Die Betankung und Simulation eines leeren Tanks  geschieht über einen Trigger bzw. muss im Script programmiert werden.
- `(L.L.kmcounter_km)` [RO]  [km]: Stand des Kilometerzählers (nur ganze km)
- `(L.L.kmcounter_m)` [RO]  [m]: Stand des Kilometerzählers (zusätzliche Meter, läuft stets nur von 0m bis 999m; hinzu kommen die Kilometer vonkmcounter_km)
- `(L.L.relrange)` [RO]  [---]: Zurückgelegte, relative Strecke. Wurde nur testweise für pfadfixierte Fahrzeuge eingeführt. Nicht benutzen!
- `(L.L.Driver_Seat_VertTransl)` [RO]  [m]: Einfederung des federnden Fahrersitzes
- `(L.L.Wheel_Rotation_0_L / ~_R ... ~_3_L / _R)` [RO]  [rad]: Drehwinkel des jeweiligen Rades, z.B. für die Animation
- `(L.L.Wheel_RotationSpeed_0_L / ~_R ... ~_3_L / _R)` [RO]  [Umdrehungen pro Minute]: Drehzahl des jeweiligen Rades
- `(L.L.Axle_Suspension_0_L / ~_R ... ~_3_L / _R)` [RO]  [m]: Einfederungsweg des jeweiligen Rades
- `(L.L.Axle_Steering_0_L / ~_R ... ~_3_L / _R)` [RO]  [rad]: Lenkwinkel des jeweiligen Rades
- `(L.L.Axle_Springfactor_0_L / ~_R ... ~_3_L / _R)`  [0 = keine, 1 = normale, 2 = doppelte Federkraft]: Faktor, mit dem radweise die Federstärke angepasst werden kann, z.B. um Luftfederungen zu simulieren.
- `(L.L.Axle_Brakeforce_0_L / ~_R ... ~_3_L / _R)`  [N]: Ermöglicht Setzen der Bremskraft pro Rad. Sollte nicht gleichzeitig mitBrakeforceverwendet werden!
- `(L.L.Axle_SurfaceID_0_L / ~_R ... ~_3_L / _R)` [RO] [On-Demand]  [Oberflächen-Codes]: Oberflächenbeschaffenheit unter dem Rad
- `(L.L.Debug_0 ... _5)`  [bliebig]: Debug-Variablen. Können im Debug-Modus in der Informationsleiste angezeigt werden, um auf diese Weise die Scripts zu testen.
- `(L.L.A_Trans_X ... _Z)` [RO]  [m/s²]: Beschleunigungen im Fahrzeug, die durch die Fahrzeugbewegungen ausgelöst werden (beim Bremsen oder bei Kurvenfahrten usw.)
- `(L.L.AI_Blinker_L, ~_R)`  [0: aus, 1: ein]: Linker oder rechter Blinker aktiv
- `(L.L.AI_Light)`  [0: Licht aus, 0.5: Standlicht an, 1: Fahrlicht an, 2: Fernlicht/Lichthupe an]: Fahrlicht, Standlicht, Lichthupe
- `(L.L.AI_Interiorlight)`  [0: aus, 1: an]: Innenbeleuchtung. Hierüber erfahren die einsteigenden Fahrgäste auch, ob es im Bus zu dunkel ist (sodass sie meckern dürfen).
- `(L.L.AI_Brakelight)` [RO]  [0: aus, 1: an]: Nur für KI-Fahrzeuge: Script soll Bremslicht einschalten
- `(L.L.AI_Engine)` [RO]  [-1: Motor ausschalten!, 0: egal/nicht-KI, 1: Motor einschalten!]: Nur für KI-Fahrzeuge: Aufforderung zum Ein- oder Ausschalten des Motors
- `(L.L.AI_target_index)` [RO] : Dient der Übergabe des Sollwertes des einzustellenden  Zielschildes bei Aufruf des Menüs oder beim Umschildern der KI-Busse.  Entspricht der Reihenfolge in der Hof-Datei; erster Eintrag = 0
- `(L.L.target_index_int)` : Über diese Variable setzt das Script, welches  Zielschild am Bus zusehen ist und steuert hierüber insbesondere die  Fahrgäste. Entspricht wieAI_target_indexdem Index der Reihenfolge in der Hof-Datei.
- `(L.L.AI_Scheduled_AtStation)`  [-1: Bus abfahrbereit machen, 0: Bus ist abfahrbereit, 1: Türen freigeben/öffnen]: Nur KI-Fahrzeuge, bidirektionale Kommunikation: Bei  Erreichen einer Station setzt OMSI den Wert auf 1, sodass das Script  mitgeteilt bekommt, dass die Türen geöffnet werden sollen. Wenn sich der  Bus abfahrbereit machen soll, setzt OMSI den Wert auf -1. Wenn der Bus  abfahrbereit ist (Türen geschlossen usw.), dann setzt das Script den  Wert auf 0, sodass die KI den Bus weiterfahren lassen kann.
- `(L.L.AI_Scheduled_AtStation_Side)` [RO] [On-Demand]  [0: rechts, 1: links, 2: beidseitig]: Gibt dem Script an, auf welcher Seite die Türen geöffnet werden können/sollen (insbesondere bei Schienenfahrzeugen)
- `(L.L.AI)` [RO]  [0: nein, 1: ja]: Hierüber erfährt das Script, ob das Fahrzeug von der AI gesteuert wird.
- `(L.L.PAX_Entry0_Open ... ~7_Open)`  [0: geschlossen, 1: offen]: Teilt OMSI mit, ob die jeweiligen Eingänge offen oder geschlossen sind.
- `(L.L.PAX_Exit0_Open)`  [0: geschlossen, 1: offen]: Teilt OMSI mit, ob die jeweiligen Ausgänge offen oder geschlossen sind.
- `(L.L.PAX_Entry0_Req ... ~7_Open)`  [0: keine Anforderung, 1: Anforderung]: Mindestens ein Fahrgast fordert den Einstieg durch diesen Eingang an.
- `(L.L.PAX_Exit0_Req)`  [0: keine Anforderung, 1: Anforderung]: Mindestens ein Fahrgast fordert den Ausstieg durch diesen Ausgang an ("Haltewunsch").
- `(L.L.GivenTicket)`  [-1: kein Ticket, 0: Tickettyp 0, 1: Tickettyp 1, ...]: Teilt OMSI mit, ob der ggf. vorhandene Fahrscheinautomat ein Ticket ausgegeben hat.
- `(L.L.humans_count)` [RO] : Anzahl der Fahrgäste im Fahrzeug
- `(L.L.FF_Vib_Period)`  [s]: Setzt die Force-Feedback-Vibrationsperiode im Lenkrad
- `(L.L.FF_Vib_Amp)`  [0...1]: Setzt die Force-Feedback-Vibrationsamplitude im Lenkrad
- `(L.L.Snd_OutsideVol)`  [0: kaum hörbar, 1: wie draußen]: Teilt OMSI mit, wie stark der Außensound im Innenraum hörbar ist; verändert sich bspw. wenn eine Tür geöffnet wird.
- `(L.L.Snd_Microphone)`  [0: aus, 1: an]: Soll das (Hardware-)Mikrofon aktiv sein?
- `(L.L.Snd_Radio)`  [0: aus, 1: an]: Soll das (Internet-)Radio laufen?
- `(L.L.Cabinair_Temp)`  [°C]: Innenraumtemperatur für die Beurteilung durch die Fahrgäste
- `(L.L.Cabinair_absHum)`  [g/m³]: Absolute Luftfeuchtigkeitim Innenraum für die Beurteilung durch die Fahrgäste
- `(L.L.Cabinair_relHum)` [RO]  [0 = 0%, 1 = 100%]: Relative Luftfeuchtigkeitim Innenraum, wird durch OMSI automatisch über die absolute Feuchtigkeit und die Innentemperatur berechnet.
- `(L.L.PrecipRate)` [RO]  [0 = kein, 1 = maximal]: Niederschlagsrate
- `(L.L.PrecipType)` [RO]  [0 = kein, 1 = Regen, 2 = Schnee]: Niederschlagstyp
- `(L.L.Dirt_Norm)` [RO]  [0 = sauber, 1 = total verdreckt]: allgemeiner Verdreckungszustand (Windschutzscheiben können sauberer sein und werden nur vom Script simuliert)
- `(L.L.DirtRate)` [RO]  [Verdreckung / s, wobei eine Verdreckung = 1 bedeutet, dass die Verdreckung maximal ist.]: aktuelle Verdreckungsrate
- `(L.L.schedule_active)` [RO]  [0: nein, 1: ja]: Ist gerade ein Fahrplan aktiv? (Bspw. für Einblendung der Fahrplankarte)
- `(L.L.train_frontcoupling)` [RO]  [0: nein, 1: ja]: Ist an der vorderen Kupplung etwas angekuppelt? (Eventuell spätere Benutzung zum Abkuppeln)
- `(L.L.train_backcoupling)` [RO]  [0: nein, 1: ja]: Ist an der hinteren Kupplung etwas angekuppelt? (Eventuell spätere Benutzung zum Abkuppeln)
- `(L.L.train_me_reverse)` [RO]  [0: nein, 1: ja]: Steht dieses Fahrzeug rückwärts zum gesamten Fahrzeugverband?
- `(L.L.TrafficPriority)`  [0: nein, 1: ja]: Diese Variable wird vom Fahrzeug gesetzt, wenn dieses  Fahrzeug (z.B. Feuerwehr oder Polizei) im Einsatz ist und Vorrang vor  allen anderen Verkehrsteilnehmern genießt.
- `(L.L.wearlifespan)` [RO]  [10: sehr gute Wartung, 1: normale Wartung, 0.1: schlechte Wartung, 0.01: sehr schlechte Wartung, 1500000: "unendlich"]: Faktor, welcher beim Initialisieren von  Verschleißteil-Lebensdauern über den Durchschnittswert multipliziert  wird. Hierrüber wirkt sich die Einstellung in den Optionen über dem  Wartungszustand auf die Lebensdauern aus.
- `(L.L.articulation_#_alpha / ~_beta)` [RO] [On-Demand]  [°]: Winkel, um den das #. Gelenk um die Hochachse (alpha) bzw. Querachse (beta) geknickt ist
- `(L.L.boogie_#_wheel_at_limit)` [RO] [On-Demand]  [0 = genau mittig auf dem Gleis, -1/1: ganz an einer der beiden Kanten - Quietschen ist das Resultat]: Wie weit läuft das Drehgestell an der Kante?
- `(L.L.boogie_#_invradius)` [RO] [On-Demand]  [1/m]: Welchen inversen Kurvenradius hat die Kurve am Drehgestell?
- `(L.L.contactshoe_#_rail_pos_x / ~_y)` [RO] [On-Demand]  [m]: An welcher Position befindet sich die Stromschiene am Gleis?
- `(L.L.contactshoe_#_rail_index)` [RO] [On-Demand]  [index]: Index der Stromschiene, -1, falls keine vorhanden ist
- `(L.L.contactshoe_#_volt_rail)` [RO] [On-Demand]  [V]: Spannung an der Stromschiene
- `(L.L.contactshoe_#_volt_veh)` [RO] [On-Demand]  [V]: Der durch diese Stromschiene ins Fahrzeug gelangte Spannung
- `(L.L.contactshoe_#_freq)` [RO] [On-Demand]  [Hz, 0: Gleichspannung]: Die Frequenz der an dieser Stromschiene liegendenSpannung

### 3.2 Predefined String Variables `(L.$.varname)` / `(S.$.varname)`

- `(L.$.ident)` [RO] : Enthält das Kraftfahrzeugkennzeichen dieses Fahrzeugs (z.B. "B-V 3503")
- `(L.$.number)` [RO] : Enthält die Fahrzeugnummer (z.B. "3503")
- `(L.$.act_route)` : (keine Verwendung)
- `(L.$.act_busstop)` [RO] : Wird zusammen mit dem internen Triggerai_scheduled_setbusstopverwendet: Enthält dann den Namen der im Fahrplansystem aktiven Haltestelle.
- `(L.$.SetLineTo)` [RO] : Wird zusammen mit dem internen Triggerai_scheduled_settargetverwendet: Enthält die einzustellende Liniennummer (oder Symbol) (bei  Verwendung vom Zielschild-Menü oder beim Umschildern von KI-Bussen)
- `(L.$.yard)` [RO] : Name der Hofdatei des Fahrzeuges, bspw. damit das Scriptsystem den Dateinamen der Zielcode-Tabelle generieren kann.
- `(L.$.file_schedule)` [RO] : Enthält den Dateinamen der Fahrplan-Bitmap, die zu der von diesem Fahrzeug gefahrenen Strecke gehört.

### 3.3 Scenery Object Predefined Variables

- `(L.L.NightLightA)` [RO]  [0 = aus, 1 = an]: Ist die Nachtbeleuchtung aktiv?
- `(L.L.InUse)` [RO]  [0 = inaktiv, 1 = aktiv]: Ist das Objekt "aktiv"? Die Sporthallen sind bspw. nur  an Schultagen vormittags "aktiv", sodass die Soundeffekte an diese  Variable gekoppelt werden können - auf diese Weise wird verhindert, dass  die Soundeffekte auch in den Ferien oder nachts zu hören sind.
- `(L.L.TrafficLightPhase)` [RO]  [0..2 = rot, 3..5 = rot-gelb, 6..8 = grün, 9..11 = gelb, sonst aus]: Hierüber wird Ampelobjekten die anzuzeigende Ampelphase mitgeteilt.
- `(L.L.TrafficLightApproach)` [RO]  [0 = keine Anforderung, 1 = Anforderung]: Hierüber kann das Script eine Ampelanforderung erkennen und bspw. eine Kennleuchte aktiviert werden.
- `(L.L.Colorscheme)` [RO]  [(-)]: (veraltet)
- `(L.L.Signal)` [RO]  [individuell]: Hierüber erhalten Signale ihr anzuzeigendes Signalbild
- `(L.L.NextSignal)` [RO]  [individuell]: Hierüber erhalten Signale mit Vorsignalfunktion das anzuzeigende Signalbild des nächsten Signals
- `(L.L.Refresh_Strings)` : Diese Variable muss auf "1" gesetzt werden, damit die  Text-Texturen aktualisiert werden! Wird nach erfolgreicher Abarbeitung  von OMSI auf 0 gesetzt.
- `(L.L.Switch)` : Mit dieser Variable kann die Weiche, die im Objekt  eingebaut ist, gestellt werden. Hierfür muss der Wert der Variable auf  den mit [switchdir] angegebenen Wert des gewünschten Pfades gesetzt  werden.

### 3.4 Human / Passenger Predefined Variables

- `(L.L.LastMovedDist)` [RO] [m]: Entfernung, die der Mensch seit dem letzten Durchlauf zurückgelegt hat. Wurde für die alten Animationen gebraucht.
- `(L.L.PAX_State)` [RO] [0: stehen, 1: gehen, 2: sitzen]: Zustand des Menschens. Wurde für die alten Animationen gebraucht.
- `(L.L.HeightOfSeat)` [RO] [m]: Höhe des Sitzes. Diese Eigenschaft wird in der  Sitzposition gebraucht, um die Höhe der Füße korrekt gesetzt. Wurde für  die alten Animationen gebraucht.
- `(L.L.Colorscheme)` [RO]: Tauschtextur-Index ("Farbschema") des Menschens.

---

## 4. System Macros `(M.V.macroname)`

OMSI internal C++ routines called with `(M.V.macroname)`. Note strict stack input and output order!

### 4.1 Script Texture Macros (Dynamic Drawing & Display)

- `(M.V.STNewTex)`
  - In: `Stack0: Texture index, corresponds to the [scripttexture] order in model.cfg` | Out: `None`
  - Purpose: Initialize script texture
- `(M.V.STLock)`
  - In: `Stack0: Texture Index` | Out: `None`
  - Purpose: "Lock" script texture, i.e., allow read and write access.
- `(M.V.STUnlock)`
  - In: `Stack0: Texture Index` | Out: `None`
  - Purpose: "Unlock" script texture, i.e., revoke read and write access and enable rendering.
- `(M.V.STFilter)`
  - In: `Stack0: Texture Index` | Out: `None`
  - Purpose: Mipmaps are generated, meaning sub-variants at smaller  resolutions. This usually happens after unlocking, but never between  locking and unlocking. If it doesn't happen, display errors occur at  greater distances ("oversharpening," flickering).
- `(M.V.STSetColor)`
  - In: `Stack4: Texture IndexStack3: AlphaStack2: RotStack 1: GreenStack0: Blue` | Out: `None`
  - Purpose: Sets the color with which further operations should be performed.
- `(M.V.STDrawPixel)`
  - In: `Stack2: Texture IndexStack1: XStack0: The` | Out: `None`
  - Purpose: Sets the color of the pixel specified by X and Y to the preselected color value.
- `(M.V.STDrawRect)`
  - In: `Stack4: Texture IndexStack3: X1Stack2: Y1Stack1: X2Stack0: Y2` | Out: `None`
  - Purpose: Draws a filled rectangle in the selected color between the specified pairs of coordinates.
- `(M.V.STTextOut)`
  - In: `Stack5: Texture IndexStack4: X1Stack3: Y1Stack2: Font-IndexStack1: Full color (or not?)Stack0:Lock pixelStringStack0: Text to be written` | Out: `None`
  - Purpose: Writes text onto the texture. The font index can be retrieved usingGetFontIndex. "Full color" indicates that the text should be written in the  currently selected color; otherwise, the symbols will be written in the  font color.
- `(M.V.GetFontIndex)`
  - In: `StringStack0: Font-Name` | Out: `Stack0: Font-Index`
  - Purpose: For use with STTextOut
- `(M.V.TextLength)`
  - In: `Stack0: Font-IndexStringStack0: Text to be checked` | Out: `Stack0: Length in pixels`
  - Purpose: Calculates the length of the submitted text in the selected font.
- `(M.V.STReadPixel)`
  - In: `StringStack2: Texture IndexStringStack1: XStringStack0: Y` | Out: `None`
  - Purpose: Sets the preselected color to the color of the specified pixel.
- `(M.V.STCopyColor)`
  - In: `Stack1: of-texture-indexStringStack0: by-texture-index` | Out: `None`
  - Purpose: The preselected color of the "after" texture is set to the preselected color of the "from" texture.
- `(M.V.STLoadTex)`
  - In: `Stack0: Texture IndexStringStack0: Texturname` | Out: `None`
  - Purpose: Draws the texture specified by filename onto the script texture.
- `(M.V.STGetR / G / B / A)`
  - In: `Stack0: Texture Index` | Out: `Stack0: Color value Red/Green/Blue/Alpha`
  - Purpose: Returns the currently selected color value.

### 4.2 Vehicle & Timetable System Macros

- `(M.V.GetRouteIndex)`
  - In: `Stack0: Route-Code` | Out: `Stack0: Route-Index`
  - Purpose: Search the farm file for the route index corresponding to the given route code.
- `(M.V.GetBusstopString)`
  - In: `Stack1: Stop index, Stack0: String index` | Out: `Stringstack0: Stop string content`
  - Purpose: Returns the stack0th string of the stack1th stop from the hof file. Both indices arezero-based.
- `(M.V.GetTerminusString)`
  - In: `Stack1: Terminus-Index, Stack0: String-Index` | Out: `Stringstack0: End stop string content`
  - Purpose: Returns the stack0th string of the stack1th terminus from the hof file. Both indices arezero-based.
- `(M.V.GetRouteTerminusIndex)`
  - In: `Stack0: Route index (zero-based)` | Out: `Stack0: Terminus index (zero-based)`
  - Purpose: Returns the terminus/end-of-line index associated with the specified route.using GetTerminusStringThe name of the end-of-line can then be retrieved  .
- `(M.V.GetTerminalIndex)`
  - In: `Stack0: Terminus-Code` | Out: `Stack0: Terminus-Index`
  - Purpose: Search the hof file for the term index belonging to the given term code.
- `(M.V.GetTerminusCode)`
  - In: `Stack0: Terminus-Index` | Out: `Stack0: Terminus-Code`
  - Purpose: Returns the terminus code in the hof file corresponding to the given terminus index and is therefore the inverse function ofGetTerminusIndex.
- `(M.V.GetBusstopCount)`
  - In: `Stack0: Route-Index` | Out: `Stack0: Number of stops`
  - Purpose: Returns the number of stops on the route selected by  index. Since this is a count, it is not zero-based – if the route has  two stops, the result of this function will also be "2".
- `(M.V.GiveChangeCoin)`
  - In: `Stack0: Coin Index` | Out: `None`
  - Purpose: Ensures that the coin with index Stack0 is ejected, e.g. in the money changer script.
- `(M.V.GetRouteBusstopIdent)`
  - In: `Stack1: Route-Index, Stack0: Route-Busstop-Index` | Out: `Stringstack0: Bus stop identifier`
  - Purpose: Provides the identifier of the Stack0'th bus stop in route Stack1. Again, zero-based indices.
- `(M.V.GetBusstopIndex)`
  - In: `Stringstack0: Bus stop identifier` | Out: `Stack0: Index of the bus stop`
  - Purpose: Search for the index of the bus stop identified by the identifier.
- `(M.V.GetTTBusstopCount)`
  - In: `None` | Out: `Stack0: Number of stops`
  - Purpose: Provides the number of stops in the current timetable.
- `(M.V.GetTTBusstopIndex)`
  - In: `None` | Out: `Stack0: Index of the current stop`
  - Purpose: Provides the index of the current stop in the current timetable.
- `(M.V.GetTTTerminusIndex)`
  - In: `None` | Out: `Stack0: Index of the final stop`
  - Purpose: Provides the index of the final stop in the current timetable.
- `(M.V.GetTTLineString)`
  - In: `None` | Out: `Stringstack0: Line number`
  - Purpose: Provides the line number of the current timetable.
- `(M.V.GetTTDelay)`
  - In: `None` | Out: `Stack0: Delay in seconds`
  - Purpose: Does it deliver the current delay according to the active timetable?
- `(M.V.GetTTBusstopName)`
  - In: `Stack0: Index of the stop in the current timetable` | Out: `Stringstack0: Name of the stop`
  - Purpose: Returns the name of the stop that is located at position 0th in the current timetable.
- `(M.V.GetTTBusstopDep)`
  - In: `Stack0: Index of the stop in the current timetable` | Out: `STack0: Departure time in seconds`
  - Purpose: Provides the departure time at the stop that is located at the 0th position in the current timetable.
- `(M.V.GetTTBusstopArr)`
  - In: `Stack0: Index of the stop in the current timetable` | Out: `STack0: Arrival time in seconds`
  - Purpose: Provides the arrival time at the stop that is located at the 0th position in the current timetable.
- `(M.V.GetTicketName)`
  - In: `Stack0: Ticket-Index` | Out: `Stringstack0: Ticket-Name`
  - Purpose: Returns the name of the ticket type indexed via index (zero-based)
- `(M.V.GetTicketValue)`
  - In: `Stack0: Ticket-Index` | Out: `Stack0: Ticket price`
  - Purpose: Provides the price of the ticket type indexed via index (zero-based)
- `(M.V.GetHumanCountOnPathLink)`
  - In: `Stack0: Pathlink-Index` | Out: `Stack0: Number of passengers`
  - Purpose: Returns the number of passengers currently on the specified section of the path.
- `(M.V.NrSpecRandom)`
  - In: `Stack0: Seed` | Out: `Stack0: Random value`
  - Purpose: Returns a random value specific to the vehicle  identification number (VIN). The key feature is that the combination of  seed and VIN always yields the same random value! If you want to use  several different random values ​​for a specific vehicle (for different  purposes), you simply use different seed values. Which seed value is  ultimately used is irrelevant (since it's a random value).
- `(M.V.GetHeightAbovePoint)`
  - In: `Stack2: xStack1: andStack0: from` | Out: `Stack0: Height above ground`
  - Purpose: Returns the height above the ground (terrain, road or  sidewalk), measured along the negative, local Z-axis of the vehicle,  starting from the point defined by x, y and z (local coordinate system).
- `(M.V.GetDepotStringGlobal)`
  - In: `Stack0: String-Index` | Out: `StringStack0: Global string from farm file`
  - Purpose: Returns a string from the currently configured hof  file; these are defined in the hof file using the [global_strings]  command; the string index then corresponds to the zero-based position in  the list.
- `(M.V.GetHumanCountOnSeat)`
  - In: `Stack0: Index of seat/standing position (zero-based)` | Out: `Stack0: Occupied/Unoccupied`
  - Purpose: Provides the current seat occupancy of the verified [passpos] entry.

### 4.3 Scenery Object System Macros (Bus Stop Displays)

- `(M.V.GetArrBusLine)`
  - In: `Stack0: Index of the incoming bus` | Out: `Stringstack0: Line number of the arriving bus`
  - Purpose: For this to work, the scenery object must be linked to a  bus stop via the "Parent to..." function. This function then writes the  line number of the bus arriving at the position specified by the index  (zero-based!) to Stringstack0.
- `(M.V.GetArrBusTerminus)`
  - In: `Stack0: Index of the incoming bus` | Out: `Stringstack0: Destination of the arriving bus`
  - Purpose: It works likeGetArrBusLineand provides the final stop of the bus.
- `(M.V.GetArrBusTimeDiff)`
  - In: `Stack0: Index of the incoming bus` | Out: `Stack0: Time until bus arrives at stop`
  - Purpose: Works likeGetArrBusLineand provides the time (in seconds) until the bus selected by index arrives at the stop.

---

## 5. System Triggers `{trigger:name}`

Simulation event hooks called by OMSI directly into `{trigger:name}` blocks.

- `collision` (from OMSI 1.00): Triggered for every collision. The`coll_~`variables are set accordingly beforehand.
- `int_request_stop_` (from OMSI 1.00): Passengers use this button to tell the bus script they  want to get off. To ensure the doors stay open if necessary, passengers  usually press the button several times!
- `railbond_#` (from OMSI 1.00): Triggering the rail joint sounds on rail vehicles
- `ai_scheduled_settarget` (from OMSI 1.00): The script is instructed to reset the displayed line  and the destination sign. The line string is specified via the vehicle  variable `SetLineTo`, and the destination index in the hof file is specified via`AI_target_index`.
- `ai_scheduled_setbusstop` (from OMSI 1.00): The script is instructed to update the bus stop, for  example, for the interior display. The new bus stop is communicated to  the script via the vehicle variable `act_busstop`.
- `malfunction_gettime` (from OMSI 2.00): The script is instructed to calculate the duration of  all necessary repairs (excluding travel time!); the result must be at  the top of the stack at the end.
- `malfunction_reset` (from OMSI 2.00): Instruction to the script to perform all repairs

---

## 6. Model & Bus Configuration Keywords (`*.cfg` / `*.bus`)

Key configuration tags used in vehicle and scenery definitions to wire meshes, materials, text textures, mouse events, and cameras.

- `[add_camera_driver]`: 
- `[add_camera_pax]`: 
- `[boundingbox]`: 
- `[char]`: 
- `[illumination_interior]`: 
- `[interiorlight]`: 
- `[light_enh]`: 
- `[light_enh_2]`: 
- `[matl]`: 
- `[mesh]`: 
- `[mouseevent]`: 
- `[newfont]`: 
- `[rendertype]`: 
- `[set_camera_std]`: 
- `[texttexture]`: 
- `[texttexture_enh]`: 
- `[useTextTexture]`: 
- `[VFDmaxmin]`: 
- `[view_schedule]`: 
- `[view_ticketselling]`: 
- `[viewpoint]`: 
- `[visible]`: 
