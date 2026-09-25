---
title: "Concentrador / Hub Modbus RTU (RS-485 4 Canales)"
date: 2026-09-25
type: project
status: active
tags:
  - type/hardware
  - hardware/pcb
  - status/active
aliases:
  - "Concentrador Modbus RTU"
  - "Hub RS-485 Pasivo-Activo 3.3V"
---

# Concentrador / Hub Modbus RTU (RS-485 4 Canales)

## Resumen del Proyecto
Concentrador con bus primario pasivo y 4 derivaciones activas secundarias. Diseño 100% hardware (sin microcontrolador / sin firmware), transparente al protocolo, con aislamiento galvánico y raíl único de $3.3\text{ V}$.

- **Fecha de diseño:** Septiembre 2026
- **Estado:** Especificaciones eléctricas cerradas — Listo para ruteado de PCB.
- **Relacionado con:** [[50-Journal/2026-09-25|Registro Diario]]

---

## 🛠️ 1. Especificaciones Generales

- **Alimentación de Entrada:** $24\text{ VDC}$ Nominal (Rango extendido: $16\text{ VDC}$ a $32\text{ VDC}$).
- **Raíl Interno de Potencia:** $3.3\text{ VDC}$ único (Lógica TTL + Transceptores + Fail-Safe Biasing).
- **Bus Primario (PLC / Backbone):** Pasivo (cobre directo con borneras dobles de paso).
- **Conmutación RFL Principal:** Selector mecánico DPDT para intercalar $120\,\Omega$ ($1/4\text{ W}$) o dar continuidad ($A1/B1 \rightarrow A2/B2$).
- **Ramas Secundarias:** 4 Canales RS-485 independientes.
- **Arquitectura de Canal:** Par de transceptores en espejo (Back-to-Back) por canal con conmutación de dirección automática en hardware TTL.
- **Aislamiento Galvánico:** $1.500\text{ VDC}$ mediante convertidor DC/DC aislado de $3\text{ W}$.

---

## ⚡ 2. Etapa de Alimentación ($16\text{–}32\text{ VDC} \rightarrow 3.3\text{ VDC}$)

### 2.1 Esquema de Protecciones y Anti-Inversión (Lado 24 V)
Protección pasiva mediante PMOS (Mosfet Canal P): cero caída de tensión, sin disipación de calor y alta inmunidad.

```text
 +24V In (16-32V) ──[PPTC 500mA]──┬────────────── S  [PMOS]  D ──┬────────── +Vin (Traco)
                                  │                             │
                                 ┌┴┐                           ┌┴┐
                         R1 10k  │ │                   C1 100uF│ │ (50V)
                                 └┬┘                           └┬┘
                                  │      Zener 12V              │
                                  ├───────[BZX84C12]────────────┤
                                  │  (Ánodo a G, Cátodo a S)    │
 GND In ──────────────────────────┴─────────────────────────────┴────────── -Vin (Traco)
```

- **Fusible Rearmable (PPTC):** $500\text{ mA}$ (ej. Bourns MF-MSMF050).
- **Protección Anti-Inversión:** PMOS de alta tensión (ej. AO3401A / IRFR9024N / SI2309DS).
- **Zener de Puerta:** $12\text{ V}$ ($0.5\text{ W}$) para fijar $V_{GS}$ en rango seguro ante entradas de $16\text{–}32\text{ V}$.
- **Protector Transitorios (Surge):** TVS Unidireccional SMAJ30A o SMBJ33A en paralelo con la entrada.

### 2.2 Convertidor DC/DC Aislado
- **Modelo Definitivo:** **Traco Power TMR 3-2410** (Encapsulado SIP-8).
  - **Entrada (Ultra-wide 4:1):** $9\text{ a }36\text{ VDC}$ (Cubre holgadamente el rango de $16\text{–}32\text{ VDC}$).
  - **Salida:** $3.3\text{ VDC}$ / $800\text{ mA}$ ($3\text{ W}$ de capacidad, $>60\%$ de margen sobre el consumo pico).
  - **Aislamiento:** $1.500\text{ VDC}$.
- **Filtrado a la salida de $3.3\text{ V}$:**
  - $1\times 10\,\mu\text{F}$ Tántalo / Cerámico a la salida del Traco.
  - $1\times$ Perla de Ferrita ($600\,\Omega$ @ $100\text{ MHz}$) para desacoplar el raíl de polarización de bus.
  - $1\times 100\text{ nF}$ Cerámico de desacoplo **pegado física y eléctricamente a los pines de $V_{CC}$ de cada integrado**.

### 2.3 Indicador LED de Estado (Alimentación OK)
- **LED Verde (Alimentación OK):** Conectado a la salida regulada de $3.3\text{ VDC}$ del Traco Power a través de una resistencia limitadora ($470\,\Omega$ / $1\,\text{k}\Omega$). Si el sistema está alimentado correctamente y el fusible/PMOS está íntegro, el LED verde permanece encendido.

---

## 📡 3. Arquitectura de Canales y Bus RS-485

### 3.1 Transceptor Seleccionado: SN65HVD78 (Texas Instruments)
- **Tensión de Trabajo:** $3.3\text{ VDC}$.
- **Carga de Bus:** 1/8 Unit Load (Permite hasta 200 nodos sin recargar el bus principal).
- **Protección ESD/Transitorios:** IEC 61000-4-2 ($\pm 12\text{ kV}$ contacto, $\pm 15\text{ kV}$ HBM, EFT/Burst $\pm 4\text{ kV}$).
- **Receptor:** True Fail-Safe con histéresis mejorada ($80\text{ mV}$).

### 3.2 Arquitectura Back-to-Back por Canal (2x SN65HVD78 por Canal)
Cada uno de los 4 canales cuenta con un par de transceptores en espejo:

```text
[BUS INTERNO PLC] <--(A/B)--> [SN65HVD78 A] <--(TTL Cruzado + 74HC14)--> [SN65HVD78 B] <--(A/B)--> [ESCLAVO REMOTO]
```

- **Cruce TTL:**
  - $RO_A \rightarrow DI_B$
  - $RO_B \rightarrow DI_A$
- **Control de Dirección:** Inversor 74HC14 (o transistor NPN BC847) activando $DE/RE$ dinámicamente mediante el bit de inicio (*start bit*) para evitar *latch-up* (bucle de realimentación).

### 3.3 Polarización Fail-Safe en Ramas Secundarias ($3.3\text{ V}$)
Para garantizar $+200\text{ mV}$ de tensión diferencial en reposo y máxima inmunidad al ruido industrial:
- **Pull-Up ($R_{PU}$):** **$560\,\Omega$** ($1/8\text{ W}$) de **Línea A** a **$+3.3\text{ V}$**.
- **Pull-Down ($R_{PD}$):** **$560\,\Omega$** ($1/8\text{ W}$) de **Línea B** a **$GND\_LOCAL$**.

---

## 🎛️ 4. Bus Primario Pasivo y Conmutador RFL

- **Continuidad:** Entrada $A1/B1$ conectada directamente en cobre a la salida $A2/B2$. Si la placa pierde la alimentación, **el bus principal del PLC no se interrumpe**.
- **Conmutador DPDT:**
  - **Posición Paso:** Mantiene el bus directo hacia el siguiente concentrador.
  - **Posición Final de Línea:** Corta el paso hacia $A2/B2$ e intercala la resistencia de terminación de $120\,\Omega$ ($1/4\text{ W}$) entre $A1$ y $B1$.

```text
                    BUS PRIMARIO PASIVO (Cobre)
           [A1] ───────────────────────────────── [A2]
           [B1] ───────────────────────────────── [B2]
             │                                     │
             ├──────────── [Switch DPDT] ──────────┤
             │             (Conecta 120Ω)          │
             │                                     │
             ├─────────────────────────────────────┤
             │                                     │
       Bus Interno A                         Bus Interno B
             │                                     │
   ┌─────────┼─────────┬─────────┐       ┌─────────┼─────────┬─────────┐
   │         │         │         │       │         │         │         │
[Pin A]   [Pin A]   [Pin A]   [Pin A]  [Pin B]   [Pin B]   [Pin B]   [Pin B]
Trans.1   Trans.2   Trans.3   Trans.4  Trans.1   Trans.2   Trans.3   Trans.4
```

---

## 🛡️ 5. Protecciones de Campo en Borneras Secundarias

- **Diodo TVS Dedicado RS-485:** **SM712** (SOT-23) en cada bornera de salida ($A, B, GND$).
- **Resistencias de Limitación (Opcional):** $10\,\Omega$ ($1/2\text{ W}$) en serie con las líneas A y B.

---

## 📋 6. Lista de Componentes Clave (BOM Definitiva)

| Bloque | Componente | Referencia / Valor | Encapsulado |
| :--- | :--- | :--- | :--- |
| **Fuente DC/DC** | DC/DC Aislado $3\text{ W}$ ($9\text{–}36\text{V} \rightarrow 3.3\text{V}$) | **Traco Power TMR 3-2410** | SIP-8 |
| **Anti-Inversión** | PMOS $V_{DS} \ge 40\text{V}$ | AO3401A / IRFR9024N | SOT-23 / DPAK |
| **Anti-Inversión** | Diodo Zener $12\text{ V}$ ($0.5\text{ W}$) | BZX84C12 | SOT-23 / SOD-123 |
| **Protección Ent.** | Fusible PPTC $500\text{ mA}$ | MF-MSMF050 | SMD 1812 |
| **Protección Ent.** | TVS Unidireccional $30\text{V}$ | SMAJ30A / SMBJ33A | SMA / SMB |
| **Transceptores** | Transceptor RS-485 $3.3\text{V}$ | **SN65HVD78D** ($8\text{ uds.}$) | SOIC-8 |
| **Inversores TTL** | Puertas Inversoras Hexagonal | 74HC14 / 74LVC14 ($1\text{ ud.}$) | SOIC-14 |
| **ESD Salidas** | TVS Asimétrico RS-485 | **SM712** ($4\text{ uds.}$) | SOT-23 |
| **Resistencias** | Polarización $R_{PU} / R_{PD}$ | **$560\,\Omega$** ($8\text{ uds.}$) | SMD 0805 |
| **Resistencias** | Terminación RFL | $120\,\Omega$ ($1/4\text{W}$) | SMD 1206 / PTH |
| **Conmutador** | Interruptor Selector DPDT | DPDT Deslizable / DIP Switch 2P | TH / SMD |
| **Indicación** | LED Verde (Alimentación OK) + Resistencia | LED 3mm / 0805 + $470\,\Omega$ | Through-hole / 0805 |

---

## 📋 Tareas de Diseño (Plugin Tasks)

- [ ] Definir dimensiones del PCB y ruteado de pistas de potencia (24V a 3.3V) 📅 2026-09-26
- [r] Validar esquema Back-to-Back con SN65HVD78 y 74HC14 📅 2026-09-26
- [ ] Verificar footprint de Traco TMR 3-2410 y transceptores SOIC-8 📅 2026-09-27
- [!] Comprobar polarización Fail-Safe con resistencias de $560\,\Omega$ en 3.3V 📅 2026-09-27
