| Supported Targets | ESP32-P4 | ESP32-S2 | ESP32-S3 |
| ----------------- | -------- | -------- | -------- |

---

Esse código C é utilizado para lidar com dispositivos USB HID (Human Interface Devices) como teclados e mouses em um ambiente ESP32 usando FreeRTOS. Vamos dividi-lo em partes e explicá-lo com analogias ao que você já sabe de Python.

### 1. **Includes e Definições**
```c
#include <stdio.h>
#include <stdbool.h>
#include <string.h>
#include <unistd.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "freertos/queue.h"
#include "esp_err.h"
#include "esp_log.h"
#include "usb/usb_host.h"
#include "driver/gpio.h"
```
Esses cabeçalhos são como as `import` em Python.
Eles trazem bibliotecas essenciais para manipular tarefas (`tasks`),
grupos de eventos, manipulação de portas GPIO,
etc. , além de lidar com dispositivos USB.

### 2. **Definição de Constantes**
```c
#define APP_QUIT_PIN GPIO_NUM_0
```
Aqui define-se uma constante,
algo semelhante a criar uma variável em Python que não muda,
usada para o botão de "sair" do programa,
ligado a um pino específico.

### 3. **Estruturas e Tipos de Dados**
```c
typedef enum {
    APP_EVENT = 0,
    APP_EVENT_HID_HOST
} app_event_group_t;
```
Essa `enum` é como um "menu" de opções.
Em Python,
seria um `enum.Enum`,
onde `APP_EVENT` representa eventos gerais,
e `APP_EVENT_HID_HOST` representa eventos relacionados ao host HID (
como conexão ou desconexão de um dispositivo)
.

### 4. **Mapeamento de Teclas**
```c
const uint8_t keycode2ascii [57][2] = {
    {0, 0}, // Nenhuma tecla pressionada
    {'a', 'A'}, // 'a' minúscula e 'A' maiúscula
    ...
};
```
Essa tabela é usada para converter códigos de teclas em caracteres.
Algo semelhante a criar um dicionário em Python
que mapeia códigos de teclas para os caracteres que eles representam.

### 5. **Callback de Eventos de Teclado**
A função `key_event_callback` lida com eventos de teclado, como pressionar ou soltar uma tecla.
```c
static void key_event_callback(key_event_t *key_event)
{
    unsigned char key_char;
    hid_print_new_device_report_header(HID_PROTOCOL_KEYBOARD);
    
    if (KEY_STATE_PRESSED == key_event->state) {
        if (hid_keyboard_get_char(key_event->modifier, key_event->key_code, &key_char)) {
            hid_keyboard_print_char(key_char);
        }
    }
}
```
Essa função é similar a criar uma função Python que é
chamada sempre que uma tecla é pressionada.
Ela faz a conversão do código da tecla (`key_code`)
para um caractere (`key_char`) e o imprime.

### 6. **Relatórios de Teclado e Mouse**
Essas funções lidam com a entrada de dispositivos USB,
processando os relatórios de entrada de teclado e mouse.

```c
static void hid_host_keyboard_report_callback(const uint8_t *const data, const int length)
```
Essa função processa os dados recebidos do teclado USB
e chama a função de callback para lidar
com as teclas pressionadas ou liberadas.
Algo equivalente em Python seria receber
dados de um dispositivo via USB e processá-los em tempo real.

```c
static void hid_host_mouse_report_callback(const uint8_t *const data, const int length)
```
De maneira similar,
essa função lida com os dados do mouse,
ajustando as coordenadas X e Y
com base nos movimentos e imprimindo essas posições.

### 7. **Funcionamento Geral**
O programa funciona em loops infinitos (típico em sistemas embarcados).
Ele espera eventos de teclado e mouse (relacionados a dispositivos USB HID)
e processa esses eventos chamando as funções apropriadas de callback.

Aqui está o fluxo básico:
- **Inicialização:** O código configura o sistema para lidar com dispositivos HID.

- **Entrada de dados:** Quando o sistema detecta um evento de teclado ou mouse,
    ele processa o relatório de entrada (dados recebidos).

- **Callback:** Funções de callback são chamadas para tratar cada evento
    (por exemplo, o `key_event_callback` para teclas pressionadas).

- **Exibição:** Os resultados são exibidos no console ou processados conforme a necessidade.

No geral, o código usa muitas abstrações típicas de C para gerenciar hardware (como USB e GPIO), enquanto em Python, essas interações seriam muito mais simplificadas.

---

# USB HID Class example
This example implements a basic USB Host HID Class Driver, and demonstrates how to use the driver to communicate with USB HID devices (such as Keyboard and Mouse or both) on the ESP32-S2/S3. Currently, the example only supports the HID boot protocol which should be present on most USB Mouse and Keyboards. The example will continuously scan for the connection of any HID Mouse or Keyboard, and attempt to fetch HID reports from those devices once connected. To quit the example (and shut down the HID driver), users can GPIO0 to low (i.e., pressing the "Boot" button on most ESP dev kits).


### Hardware Required
* Development board with USB capable ESP SoC (ESP32-S2/ESP32-S3)
* A USB cable for Power supply and programming
* USB OTG Cable

### Common Pin Assignments

If your board doesn't have a USB A connector connected to the dedicated GPIOs, 
you may have to DIY a cable and connect **D+** and **D-** to the pins listed below.

```
ESP BOARD    USB CONNECTOR (type A)
                   --
                  | || VCC
[GPIO19]  ------> | || D-
[GPIO20]  ------> | || D+
                  | || GND
                   --
```

### Build and Flash

Build the project and flash it to the board, then run monitor tool to view serial output:

```
idf.py -p PORT flash monitor
```

The example serial output will be the following:

```
I (198) example: HID HOST example
I (598) example: Interface number 0, protocol Mouse
I (598) example: Interface number 1, protocol Keyboard

Mouse
X: 000883       Y: 000058       |o| |
Keyboard
qwertyuiop[]\asdfghjkl;'zxcvbnm,./
Mouse
X: 000883       Y: 000058       | |o|
```

Where every keyboard key printed as char symbol if it is possible and a Hex value for any other key. 

#### Keyboard input data
Keyboard input data starts with the word "Keyboard" and every pressed key is printed to the serial debug.
Left or right Shift modifier is also supported. 

```
Keyboard
Hello, ESP32 USB HID Keyboard is here!
```

#### Mouse input data 
Mouse input data starts with the word "Mouse" and has the following structure. 
```
Mouse
X: -00343   Y: 000183   | |o|
     |            |      | |
     |            |      | +- Right mouse button pressed status ("o" - pressed, " " - not pressed)
     |            |      +--- Left mouse button pressed status ("o" - pressed, " " - not pressed)
     |            +---------- Y relative coordinate of the cursor 
     +----------------------- X relative coordinate of the cursor 
```
