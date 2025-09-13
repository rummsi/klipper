# Projeto Ender‑3 + Klipper (Resumo do Projeto)

Este repositório/documento resume **o hardware**, a **topologia** e os **métodos de configuração** que já utilizámos na tua Ender‑3 com Klipper + Fluidd.

## 🧰 Hardware atual
- Base: **Creality Ender‑3** (cama aquecida com **vidro**)
- Mainboard: **BIGTREETECH SKR Mini E3 V3.0** (STM32G0B1, drivers TMC2209 UART)
- Extrusor/Hotend: **Creality Sprite** em **direct‑drive**
- Nivelamento: **BLTouch/3DTouch** (clone) — offsets usados: **X = −41 mm**, **Y = +42 mm**
- Sensor de filamento: **runout** em **E0‑STOP**
- Aceleração/Input Shaper: **ADXL345** ligado ao **Raspberry Pi Zero 2 W** (SPI)
- Host: **Raspberry Pi Zero 2 W** via Wi‑Fi, a correr **Klipper + Moonraker + Fluidd**

> Dica: tens um README específico com o **mapa de ligações** e excertos do `printer.cfg`: `README-hardware-wiring.md`.

---

## 🧩 Estrutura de ficheiros principal
```
/config
  ├─ printer.cfg          # Configuração principal (inclui fluidd.cfg)
  ├─ fluidd.cfg           # Requisitos/atalhos do Fluidd
  ├─ macros.cfg           # (opcional) macros como CANCEL_PRINT/PAUSE/RESUME
  └─ adxl.cfg             # (opcional) configuração/medições do ADXL345
```
> No nosso caso, o `printer.cfg` já inclui `fluidd.cfg` na primeira linha.

---

## 🔧 Métodos de configuração que já usámos

### 1) Flash do Klipper na SKR Mini E3 V3.0
1. Compilar o firmware (`make` no host Klipper) para **STM32G0B1** com **8KiB bootloader** e **USB**.
2. Copiar `out/klipper.bin` para um SD como **`firmware.bin`**.
3. Inserir o SD na SKR e **reiniciar** a placa (auto‑flash).

### 2) Acesso ao Fluidd e `printer.cfg`
- Aceder via browser ao IP do Pi (Fluidd).
- Carregar o `printer.cfg` base e **reiniciar** Klipper.
- Confirmar USB da SKR (ex.: `/dev/serial/by-id/...`).

### 3) BLTouch — offsets e Z‑offset
```ini
[bltouch]
sensor_pin: ^PC14
control_pin: PA1
x_offset: -41
y_offset: 42
# z_offset: (calibrar)
```
Passos:
1. `G28` (home geral) → `PROBE_CALIBRATE`
2. Ajustar a altura “papel” → `ACCEPT` → `SAVE_CONFIG`
3. O `z_offset` fica gravado no final do `printer.cfg`.

> **Nota:** o `^` em `sensor_pin` ativa **pull‑up interno** (importante para estabilizar o sinal da sonda).

### 4) Safe Z‑Home (homing da Z ao centro)
```ini
[safe_z_home]
home_xy_position: 110,110
speed: 50
z_hop: 10
```
Evita tocar fora da área útil e **foge às molas/gar clips** do vidro.

### 5) Bed Mesh (malha da mesa)
```ini
[bed_mesh]
speed: 120
horizontal_move_z: 5
mesh_min: 15,15
mesh_max: 214,214
probe_count: 5,5
algorithm: bicubic
```
**Como definimos os limites (regra prática):**
- A sonda mede em `nozzle_xy + (x_offset, y_offset)`.
- Para a sonda **nunca sair da cama (0..220 mm)**:
  - `nozzle_x_min ≥ -x_offset` → com `x_offset = -41` ⇒ `nozzle_x_min ≥ 41`
  - `nozzle_y_max ≤ 220 - y_offset` → com `y_offset = +42` ⇒ `nozzle_y_max ≤ 178`
- Por isso a malha típica fica **X≈[45..214], Y≈[15..178]** (com margens).

Comandos usados:
- `G28` → `BED_MESH_CALIBRATE` → `SAVE_CONFIG`  
- Para ativar a malha ao iniciar: `BED_MESH_PROFILE LOAD=default` (ou usar `BED_MESH_CALIBRATE` no start do slicer).

### 6) Ajuste dos parafusos da mesa (Screws Tilt Adjust)
```ini
[screws_tilt_adjust]
screw1: 30,30
screw1_name: front_left
screw2: 190,30
screw2_name: front_right
screw3: 30,190
screw3_name: back_left
screw4: 190,190
screw4_name: back_right
speed: 120
horizontal_move_z: 10
```
Comandos: `SCREWS_TILT_CALCULATE` (seguir instruções no ecrã/console).

### 7) Sensor de filamento (runout)
```ini
[filament_switch_sensor filamento]
switch_pin: ^PA4     # E0‑STOP
pause_on_runout: True
```
Teste: `QUERY_FILAMENT_SENSOR SENSOR=filamento`  
Se a lógica estiver invertida, usar `^!PA4` (com `!`).

### 8) Ventoinhas e aquecimentos (mapeamento típico)
```ini
# Ventoinha de peça (FAN0)
[fan]
pin: PC6

# Ventoinha do hotend (FAN1) auto > 50 ºC
[heater_fan hotend_fan]
pin: PC7
heater: extruder
heater_temp: 50.0

# Ventoinha caixa/controlador (FAN2)
[controller_fan caixa]
pin: PB15
idle_timeout: 60

# Hotend e cama (pinos)
[extruder]
heater_pin: PC8
sensor_pin: PA0
# sensor_type: Generic 3950

[heater_bed]
heater_pin: PC9
sensor_pin: PC4
# sensor_type: Generic 3950
```

### 9) ADXL345 e Input Shaper
```ini
[mcu rpi]
serial: /tmp/klipper_host_mcu

[adxl345]
cs_pin: rpi:None

[resonance_tester]
accel_chip: adxl345
probe_points: 110,110,20
```
Passos usados:
1. Ligar ADXL ao **SPI0** do Pi (3V3, GND, SCLK, MOSI, MISO, CE0).
2. `ACCELEROMETER_QUERY` para validar.
3. `SHAPER_CALIBRATE` e depois `SAVE_CONFIG`.

### 10) Macros úteis (compatibilidade com o Fluidd)
```ini
[gcode_macro CANCEL_PRINT]
description: Cancelar impressão (compatível com Fluidd)
gcode:
  TURN_OFF_HEATERS
  M106 S0
  G91
  G1 Z10 F600
  G90
  M84
```
> Podes adicionar `PAUSE`/`RESUME` conforme preferires. O Fluidd costuma alertar se faltar `CANCEL_PRINT`.

### 11) Comandos rápidos do dia‑a‑dia
- **Home:** `G28`
- **Testar BLTouch:** `BLTOUCH_DEBUG COMMAND=self_test`, `pin_down`, `touch_mode`, `QUERY_PROBE`
- **Malha:** `BED_MESH_PROFILE LOAD=default`
- **Sensor filamento:** `QUERY_FILAMENT_SENSOR SENSOR=filamento`
- **Ventoinha peça:** `M106 S255` (liga), `M106 S0` (desliga)
- **Guardar config:** `SAVE_CONFIG`

---

## 🧯 Resolução de problemas (que já vimos)
- **Move out of range** durante a malha → ajustar `mesh_min/mesh_max` de acordo com os **offsets da sonda** e garantir `safe_z_home` ao centro.
- **Falta de `z_offset`** no arranque → executar `PROBE_CALIBRATE` e `SAVE_CONFIG`.
- **BLTouch instável** → garantir `sensor_pin` com `^` (pull‑up) e cablagem correta; verificar `stow_on_each_sample` se necessário.

---

## ✅ Próximos passos recomendados
- `PID_CALIBRATE HEATER=extruder TARGET=...` e `PID_CALIBRATE HEATER=heater_bed TARGET=...`
- Confirmar `rotation_distance` do extrusor Sprite e fluxo (calibração de extrusão).
- Rever `max_accel`, `max_z_velocity` e `input_shaper` (após ADXL).

---

*Qualquer ajuste adicional (ex.: macros personalizadas, part cooling avançado, linear advance via pressure advance, etc.) posso adicionar numa nova revisão.*
