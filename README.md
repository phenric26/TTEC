# 📡 Telemetria via Heltec LoRa V3

Este projeto implementa um sistema de telemetria utilizando comunicação LoRa (915 MHz). Ele é composto por um nó transmissor (Tx) responsável por fazer a aquisição de dados de geolocalização de alta precisão, orientação espacial (9 eixos) e condições climáticas, e um nó receptor (Rx) que desserializa os dados para processamento no lado do servidor/estação base.

## 🚀 Arquitetura do Sistema e Design

O sistema foi desenhado com foco em alta disponibilidade, resiliência contra falhas de I/O e otimização de banda de rede. A arquitetura é dividida em três camadas principais: Hardware (Sensoriamento), Firmware (Processamento Lógico) e Telemetria (Camada Física e Enlace).

### 1. Arquitetura de Hardware e Topologia (Sensoriamento)
* **Unidade de Processamento Central:** O *core* do nó transmissor é o **ESP32-S3** (Heltec WiFi LoRa 32 V3), operando como orquestrador central de eventos assíncronos.
* **Topologia Dual-Bus I2C (Desacoplamento de Hardware):** Para mitigar gargalos de I/O e colisões de clock, a arquitetura emprega a segregação física dos barramentos I2C:
  * **Barramento 0 (`Wire`):** Dedicado a sensores de tráfego determinístico e menor frequência de amostragem (u-blox GNSS a 1Hz e BME280).
  * **Barramento 1 (`Wire1`):** Dedicado exclusivamente ao IMU BNO086. Devido à altíssima taxa de interrupções e geração de dados do coprocessador interno do BNO, isolá-lo em um canal rodando a 100kHz previne *buffer overflows* e garante a integridade do barramento primário.

### 2. Arquitetura de Firmware e Resiliência (Processamento)
* **Design Orientado a Eventos (Event-Driven):** O laço principal (`loop`) não utiliza *delays* bloqueantes. A extração de dados do IMU utiliza uma técnica de esvaziamento contínuo de buffer (`while loop` no registro SHTP) para garantir que pacotes antigos não saturem a memória do conversor.
* **Mecanismos de Self-Healing (Tolerância a Falhas):** O firmware incorpora uma rotina de *watchdog* de software monitorando a flag `wasReset()` do IMU. Em caso de surtos elétricos, oscilações no barramento ou travamentos do coprocessador, o ESP32 reconfigura os registradores do sensor *on-the-fly*, sem necessidade de um *hard reset* no sistema inteiro.
* **Isolamento de Domínio (Prevenção de Name Collision):** A arquitetura de software garante o encapsulamento estrito das bibliotecas. O conflito de macros (`MODE_SLEEP`) entre a camada PHY do rádio SX1262 e os registradores do sensor BME280 é resolvido na etapa de pré-compilação através do ordenamento hierárquico de inclusão dos *headers*, preservando o escopo de ambas as bibliotecas.

### 3. Arquitetura de Comunicação e Telemetria (LoRa)
* **Serialização de Payload:** Como a telemetria agrupa múltiplos tipos de dados (Geometria 3D, Clima e Coordenadas Globais), adotou-se um modelo de empacotamento baseado em *String CSV Nomeada*. Isso permite flexibilidade no receptor para realizar o *parsing* (via `sscanf`) sem a necessidade de dicionários complexos ou desserializadores pesados como Protobuf.
* **Otimização de Frame LoRa:** O buffer dinâmico foi projetado para o limite de ~250 bytes de *payload* (MTU prático para LoRa com Fator de Espalhamento / SF 7). O controle de estado `lora_idle` atua como um semáforo de rede, garantindo que o rádio só despache um novo frame após o término do processamento de interrupção (IRQ) do pacote anterior.

---

## 🛠️ Requisitos de Hardware

1. 2x Placas **Heltec WiFi LoRa 32 (V3)** (Nó Tx e Nó Rx)
2. Módulo GNSS **u-blox SAM-M10Q** (ou família F9/M10 com suporte I2C)
3. Sensor IMU 9-DoF **BNO086** (ou BNO085/BNO080)
4. Sensor Ambiental **BME280** (Temperatura, Pressão e Umidade)
5. Cabos com conectores Qwiic / STEMMA QT ou Jumpers padrão.

---

## 🔌 Diagrama de Conexões (Pinagem do Nó Tx)

Para garantir estabilidade, evite os pinos 39 e 40 do ESP32-S3 devido ao uso nativo pelo barramento JTAG. A pinagem adotada divide a carga elétrica e de dados.

| Sensor / Módulo | Barramento | Heltec V3 (Pino) | Módulo (Pino) | Observação |
| :--- | :--- | :--- | :--- | :--- |
| **Todos** | Alimentação | `3V3` | `3V3` | Atenção à corrente de pico do GPS. |
| **Todos** | Terra | `GND` | `GND` | Terra comum obrigatório. |
| **u-blox GNSS** | I2C 0 (`Wire`) | `41` | `SDA` | Amostragem de ~1Hz (PVT) |
| **u-blox GNSS** | I2C 0 (`Wire`) | `42` | `SCL` | |
| **BME280** | I2C 0 (`Wire`) | `41` | `SDA` | Cascata com o GNSS (Endereço: 0x77) |
| **BME280** | I2C 0 (`Wire`) | `42` | `SCL` | |
| **BNO086** | I2C 1 (`Wire1`) | `48` | `SDA` | Isolado. Clock reduzido para 100kHz. |
| **BNO086** | I2C 1 (`Wire1`) | `47` | `SCL` | (Endereço: 0x4B) |

---

## 📦 Dependências de Software

Certifique-se de instalar as seguintes bibliotecas através do *Library Manager* da Arduino IDE:

* `SparkFun u-blox GNSS v3` (Para o SAM-M10Q)
* `SparkFun BME280` (Para temperatura/pressão/umidade)
* `SparkFun BNO08x` (Motor SHTP para IMU)
* `Heltec ESP32 Dev-Boards` (Pacote de placas que inclui a biblioteca `LoRaWan_APP.h`)

---

## 📡 Estrutura do Payload (Protocolo LoRa)

Os dados são transmitidos em uma string concatenada em ASCII. O tamanho médio do pacote é de ~150 bytes, bem abaixo do limite de 250 bytes estabelecido no `BUFFER_SIZE`.

**Formato do Pacote:**
`Lat:%ld,Lon:%ld,Alt:%ld,Sat:%d,T:%.1f,P:%.1f,U:%.1f,AX:%.2f,AY:%.2f,AZ:%.2f,GX:%.2f,GY:%.2f,GZ:%.2f`

**Exemplo Prático (Tx):**
```text
Lat:-157640903,Lon:-478690584,Alt:1032856,Sat:5,T:24.7,P:905.0,U:51.1,AX:0.00,AY:0.00,AZ:9.81,GX:0.00,GY:0.00,GZ:0.00
