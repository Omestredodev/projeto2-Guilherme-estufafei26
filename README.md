# Estufa inteligente na plataforma FIWARE — Grupo `estufafei26`

**Disciplina:** CCM520 — Internet das Coisas (Projeto 2)  
**Integrantes:** `[Nome completo — RA]`, `[Nome completo — RA]`  
**Simulação Wokwi:** https://wokwi.com/projects/480422279547961345  
**Vídeo de demonstração (opcional):** `[inserir link, se houver]`

## 1. Descrição

A Estufa Inteligente monitora temperatura, umidade relativa do ar, umidade do solo e luminosidade por meio de uma ESP32 simulada no Wokwi. O sistema também controla um relé (irrigação simulada) e um servomotor (abertura da janela), recebe comandos remotos para alterar os atuadores, o intervalo de amostragem e a publicação de medidas, e disponibiliza as informações na plataforma FIWARE. O IoT Agent JSON integra o MQTT ao Orion Context Broker; o Cygnus persiste o histórico no MySQL; um consumidor recebe alertas de solo seco; e o Grafana apresenta um painel de monitoramento.

## 2. Arquitetura

```text
ESP32 simulada no Wokwi
  │ MQTT TCP/1883 — medidas, comandos e respostas
  ▼
broker.hivemq.com:1883 (broker público)
  │ Ponte MQTT bidirecional, conforme os tópicos configurados
  ▼
Mosquitto (Docker, porta 1883)
  │ MQTT
  ▼
IoT Agent JSON (Docker, HTTP/4041; MQTT via broker local)
  │ NGSIv2 / HTTP
  ▼
Orion Context Broker (Docker, HTTP/1026)
  ├── subscription de histórico → Cygnus (HTTP/5050) → MySQL (TCP/3306)
  │                                                    └── Grafana (HTTP/3000)
  └── subscription de solo seco → consumidor (HTTP interno/3000,
                                    publicado no host em localhost:3001)
```

As portas acima indicam os serviços utilizados; entre contêineres devem ser usados os nomes de serviços da rede Docker. O endereço do consumidor dentro da rede é `http://consumidor:3000/alerta-estufa`, e o destino do Cygnus é `http://cygnus:5050/notify`.

A ponte local é necessária também porque o IoT Agent JSON utiliza assinaturas MQTT compartilhadas, que poderiam repartir mensagens entre agentes de outros usuários em um broker público.

## 3. Hardware

| Componente | Pino da ESP32 | Observação |
| --- | --- | --- |
| DHT22 | GPIO 4 | Temperatura e umidade relativa do ar |
| Sensor de umidade do solo (analógico) | GPIO 34 | Leitura ADC convertida em porcentagem |
| LDR (analógico) | GPIO 35 | Leitura ADC convertida em porcentagem de luminosidade |
| Relé | GPIO 26 | Estado binário: desligado (0) / ligado (1) |
| Servomotor | GPIO 27 | Abertura simulada de 0° a 180° |

**Calibração no simulador:** as leituras ADC são convertidas para uma escala de 0 a 100% usando os extremos definidos no firmware (`ADC_SECO_ESCURO = 4095` e `ADC_MOLHADO_CLARO = 0`), com tratamento de inversão conforme o sensor. Esses são valores de referência da simulação, não medições de calibração em hardware físico. O circuito completo pode ser consultado no link do Wokwi; recomenda-se manter também uma cópia local de `firmware/diagram.json`.

## 4. Identificadores

| Item | Valor |
| --- | --- |
| API key | `estufafei26` |
| Client ID da ponte | `ponte_estufafei26` (`remote_clientid` no Mosquitto) |
| `fiware-service` / `fiware-servicepath` | `fei` / `/estufa` |
| `device_id` | `estufa001` |
| Entidade NGSIv2 | `urn:ngsi-ld:Greenhouse:001` |
| Tipo da entidade | `Greenhouse` |
| Client ID MQTT da ESP32 | `esp32_estufafei26_estufa001` |

## 5. Modelo de dados

### 5.1 Medidas

| `object_id` no payload | Atributo NGSIv2 | Tipo | Unidade/faixa |
| --- | --- | --- | --- |
| `t` | `temperature` | Number | °C |
| `h` | `relativeHumidity` | Number | % |
| `s` | `soilMoisture` | Number | 0–100% |
| `l` | `illuminance` | Number | 0–100% |
| `r` | `relayState` | Number | 0/1 |
| `a` | `windowAngle` | Number | 0–180° |
| `i` | `samplingInterval` | Number | segundos |

**Exemplo de publicação MQTT:**

```json
{"t":24.0,"h":40.0,"s":100,"l":75.6,"r":1,"a":90,"i":5}
```

A entidade também inclui o atributo temporal `TimeInstant` e atributos estáticos como `name` e `location`.

### 5.2 Comandos

| Comando | Valores aceitos | Sucesso | Exemplo de erro |
| --- | --- | --- | --- |
| `setRelay` | `0` ou `1` | `{"setRelay":"OK"}` | `ERRO: ...` para valor inválido |
| `setServo` | Inteiro entre `0` e `180` | `{"setServo":"OK"}` | `ERRO: setServo aceita 0 a 180` |
| `setInterval` | Inteiro entre `2` e `10` segundos | `{"setInterval":"OK"}` | `ERRO: setInterval aceita 2 a 10 segundos` |
| `setSending` | `0` ou `1` | `{"setSending":"OK"}` | `ERRO: ...` para valor inválido |

O intervalo padrão é de **3 segundos**. `setSending = 0` interrompe a publicação periódica de medidas sem impedir o processamento de comandos; `setSending = 1` retoma o envio. O firmware devolve resultados no tópico `cmdexe`, permitindo ao IoT Agent atualizar `<comando>_status` e `<comando>_info` no Orion.

### 5.3 Tópicos MQTT

| Sentido | Tópico |
| --- | --- |
| ESP32 → IoT Agent: medidas | `/json/estufafei26/estufa001/attrs` |
| IoT Agent → ESP32: comandos | `/estufafei26/estufa001/cmd` |
| ESP32 → IoT Agent: resultado | `/json/estufafei26/estufa001/cmdexe` |

### 5.4 Ponte com o broker público

| Padrão | Direção na ponte Mosquitto | Motivo |
| --- | --- | --- |
| `/json/estufafei26/estufa001/attrs` | `in` | Encaminhar medições do broker público ao IoT Agent local |
| `/json/estufafei26/estufa001/cmdexe` | `in` | Encaminhar resultados dos comandos ao IoT Agent local |
| `/estufafei26/estufa001/cmd` | `out` | Encaminhar comandos do IoT Agent à ESP32 no broker público |

A conexão da ponte usa `broker.hivemq.com:1883`, com `remote_clientid ponte_estufafei26`. Essa configuração está em `plataforma/mosquitto/mosquitto.conf`.

## 6. Como reproduzir

**Pré-requisitos:** Docker Desktop com Docker Compose, Postman Desktop, navegador e acesso à simulação Wokwi. É necessário acesso à Internet para o broker MQTT público. Não publique senhas pessoais ou arquivos de ambiente contendo segredos.

1. **Obter o projeto.** Clone o repositório de entrega (`git clone [URL_DO_REPOSITORIO]`) ou extraia os arquivos enviados pelo grupo. Abra a pasta raiz `estufa-inteligente`.
2. **Conferir a ponte MQTT.** No arquivo `plataforma/mosquitto/mosquitto.conf`, confirme broker `broker.hivemq.com:1883`, client ID e os três tópicos da seção 5.4.
3. **Subir os serviços.** No terminal:

   ```powershell
   cd plataforma
   docker compose up -d
   docker compose ps
   ```

   Espere os serviços essenciais ficarem saudáveis. A pilha contém Orion, IoT Agent, Mosquitto, MongoDB, MySQL, Cygnus, Grafana, consumidor e simulador de lâmpada. O MongoDB da estufa é publicado no host na porta **27018** neste projeto, evitando conflito com outra instância local.
4. **Abrir o Postman.** Importe `postman/Projeto 2.postman_collection.json` e configure as variáveis usadas pela coleção, incluindo `orion=localhost:1026`, `iotagent=localhost:4041`, `servico=fei`, `caminho=/estufa`, `apikey=estufafei26`, `device_id=estufa001` e `entidade=urn:ngsi-ld:Greenhouse:001`.
5. **Provisionar o dispositivo.** Execute, na ordem, as requisições relevantes das pastas **Etapa 1 – Ambiente** e **Etapa 2 – Provisionamento**, criando o service group e a estufa apenas se ainda não existirem. Evite repetir requisições `POST` que criam registros, para não duplicar entidades ou subscriptions.
6. **Iniciar a ESP32.** Abra https://wokwi.com/projects/480422279547961345, confira o firmware de `firmware/main.py` (cópia do código do Wokwi) e inicie a simulação. O Serial Monitor deve indicar conexão Wi-Fi/MQTT e publicação de medidas. A ESP32 usa `Wokwi-GUEST`.
7. **Validar as medidas e comandos.** Execute a pasta **Etapa 3 – Firmware e comandos**, respeitando a ordem e um intervalo suficiente entre requisições de comando e verificações. A entidade deve apresentar os sete atributos numéricos e o `TimeInstant`; os status dos comandos devem passar de `PENDING` para `OK` quando as respostas chegarem.
8. **Configurar histórico e alertas.** Execute as requisições correspondentes da **Etapa 4**: subscription para Cygnus (`http://cygnus:5050/notify`, formato `normalized`), subscription de solo seco (`http://consumidor:3000/alerta-estufa`, formato `keyValues`, `throttling=5`) e verificações. Não recrie subscriptions existentes.
9. **Configurar Grafana.** Abra http://localhost:3000, configure uma senha própria de administrador e crie a fonte de dados **MySQL** com host interno `mysql-db:3306`, database `fei` e usuário `grafana`. Obtenha a senha da configuração local do projeto; não a inclua em documentação pública. O usuário possui privilégio `SELECT` em `fei`.
10. **Importar ou reconstruir o dashboard.** Utilize `plataforma/painel-estufa.json` (cópia do JSON exportado) ou `grafana/estufa-inteligente-dashboard.json`. Ao importar, associe a fonte de dados MySQL correta. O dashboard **Estufa inteligente** contém as visualizações especificadas na seção 8.
11. **Validar resultados.** Execute as verificações da Etapa 4 no Postman. O histórico MySQL pode ser conferido em `fei.estufa`; a tabela apresenta colunas como `recvTimeTs`, `recvTime`, `entityId`, `attrName` e `attrValue`.

**Observação sobre credenciais:** o login do Grafana, a senha do MySQL e eventuais variáveis de ambiente devem ser configurados pelo responsável pela execução, sem compartilhá-los no repositório. O usuário `grafana` é somente de leitura.

## 7. Subscrições

| Subscrição | Condição | Destino | Atributos e formato | Justificativa |
| --- | --- | --- | --- | --- |
| Histórico | Alterações nos sete atributos de medidas | `http://cygnus:5050/notify` | `temperature`, `relativeHumidity`, `soilMoisture`, `illuminance`, `relayState`, `windowAngle`, `samplingInterval`; `normalized` | Armazenar séries históricas no MySQL via Cygnus |
| Solo seco | Alterações em `soilMoisture`, com expressão `soilMoisture<30` | `http://consumidor:3000/alerta-estufa` | `soilMoisture`, `relayState`, `TimeInstant`; `keyValues` | Avisar o consumidor quando a condição de solo seco for atendida |

A subscription do alerta usa `throttling=5` segundos para limitar a frequência de notificações. Esse mecanismo **não garante**, sozinho, um único aviso por episódio de seca; ele apenas limita a taxa de envio. A regra de transição pode ser aprimorada em uma implementação futura.

## 8. Painel Grafana

**Nome:** `Estufa inteligente`  
**Fonte de dados:** MySQL (`mysql-db:3306`, banco `fei`, usuário de leitura `grafana`)  
**JSON exportado:** `plataforma/painel-estufa.json` e `grafana/estufa-inteligente-dashboard.json`.

| Visualização | Tipo | O que apresenta |
| --- | --- | --- |
| Temperatura e umidade do ar | Time series | Séries independentes de `temperature` e `relativeHumidity` |
| Umidade do solo | Time series | Evolução de `soilMoisture`, incluindo os testes de solo seco |
| Estado do relé | State timeline | Alternância entre `Desligado` (0) e `Ligado` (1) |
| Luminosidade atual | Stat | Último valor de `illuminance` em porcentagem |

As consultas de histórico filtram por `entityId` e usam os campos `recvTimeTs`/`recvTime` do Cygnus. As consultas temporais usam `$__timeFilter` para respeitar o intervalo selecionado no Grafana; a conexão MySQL foi configurada com sessão UTC (`+00:00`).

**Captura do painel:** adicionar em `evidencias/grafana/dashboard-completo.png` antes da entrega.

## 9. Testes e evidências

A validação foi realizada usando a coleção Postman do professor e o Serial Monitor do Wokwi.

| Conjunto | Resultado observado |
| --- | --- |
| Etapa 1 — Ambiente | 10/10 testes aprovados |
| Etapa 2 — Provisionamento | Grupo e dispositivo cadastrados; medições e registros de comandos verificados |
| Etapa 3 — Firmware e comandos | 39/39 testes aprovados |
| Etapa 4 — Histórico, alertas e Grafana | Requisições 4.1–4.13 validadas; verificação final 4.13: 8/8 |

Foram observados: publicação de medidas, execução e confirmação de `setRelay`, `setServo`, `setInterval` e `setSending`; rejeição de valores fora da faixa; histórico MySQL; alerta de solo seco; e os quatro painéis do dashboard.

**Evidências:** organize capturas em `evidencias/etapa1`, `etapa2`, `etapa3`, `etapa4`, `grafana` e `wokwi`. Se o Postman permitir exportar os resultados do Runner, coloque o arquivo exportado em `postman/`. A coleção utilizada **não contém pasta `99`**, apesar da referência a ela no modelo original do README; utilizar as verificações efetivamente presentes na coleção.

## 10. Decisões de projeto e limitações

- **Ponte MQTT local:** evita conectar o IoT Agent JSON diretamente ao broker público, reduzindo o conflito potencial entre suas assinaturas compartilhadas e as de outros agentes no broker.
- **Identificadores específicos:** API key, entidade, device ID e tópicos configurados de forma consistente entre Wokwi, ponte e provisionamento.
- **Validação de comandos:** verificação de tipos e faixas; comandos inválidos retornam `ERRO: ...` sem aplicar valores fora dos limites.
- **Laço não bloqueante:** durante a operação, o firmware continua verificando comandos enquanto publica medições periodicamente; `setSending` desativa apenas o envio de medidas.
- **Tratamento de falhas:** reconexão automática do MQTT e nova assinatura do tópico de comandos, observadas durante os testes.
- **Calibração de sensores:** escalas ADC lineares adequadas à simulação; exigiriam calibração experimental em equipamento físico.
- **Broker MQTT público sem autenticação/TLS:** adequado ao exercício, mas inadequado para dispositivos reais com dados e controle sensíveis; seria necessário broker privado/gerenciado, autenticação, TLS e controles de acesso.
- **Notificações:** `throttling` limita frequência, mas não implementa lógica de borda de evento nem substitui uma automação de irrigação.
- **Persistência:** dados históricos são armazenados pelo Cygnus no MySQL; o dispositivo simulado não é um sistema físico de irrigação.

## 11. Desafios realizados

| Desafio opcional | Situação | Evidência / localização |
| --- | --- | --- |
| D.1 — Automação em malha fechada | Não realizado | Não há serviço automatizando `setRelay` pelo Orion e segunda subscription de desligamento |
| D.2 — Reconexão | **Realizado** | `firmware/main.py`; Serial Monitor mostrou `Conexao MQTT perdida: -1`, reconexão ao broker e nova assinatura dos comandos |
| D.3 — TLS | Não realizado | Comunicação com o broker público usa TCP/1883, sem TLS |
| D.4 — Segunda estufa | Não realizado | Há apenas `estufa001` |

## 12. Uso de ferramentas de IA

Foram utilizadas ferramentas de IA como apoio ao desenvolvimento e à documentação do projeto, incluindo explicações de conceitos FIWARE/MQTT, auxílio na depuração de configurações e comandos, revisão de consultas SQL do Grafana, estruturação de testes e redação/revisão deste README. As configurações e os resultados técnicos foram verificados pela equipe nas ferramentas utilizadas (Docker, Postman, Wokwi, Orion, MySQL e Grafana). **Os integrantes devem revisar esta declaração e ajustá-la à utilização real e às regras de integridade acadêmica da disciplina.**

## 13. Referências

- Material de aula e roteiro do **Projeto 2 — CCM520, Internet das Coisas**, fornecidos pelo professor.
- Documentação oficial FIWARE: https://fiware-orion.readthedocs.io/ e https://fiware-iotagent-json.readthedocs.io/
- Documentação Grafana — MySQL: https://grafana.com/docs/grafana/latest/datasources/mysql/
- Documentação Eclipse Mosquitto: https://mosquitto.org/man/mosquitto-conf-5.html
- Documentação Wokwi: https://docs.wokwi.com/
- Broker MQTT público HiveMQ: https://www.hivemq.com/mqtt/public-mqtt-broker/

---

### Conferência antes da entrega

- [ ] Substituir nomes/RA dos integrantes e URL do repositório.
- [ ] Conferir que `firmware/main.py` contém o firmware final testado e salvar `firmware/diagram.json` se disponível.
- [ ] Conferir que o JSON exportado está nos caminhos indicados e importa corretamente.
- [ ] Adicionar capturas e exportações de resultados existentes às pastas `evidencias/` e `postman/`.
- [ ] Remover quaisquer senhas, tokens ou dados pessoais dos arquivos que serão compartilhados.
- [ ] Conferir variáveis de ambiente e arquivos de inicialização necessários a uma reprodução em outro computador.
