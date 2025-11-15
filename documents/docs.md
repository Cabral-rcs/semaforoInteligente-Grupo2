# Semáforo Inteligente com ESP32 - GRUPO 2

## Objetivo

Desenvolver um sistema de semáforo inteligente que:

- Funcione em modo normal com alternância entre verde, amarelo e vermelho.
- Ative alerta visual quando um objeto é detectado próximo (sensor ultrassônico).
- Entre em modo noturno com pisca amarelo quando a luminosidade ambiente estiver baixa.
- Permita controle remoto via Ubidots (ligar/desligar sistema).
- Envie dados de luminosidade (LDR) para monitoramento remoto.

## Fotos da construção do protótipo

## link do vídeo de funcionamento 
https://youtube.com/shorts/KOQPLC6UCcs?si=yp198-c1N_eYS5GT

## Link da interface
https://inteli-ubidots.iot-application.com/app/dashboards/public/dashboard/ZEpf9fhguvvVloX77q80IcCwXU5eN4fkf9ZE2BhYvgs?navbar=true&contextbar=false&layersBar=false

## Materiais Utilizados

| Componente                       | Quantidade | Função                                      |
|----------------------------------|------------|---------------------------------------------|
| ESP32                     | 1          | Microcontrolador principal                  |
| Sensor LDR                       | 1          | Detecta luminosidade ambiente               |
