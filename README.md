# Módulo IoT para Sistemas Embarcados Baseados em Arduino 

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](https://opensource.org/licenses/MIT)
[![Language: C++](https://img.shields.io/badge/Language-C++-blue.svg?style=flat-square)](https://isocpp.org/)
[![Platform: Arduino & ESP](https://img.shields.io/badge/Platform-Arduino%20%7C%20ESP8266%20%7C%20ESP32-lightgrey.svg?style=flat-square)](#)

Módulo desenvolvido para integrar com mais facilidade (e qualidade) projetos com o ecossistema Arduino ao universo da Internet das Coisas (IoT). 

> Projeto desenvolvido na Universidade do Estado do Amazonas (EST/UEA), com apoio do Programa de Apoio à Iniciação Científica 2025-2026 (PAIC/FAPEAM).

---

## 1. Contextualização

Para sair do projeto básico em Arduino e integrar ele à internet, é necessário tomar cuidados que muitas vezes não se dá a devida atenção. Em grande parte da comunidade MAKER e até mesmo em Trabalhos de Conclusão de Curso (TCCs), é muito comum a adoção de uma prática conhecida como _hardcoding_. Essa prática consiste em colocar as credenciais da rede WiFi (nome e senha) diretamente no código, que, embora possa funcionar em ambientes de prototipagem, apresenta falahas críticas em cenários de aplicação real. Isso porque ela dificulta imensamente a escalabilidade do projeto, compromete a segurança das credenciais e dificulta a manutenção desses sistemas, visto que após qualquer alteração na rede, seria necessário reprogramar o microcontrolador para que ele se conecte novamente à rede.

## 2. Objetivos

Dessa forma, o objetivo principal é desenvolver um módulo, de _software_ e _hardware_, para integração dos projetos que utilizam microcontroladores da família Arduino com a IoT.

O módulo deve:

- Ser genérico e abranger situações em que o sistema embarcado ainda não possui chip de WiFi, sendo necessário um chip externo
- Permitir a configuração dinâmica e segura das credênciais de rede, elimando o _hardcoding_
- Permitir que seja possível localizar o dispositivo na rede através do seu _hostname_

## 3. Metodologia

O sistema é dividido em duas camadas que operam de forma independente:

### Software
A camada de conexão com a rede WiFi é embarcada no chip WiFi (como o ESP-01 ou ESP32) e opera em dois estados:

1) O dispositivo não está conectado no WiFi. Para isso ele precisa ser configurado, iniciando no modo de Access Point, ele cria um Captive Portal onde é inserido as credênciais de rede via uma interface web fácil e intuitiva. Essas credenciais ficam salvas de forma segura na memória do dispositivo.

2) O dispositivo já está conectado ao WiFi. Com isso, ele inicia o protocolo Multicast DNS (mDNS), possibilitando que seja localizado por um _hostname_ na rede.

--

### Hardware
A comunicação entre o microcontrolador mestre (responsável pela leitura de sensores e controle de atuadores) e o chip WiFi é realizada fisicamente via barramento UART padrão. Para garantir a integridade dos dados trafegados e evitar o esgotamento do buffer serial, a arquitetura exige o empacotamento das mensagens no formato JSON, utilizando a biblioteca ArduinoJson.

A comunicação assíncrona é gerenciada pela classe dedicada de comunicação serial desenvolvida neste framework. A sincronização entre as placas ocorre por meio de uma rotina não-bloqueante de handshake, que assegura que o mestre e o escravo iniciem a troca de cargas úteis apenas quando ambas as extremidades atestarem disponibilidade, mitigando a perda de pacotes.

## 4. Integração Física e Restrições Técnicas

Para a correta implementação da topologia Mestre-Escravo (ex: Arduino Nano + ESP-01), as conexões físicas requerem atenção aos níveis lógicos das portas UART:
* O pino TX do mestre deve ser conectado ao pino RX do transceptor.
* O pino RX do mestre deve ser conectado ao pino TX do transceptor.
* É mandatório o uso de um divisor de tensão (ou conversor de nível lógico) no barramento receptor do transceptor, caso este opere em 3.3V (como o ESP-01) e o mestre opere em 5V (como o Arduino Uno/Nano).

Devido ao uso da UART de hardware nativa para comunicação entre as placas, a porta de comunicação USB compartilhada não deve ser utilizada simultaneamente para rotinas de debug via monitor serial.

## 5. Guia Rápido de Uso

Abaixo é apresentado o modelo de inicialização e sincronismo utilizando a classe de comunicação serial.

```cpp
#include <Arduino.h>
#include "serial_comm.h"

Serial_comm espComm;

void setup() {
  Serial.begin(9600);
  
  // Define o intervalo não-bloqueante de tentativas
  espComm.setHandshakeInterval(1000); 
  
  // Inicia o processo de sincronismo com o módulo transceptor
  espComm.doHandshake("master", "syn", "esp_module");
}

void loop() {
  espComm.getJson();
  
  if (espComm.jsonUpdateCheck()) {
    // Processamento da carga útil recebida
    if (espComm.state == "LIGAR") {
      // Lógica de atuação
    }
  }
}
