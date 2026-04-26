# Processo Seletivo – Intensivo Maker | IoT
## Etapa Prática – Sistemas Embarcados

👤 Identificação do Candidato

Nome completo: Manuela Menezes Alves
GitHub: manuela-menezes


---

### 1️⃣ Visão Geral da Solução
  O projeto implementa um semáforo veicular com botão de pedestre simulado em MicroPython, executando sobre um ESP32 no ambiente Wokwi.
  O sistema alterna ciclicamente entre os estados VERMELHO -> VERDE -> AMARELO -> VERMELHO, controlando três LEDs físicos com temporização definida por estado. Um botão de pedestre (sensor de entrada) pode ser pressionado a qualquer momento para solicitar travessia; ao detectar o pedido, o sistema antecipa a transição para VERMELHO de forma segura, passando por AMARELO antes.
  Toda a temporização é não-bloqueante, implementada via uasyncio, permitindo que o semáforo e o monitoramento do botão rodem em paralelo como corrotinas independentes.


---

### 2️⃣ Arquitetura do Sistema Embarcado
  O programa é organizado em torno de uma máquina de estados finita (FSM) com duas corrotinas assíncronas rodando em paralelo via asyncio.gather.

  ### Fluxo principal:
  asyncio.run(main())
          ↓
    asyncio.gather(traffic_light(), pedestrian_button())
          │                               │
          │                               └─ polling GPIO 14 a cada 100ms
          │                                  detecta borda de descida (pull-up)
          │                                  seta flag: pedestrian_request = True
          │
          └─ apply_state(current)
            await logger(current, duration)
            verifica pedestrian_request
            → se True e não RED: força YELLOW → RED
            → senão: transita normalmente
            repete indefinidamente


  ### Tabela de transições (FSM):
    Estado              Duração              Próximo
    ---------------------------------------------------------
    RED                 5s                   GREEN
    GREEN               4s                   YELLOW
    YELLOW              2s                   RED

### 3️⃣ Componentes Utilizados na Simulação

  Componente              ID                 Pino         Função
  ------------------------------------------------------------------------------------------------
  ESP32 DevKit C v4       esp                —            Microcontrolador principal (MicroPython)
  LED vermelho            led_red            GPIO 25      Sinaliza estado VERMELHO
  LED amarelo             led_yellow         GPIO 26      Sinaliza estado AMARELO
  LED verde               led_green          GPIO 27      Sinaliza estado VERDE
  Resistor 220 Ω (×3)     r1, r2, r3         —            Limita corrente dos LEDs (~6 mA cada)
  Botão de pedestre       btn_pedestrian     GPIO 14      Sensor de entrada — solicita travessia
  Monitor Serial          —                  TX/RX        Log de progressão dos estados

  ### Cálculo do resistor:
    R = (Vcc − Vf) / If = (3,3 V − 2,0 V) / 0,006 A ≈ 217 Ω → 220 Ω (valor comercial padrão).
    O botão usa o pull-up interno do ESP32 (Pin.PULL_UP): repouso em HIGH, ativo em LOW - sem resistor externo necessário.

### 4️⃣ Decisões Técnicas
- Temporização não-bloqueante (uasyncio):
    Com await asyncio.sleep(), o event loop permanece ativo entre os ticks, permitindo que pedestrian_button() monitore o GPIO em paralelo sem interferir no ciclo do semáforo.
- Dicionário LEDS indexado por nome de estado:
    Os pinos são armazenados num dicionário cujas chaves coincidem com os nomes dos estados da FSM. Isso elimina estruturas if/elif em apply_state() - a função acende o LED correto com uma única linha, independentemente do número de estados.
- Flag compartilhada pedestrian_request:
    A comunicação entre as duas corrotinas é feita via variável global booleana. É a abordagem idiomática para uasyncio single-threaded: sem necessidade de locks ou filas, pois o event loop é cooperativo e não há condição de corrida real.
- Debounce por software (200 ms):
    Após detectar a borda de descida do botão, a corrotina aguarda 200 ms antes de retomar o polling, evitando leituras múltiplas de um único pressionamento.
- all_off() antes de cada transição:
    Garante invariante de segurança: nunca dois LEDs acesos simultaneamente, independentemente da ordem de execução das operações nos pinos.

### 5️⃣ Resultados Obtidos
  O sistema opera conforme esperado:

  - Os três LEDs alternam na sequência VERMELHO → VERDE → AMARELO → VERMELHO.
  - Apenas um LED permanece aceso por vez em cada estado.
  - As durações respeitam os tempos configurados (5 s / 4 s / 2 s).
  - O botão de pedestre, quando pressionado durante o estado VERDE ou AMARELO, antecipa a transição para VERMELHO de forma segura.
  - O event loop permanece responsivo durante toda a execução.
  - A simulação executa continuamente sem erros, satisfazendo o critério de validação automática via GitHub Actions.

### 6️⃣ Comentários Adicionais
  - No ambiente de CI (GitHub Actions), o botão nunca é pressionado; a simulação roda headless, sem interação. O código de leitura do sensor está implementado e funcional, mas o evento não ocorre automaticamente. Para validação completa do comportamento do pedestre, é necessário testar manualmente no Wokwi.

