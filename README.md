# SPRINT_DSA

# Totem de Recarga Inteligente — SPRINT DSA

> Simulação completa de um totem de carregamento de veículos elétricos com protocolos industriais reais (OCPP e Modbus), controle dinâmico de potência, múltiplas sessões simultâneas e exportação de relatórios.

---

## Desenvolvido por

**Equipe Lumos — FIAP · Análise e Desenvolvimento de Sistemas · 2026**

| Nome | RM |
|------|----|
| Pietro Lorande da Silva | 569125 |
| Gustavo Bonamico Piccoli | 569984 |
| Maria Eduarda Medeiros Lemos | 574094 |
| Ana Beatriz Berbel Marini | 574176 | 
| Julian Nayde Moncoski | 572603 | 
| Marcelo Francisco Josafá Ribeiro Martins | 573905 |

> *"迈向更绿色的未来 — Rumo a um Futuro mais Verde"*
---

## Visão Geral

Este sistema simula um **totem de carregamento inteligente para veículos elétricos**, integrando dois protocolos industriais amplamente usados no setor de EVs — **OCPP** e **Modbus RTU** — com um gerenciador de energia chamado **Lynx Power Manager**.

O sistema é capaz de:

- Registrar múltiplas sessões de recarga e persisti-las em JSON
- Comunicar-se com um servidor central via frames OCPP (simulado)
- Controlar o hardware do carregador via registros Modbus (simulado)
- Aplicar limites dinâmicos de potência conforme a bandeira tarifária e a carga simultânea
- Integrar-se a APIs externas para pagamento, sincronização cloud e consulta da rede ANEEL
- Exportar relatórios individuais e consolidados em arquivo `.txt`

---

## Protocolos Simulados

### OCPP — Open Charge Point Protocol

**O que é:**
O OCPP (Open Charge Point Protocol) é o padrão de comunicação entre um **Charge Point** (o carregador físico) e um **Central System** (servidor de gerenciamento da rede de recarga). É o "idioma" que permite que totens de diferentes fabricantes conversem com qualquer backend de operação.

**Como funciona na prática:**
A comunicação se dá via troca de mensagens JSON sobre WebSocket. Cada mensagem pode ser de três tipos:

| Tipo | Código | Significado |
|------|--------|-------------|
| `CALL` | `2` | O carregador envia uma requisição ao servidor |
| `CALLRESULT` | `3` | O servidor responde confirmando a requisição |
| `CALLERROR` | `4` | O servidor responde com um erro |

**Ações OCPP implementadas neste sistema:**

| Ação | Momento de uso |
|------|----------------|
| `BootNotification` | Ao iniciar o sistema — o totem se apresenta ao servidor e aguarda autorização |
| `StatusNotification` | Notifica mudanças de estado do conector (ex: `Charging`, `Available`) |
| `StartTransaction` | Registra o início de uma sessão de recarga no servidor central |
| `MeterValues` | Envia leituras de energia e potência a cada 10 minutos durante a recarga |
| `StopTransaction` | Finaliza a transação e informa o medidor final ao servidor |

**Exemplo de frame OCPP gerado pelo sistema:**
```
[OCPP][14:32:01] SEND  >> [2, "A3F9C1B2", "StartTransaction", {"connectorId": 1, "idTag": "123456", "meterStart": 0, "timestamp": "2026-06-12T14:32:01"}]
[OCPP][14:32:01] RECV  << [3, "A3F9C1B2", {"transactionId": 57382, "idTagInfo": {"status": "Accepted"}}]
```

**Por que é importante:**
O OCPP garante que cada sessão seja auditável — início, leituras intermediárias de medidor e encerramento ficam registrados no servidor central, viabilizando faturamento, rastreabilidade e conformidade regulatória.

---

### Modbus RTU

**O que é:**
Modbus é um protocolo serial de comunicação industrial criado em 1979 e ainda amplamente usado em sistemas de automação. Na versão **RTU** (Remote Terminal Unit), os dados são transmitidos em formato binário compacto via RS-485, tornando-o ideal para ambientes industriais ruidosos.

Enquanto o OCPP cuida da comunicação com o servidor remoto, o **Modbus cuida da comunicação com o hardware local** — o controlador de carga, sensores e medidores de energia instalados fisicamente no totem.

**Como funciona:**
Cada mensagem Modbus contém: o ID do dispositivo alvo, o código de função (FC), o endereço do registro e o valor ou quantidade a ler/escrever.

**Códigos de função implementados:**

| Código | Nome | Uso no sistema |
|--------|------|----------------|
| `0x03` | Read Holding Registers | Lê configurações e status do controlador |
| `0x04` | Read Input Registers | Lê leituras do medidor (tensão, corrente, potência) |
| `0x06` | Write Single Register | Configura limite de potência ou habilita/desabilita carga |
| `0x10` | Write Multiple Registers | Atualiza múltiplos parâmetros em uma única operação |

**Registros usados:**

| Endereço | Função |
|----------|--------|
| `0x0100` | Limite máximo de potência (× 10, em décimos de kW) |
| `0x0101` | Habilitar / desabilitar carga (`1` = on, `0` = off) |
| `0x0110` | Limite secundário de potência (write multiple) |
| `0x0200` | Registro de status geral do controlador |
| `0x0300` | Tensão (V) do medidor |
| `0x0301` | Corrente (A) do medidor |
| `0x0302` | Potência ativa (kW) do medidor |

**Exemplo de tráfego Modbus gerado:**
```
[MODBUS][14:32:05] TX >> ID=0x01 FC=0x06(Write Single Register) REG=0x0100 VAL=220
[MODBUS][14:32:05] TX >> ID=0x01 FC=0x06(Write Single Register) REG=0x0101 VAL=1
[MODBUS][14:32:05] TX >> ID=0x01 FC=0x03(Read Holding Registers) REG=0x0200 QTY=1
[MODBUS][14:32:05] RX << ID=0x01 DATA=[0x4f2a] (20266)
```

---

## Arquitetura do Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                    TOTEM LUMOS-T1                           │
│                                                             │
│  ┌──────────────┐    OCPP/WebSocket    ┌──────────────────┐ │
│  │  Charge Point│◄────────────────────►│ Central System   │ │
│  │  (este app)  │                      │ (GoodWe Cloud)   │ │
│  └──────┬───────┘                      └──────────────────┘ │
│         │                                                   │
│         │ Modbus RTU (RS-485)                               │
│         ▼                                                   │
│  ┌──────────────────────────────────┐                       │
│  │  Controlador de Carga            │                       │
│  │  - Limite de potência (0x0100)   │                       │
│  │  - Enable/Disable (0x0101)       │                       │
│  │  - Medidor V/I/kW (0x03xx)       │                       │
│  └──────────────────────────────────┘                       │
│                                                             │
│  ┌──────────────────────────────────┐                       │
│  │  Lynx Power Manager              │                       │
│  │  - Bateria local                 │                       │
│  │  - Fonte solar / rede de apoio   │                       │
│  │  - Limite por bandeira tarifária │                       │
│  └──────────────────────────────────┘                       │
│                                                             │
│  ┌──────────────────────────────────┐                       │
│  │  Banco de Sessões (JSON local)   │                       │
│  │  sessoes_totem.json              │                       │
│  └──────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Sessões de Recarga

### Registro de Sessões Múltiplas

O sistema persiste todas as sessões em um arquivo **`sessoes_totem.json`** no diretório de execução. Isso garante que o histórico sobreviva entre reinicializações do programa.

**Como funciona:**

1. Ao iniciar, `carregar_sessoes()` lê o arquivo JSON e retorna a lista de sessões anteriores (ou uma lista vazia se o arquivo não existir).
2. Ao fim de cada sessão bem-sucedida, a sessão é adicionada à lista em memória e `salvar_sessoes()` reescreve o arquivo completo.
3. Os IDs de sessão são gerados sequencialmente com a data embutida:
   ```
   SESS-0001-20260612
   SESS-0002-20260612
   ...
   ```

**Função de geração de ID:**
```python
def gerar_id_sessao(sessoes):
    return f"SESS-{len(sessoes) + 1:04d}-{datetime.now().strftime('%Y%m%d')}"
```

Isso garante que, mesmo após reiniciar o sistema com o arquivo existente, os IDs continuem incrementais e com data correta.

---

### Controle de Potência por Sessões Simultâneas

O sistema implementa o **Lynx Power Manager** — um gerenciador que decide a potência disponível para cada sessão com base em três fatores:

**1. Bandeira tarifária:**

| Bandeira | Potência máxima |
|----------|----------------|
| Verde | 22,0 kW |
| Amarela | 15,0 kW |
| Vermelha | 7,4 kW |

**2. Estado da bateria Lynx (armazenamento local):**

| Bateria | Comportamento |
|---------|--------------|
| ≥ 70% | Fonte: Solar + Lynx — tarifa reduzida (R$ 0,85/kWh) |
| 40–69% | Fonte: Bateria parcial — tarifa intermediária (R$ 1,20/kWh) |
| 20–39% | Potência limitada a 7,4 kW |
| < 20% | Modo carga lenta — potência limitada a 3,7 kW |

**3. Número de sessões simultâneas ativas:**

Quando há mais sessões ativas do que o limite padrão (`MAX_SESSOES_SIMULTANEAS = 2`), o sistema aplica um fator de redução progressivo de 10% por sessão excedente, com piso de 50%:

```python
fator = max(1 - (0.10 * excedente), 0.5)
limite *= fator
```

| Sessões ativas | Fator aplicado |
|---------------|---------------|
| 1–2 | 100% (sem redução) |
| 3 | 90% |
| 4 | 80% |
| 7 ou mais | 50% (piso) |

O resultado final sempre é limitado ao máximo físico do totem: **22,0 kW**.

---

## Exportação de Relatórios

O sistema oferece três formas de relatório:

### Relatório individual (na tela)
Exibido automaticamente ao fim de cada sessão. Contém todos os dados daquela recarga específica.

### Relatório consolidado (na tela)
Acessado pelo menu (opção 2). Agrega estatísticas de todas as sessões:
- Total de sessões, energia acumulada e receita total
- Duração total e ticket médio
- Marca de veículo mais recorrente
- Distribuição de sessões por bandeira tarifária
- Últimas 5 sessões
- Alerta de sessões pendentes de sincronização cloud

### Exportação para `.txt` (arquivo)
Acessada pelo menu (opção 4). Gera um arquivo com timestamp no nome (ex: `relatorio_20260612_143201.txt`) contendo **duas seções**:

**Seção 1 — Detalhamento por Sessão:**
Cada sessão recebe um bloco completo com todos os seus campos (tempo, veículo, energia, cobrança, integração cloud).

**Seção 2 — Consolidado Geral:**
O mesmo resumo exibido na tela, mas escrito no arquivo para consulta futura.

**Função de exportação:**
```python
def exportar_relatorio_txt(sessoes):
    nome_arquivo = f"relatorio_{datetime.now().strftime('%Y%m%d_%H%M%S')}.txt"
    # monta todas as linhas e salva com encoding UTF-8
    with open(nome_arquivo, "w", encoding="utf-8") as f:
        f.write("\n".join(linhas))
```

O arquivo é salvo no diretório de execução do script. O caminho absoluto é exibido no terminal após a geração.

---

## Fluxo Completo de uma Sessão

```
1. Usuário informa nome e token (6 dígitos)
        │
2. BootNotification OCPP → servidor confirma o totem
        │
3. Detecção simulada do veículo (marca, % bateria)
        │
4. Cálculo de potência disponível (Lynx Power Manager)
   └── Modbus: configura controlador de carga
        │
5. Consulta ANEEL para estabilidade da rede
        │
6. Usuário escolhe forma de pagamento
   └── Validação via API externa (PIX / Boleto / TAG / Cartão)
        │
7. OCPP StartTransaction → servidor retorna transactionId
   └── StatusNotification: "Charging"
        │
8. Loop de recarga minuto a minuto
   └── A cada 10 min: MeterValues OCPP + leitura Modbus
        │
9. Fim do tempo → OCPP StopTransaction
   └── Modbus: desabilita carga
   └── StatusNotification: "Available"
        │
10. Sincronização com GoodWe Cloud
        │
11. Exibição do relatório da sessão
        │
12. Sessão salva em sessoes_totem.json
```

---

## Menu Principal

```
  1 - Iniciar nova sessão de recarga
  2 - Ver relatório consolidado
  3 - Ver detalhes de uma sessão específica
  4 - Exportar relatório para .txt
  5 - Sair
```

---

## Integrações Externas Simuladas

Todas as integrações são simuladas com `simular_requisicao_api()`, que introduz um delay realista e tem 10% de chance de simular timeout. Em caso de falha no pagamento, o sistema tenta novamente uma vez antes de cancelar a sessão.

| Sistema | Propósito |
|---------|-----------|
| GoodWe Cloud | Sincronização das sessões para o servidor remoto |
| Banco Central (PIX) | Validação de pagamento via PIX |
| Bradesco Boletos | Validação de boleto bancário |
| Sem Parar TAG | Validação de tag veicular |
| Mastercard/Visa | Validação de pagamento por aproximação |
| ANEEL Grid | Confirmação da bandeira tarifária e estabilidade da rede |

---

## Instalação e Execução

**Requisitos:**
- Python 3.8 ou superior
- Nenhuma dependência externa — apenas bibliotecas padrão (`time`, `random`, `json`, `os`, `uuid`, `datetime`)

**Executar:**
```bash
python SPRINT_DSA_Completa.py
```

O arquivo `sessoes_totem.json` será criado automaticamente na primeira sessão e reutilizado nas execuções seguintes.

---

## Estrutura de Dados de uma Sessão

Cada sessão é armazenada como um objeto JSON com os seguintes campos:

```json
{
  "id": "SESS-0001-20260612",
  "ativa": false,
  "usuario": "Pietro",
  "token": "123456",
  "inicio": "14:32:01",
  "fim": "15:02:01",
  "duracao_min": 30,
  "marca_carro": "BYD",
  "bateria_inicial": 45,
  "bateria_final": 75.0,
  "bateria_lynx": 82,
  "fonte": "Solar + Bateria Lynx",
  "bandeira": "Verde",
  "potencia_kw": 22.0,
  "energia_kwh": 11.0,
  "estabilidade_rede": "Estavel",
  "tarifa": 0.85,
  "taxa_fixa": 15,
  "valor_energia": 9.35,
  "valor_total": 24.35,
  "pagamento": "1",
  "pagamento_nome": "PIX (Governo)",
  "pagamento_ok": true,
  "ocpp_transaction_id": 57382,
  "cloud_sync": true,
  "cloud_timestamp": "2026-06-12T14:32:05.123456"
}
```
