# Projeto Ender‑3 + Klipper — Mapa de Ligações (Hardware)

> **Resumo do hardware actual**
> - Base: **Creality Ender‑3** (cama aquecida + vidro)
> - Mainboard: **BIGTREETECH SKR Mini E3 V3.0** (STM32G0B1, TMC2209 UART)
> - Extrusor/Hotend: **Creality Sprite** (direct‑drive)
> - Nivelamento: **BLTouch/3DTouch** (clone) — offsets medidos: **X = −41 mm**, **Y = +42 mm**
> - Sensor de filamento: **interruptor/runout** 3 pinos
> - Aceleração/Input Shaper: **ADXL345** ligado ao **Raspberry Pi Zero 2 W** (SPI)
> - Host: **Raspberry Pi Zero 2 W** com Klipper + Moonraker + Fluidd, ligação USB à SKR

---

## 1) Ligações principais na SKR Mini E3 V3.0

### Endstops
| Função | Porta física | Pino Klipper |
|---|---|---|
| X‑MIN | **X‑STOP** | `PC0` |
| Y‑MIN | **Y‑STOP** | `PC1` |
| Z‑MIN | **Z‑STOP** *(não usado com BLTouch)* | `PC2` |
| Z por sonda | — | `probe:z_virtual_endstop` |

### BLTouch / 3DTouch
Usar o conector **PROBE** de 5 pinos da SKR (ligar os 5 fios). Config em Klipper:
```ini
[bltouch]
sensor_pin: ^PC14      # fio branco (sinal da sonda) – com pull‑up
control_pin: PA1       # fio amarelo (servo)
x_offset: -41
y_offset: 42
# z_offset: (calibrar)
```
E no Z:
```ini
[stepper_z]
endstop_pin: probe:z_virtual_endstop
```

### Sensor de filamento (runout)
Ligar ao **E0‑STOP** (3 pinos: S/GND/V). Config inicial (ajuste do `!` pode ser necessário conforme o sensor abre/fecha):
```ini
[filament_switch_sensor filamento]
switch_pin: ^PA4       # E0‑STOP (puxador interno + pull‑up). Adicionar "!" se lógica invertida.
pause_on_runout: True
```
Teste no terminal: `QUERY_FILAMENT_SENSOR SENSOR=filamento`

### Ventoinhas (3 PWM)
| Porta na placa | Uso recomendado | Pino Klipper | Secção típica |
|---|---|---|---|
| **FAN0** | Ventoinha de peça (part cooling) | `PC6` | `[fan]` |
| **FAN1** | Ventoinha do hotend (dissipador) | `PC7` | `[heater_fan hotend_fan]` |
| **FAN2** | Ventoinha caixa/controlador | `PB15` | `[controller_fan]` ou `[fan_generic]` |

Exemplo de configuração:
```ini
# Part cooling (controlada pelo slicer / M106)
[fan]
pin: PC6
kick_start_time: 0.5

# Hotend heatsink automática >50 ºC
[heater_fan hotend_fan]
pin: PC7         # se ligaste no FAN2, usa PB15 em vez de PC7
heater: extruder
heater_temp: 50.0
max_power: 1.0

# Ventoinha da caixa ao mover motores (opcional)
[controller_fan caixa]
pin: PB15        # se FAN2 for a caixa
idle_timeout: 60
```

### Aquecimentos e termístores
| Elemento | Porta | Pino Klipper |
|---|---|---|
| Hotend — **heater** | HE0 | `PC8` |
| Hotend — **sensor** | TH0 | `PA0` |
| Cama — **heater** | BED | `PC9` |
| Cama — **sensor** | TB | `PC4` |

> **Nota:** Ajustar `sensor_type` dos termístores conforme o que tens (ex.: `ATC Semitec 104GT-2`, `Generic 3950`, etc.).

### Motores (passo/dir/habilitar — para referência)
| Eixo | STEP | DIR | ENABLE |
|---|---|---|---|
| X | `PB13` | `PB12` | `PB14` |
| Y | `PB10` | `PB2` | `PB11` |
| Z | `PB0` | `PC5` *(inverter com `!` conforme necessário)* | `PB1` |
| Extrusor (E) | `PB3` | `PB4` | `PD1` |

---

## 2) ADXL345 no Raspberry Pi (SPI)

### Ligações sugeridas (SPI0 do Pi)
| ADXL345 | Raspberry Pi (GPIO) |
|---|---|
| VCC | **3V3** (pin 1) |
| GND | **GND** (pin 6) |
| SCL/SCLK | **GPIO11 SCLK** (pin 23) |
| SDA/MOSI | **GPIO10 MOSI** (pin 19) |
| SDO/MISO | **GPIO9 MISO** (pin 21) |
| CS/CSB | **GPIO8 CE0** (pin 24) |

### Configuração em Klipper
```ini
[mcu rpi]
serial: /tmp/klipper_host_mcu

[adxl345]
cs_pin: rpi:None        # usa CS automático do spidev0.0
# spi_bus: spidev0.0    # opcional; por defeito usa 0.0

[resonance_tester]
accel_chip: adxl345
probe_points:
   110,110,20           # centro aproximado da Ender‑3
```
Ativa o SPI no Pi e testa com: `ACCELEROMETER_QUERY`.

---

## 3) Safe Z‑Home & Bed Mesh (referência)
```ini
[safe_z_home]
home_xy_position: 110,110
speed: 50
z_hop: 10

[bed_mesh]
speed: 120
mesh_min: 15, 15
mesh_max: 214, 214
probe_count: 5,5
algorithm: bicubic
```

---

## 4) Testes rápidos (úteis)
- **BLTouch:**  
  `BLTOUCH_DEBUG COMMAND=self_test` → `pin_down` → `touch_mode` → `QUERY_PROBE`  
- **Sensor de filamento:**  
  `QUERY_FILAMENT_SENSOR SENSOR=filamento`
- **Ventoinha de peça:**  
  `M106 S255` (liga), `M106 S0` (desliga) — ou `SET_FAN_SPEED FAN=fan SPEED=1.0`
- **ADXL345:**  
  `ACCELEROMETER_QUERY`

---

## 5) Notas e dicas
- A SKR Mini E3 V3 tem **polaritidades** de alguns conectores invertidas face às placas Creality — confirma antes de ligar a cama/hotend.
- Se a ventoinha do hotend e a da caixa estiverem trocadas, troca os conectores **FAN1 ↔ FAN2** ou ajusta os pinos (`PC7`/`PB15`) na config.
- No sensor de filamento, se leres *“RUNOUT”* quando **há** filamento, adiciona o `!` ao `switch_pin` (ex.: `^!PA4`).

---

### Apêndice: excertos prontos a colar no `printer.cfg`
> **Ajusta** os tipos de termístor e o `z_offset` conforme o teu hardware.

```ini
# ==== BLTOUCH ====
[bltouch]
sensor_pin: ^PC14
control_pin: PA1
x_offset: -41
y_offset: 42
# z_offset: (calibrar)

[stepper_z]
endstop_pin: probe:z_virtual_endstop

# ==== FILAMENTO ====
[filament_switch_sensor filamento]
switch_pin: ^PA4
pause_on_runout: True

# ==== FANS ====
[fan]
pin: PC6

[heater_fan hotend_fan]
pin: PC7
heater: extruder
heater_temp: 50.0

[controller_fan caixa]
pin: PB15
idle_timeout: 60

# ==== HOTEND / BED ====
[extruder]
heater_pin: PC8
sensor_pin: PA0
# sensor_type: Generic 3950

[heater_bed]
heater_pin: PC9
sensor_pin: PC4
# sensor_type: Generic 3950

# ==== ADXL / INPUT SHAPER ====
[mcu rpi]
serial: /tmp/klipper_host_mcu

[adxl345]
cs_pin: rpi:None

[resonance_tester]
accel_chip: adxl345
probe_points: 110,110,20
```
