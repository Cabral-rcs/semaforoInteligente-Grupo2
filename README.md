# semaforoInteligente-Grupo2

Este projeto implementa um **Semáforo Inteligente com ESP32**, capaz de operar de forma automática e remota. O sistema controla dois semáforos, detecta objetos próximos com um sensor ultrassônico, ajusta o funcionamento conforme a luminosidade ambiente (modo noturno) e envia dados em tempo real para a plataforma **Ubidots** via MQTT.

### Principais recursos

-  **Modo normal:** alternância automatizada entre verde, amarelo e vermelho.  
-  **Sensor ultrassônico:** ativa alerta ao detectar objetos entre 2 cm e 8 cm.  
-  **Modo noturno automático:** pisca amarelo quando a luminosidade está baixa.  
-  **Controle remoto via Ubidots:** ligar/desligar o sistema e ativar modo noturno.  
-  **Monitoramento:** envio contínuo da luminosidade (LDR) para o Ubidots.  

Um projeto que integra **eletrônica, IoT, automação e comunicação MQTT**.
