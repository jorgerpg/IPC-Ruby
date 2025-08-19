# SD — Desafio 01 (IPC: Produtor–Consumidor em Ruby)

Sistema produtor–consumidor usando **FIFO (named pipe)** no Linux/WSL, com:

* Produtor interativo (prompt) que envia **mensagens variáveis** em **JSON**.
* Consumidor que **processa** cada mensagem (números e strings), **loga** a saída e responde a **sinais**.
* Tratamento de **erros** e **bloqueios** (pipe cheio, leitor ausente) com escrita **não-bloqueante**.

> Testado em Ubuntu/WSL com Ruby 3.x.

---

## 1) Pré-requisitos

* **WSL/Ubuntu** (no Windows) ou Linux/macOS.
* **Ruby 3.x** + bundler.
* `mkfifo` (já existe por padrão no Linux).

Instale dependências do projeto (na raiz):

```bash
bundle install
```

Crie o FIFO:

```bash
mkdir -p tmp logs
mkfifo tmp/ipc_fifo
```

---

## 2) Como executar

### Passo A — iniciar o consumidor

```bash
./bin/consumer
# (deixa rodando)
```

O consumidor imprime o PID ao iniciar (ou use os comandos da seção “Descobrir PID”).

### Passo B — iniciar o produtor (em outro terminal)

```bash
./bin/producer
```

Digite mensagens e aperte Enter. Exemplos:

```
12 + 5
7*3
10 / 2
9 - 4
sum 1,2,3,4,5
upper Olá mundo
lower ABC
reverse abcdef
count_words foo bar baz
```

Para sair do produtor: `quit`.

Os resultados aparecem **imediatamente** no terminal do consumidor e são gravados em `logs/consumer.log`.

---

## 3) Comandos suportados (consumidor)

* **Aritmética**: `NUM OP NUM` (OP: `+ - * /`)
  Ex.: `12 + 5`, `7*3`, `10 / 2`, `9 - 4`

  > Divisão por zero retorna `NaN`.

* **Agregação**: `sum 1,2,3,4` (separado por vírgula e/ou espaço)

* **Strings**:

  * `upper <texto>` → maiúsculas
  * `lower <texto>` → minúsculas
  * `reverse <texto>` → inverte
  * `count_words <texto>` → conta palavras (separação por espaço)

* **Desconhecido**: qualquer linha fora dos padrões acima é marcada como `kind: "unknown"`.

Cada mensagem enviada pelo produtor contém também um campo aleatório `rnd` para garantir variabilidade.

---

## 4) Testes por requisito (checklist com comandos)

### ✅ Requisito 1 — Dois processos independentes

* **Como testar**:

  1. Rode `./bin/consumer` em um terminal.
  2. Rode `./bin/producer` em outro.
  3. Verifique que ambos têm **PIDs diferentes**:

     ```bash
     pgrep -af bin/consumer
     pgrep -af bin/producer
     ```
* **Esperado**: dois processos distintos em execução.

---

### ✅ Requisito 2 — ≥ 10 envios/recebimentos

* **Opção A (manual)**: digite 10+ linhas no produtor (qualquer combinação dos exemplos).
* **Opção B (rápida, automatizada)**:

  ```bash
  # Gera 20 operações aritméticas aleatórias e alimenta o producer
  ruby -e '20.times{puts "#{rand(1..50)} + #{rand(1..50)}"; sleep 0.05}' | ./bin/producer
  ```
* **Verificação**:

  ```bash
  # contar entradas processadas no log
  grep -c "kind=>" logs/consumer.log
  # ou
  grep -c "\[proc\]" logs/consumer.log
  ```
* **Esperado**: contagem ≥ 10.

---

### ✅ Requisito 3 — Dados variáveis, não repetitivos

* **Como testar**:

  * Envie combinações diferentes (números, strings, sum, etc.) e note que cada mensagem do produtor inclui `rnd` aleatório.
  * Verifique no log:

    ```bash
    tail -n 5 logs/consumer.log
    ```
* **Esperado**: entradas diferentes (tanto pelo conteúdo quanto pelo `rnd`).

---

### ✅ Requisito 4 — Consumidor processa e registra em log/arquivo

* **Como testar**:

  1. Envie:

     ```
     12 + 5
     sum 1,2,3,4,5
     upper Olá mundo
     reverse abc
     count_words foo bar baz
     ```
  2. Veja o terminal do consumidor e o `logs/consumer.log`:

     ```bash
     tail -n 10 logs/consumer.log
     ```
* **Esperado**: cada linha com um hash contendo campos como:

  ```
  {:id=>1, :ts=>"2025-08-19T22:10:00Z", :input=>"12 + 5", :kind=>"arith", :expr=>"12 + 5", :result=>17.0}
  {:id=>2, :ts=>"...", :input=>"sum 1,2,3,4,5", :kind=>"sum", :list=>[1.0,2.0,3.0,4.0,5.0], :result=>15.0}
  {:id=>3, :ts=>"...", :input=>"upper Olá mundo", :kind=>"upper", :from=>"Olá mundo", :result=>"OLÁ MUNDO"}
  ...
  ```

---

### ✅ Requisito 5 — Tratar erros e situações de bloqueio

**5.a) Leitor ausente (consumer não iniciado)**

* **Como testar**:

  1. **Não** rode o consumidor.
  2. Rode **só** o produtor e digite algo (ex.: `1 + 1`).
* **Esperado**: o produtor exibe mensagens como:

  ```
  [produtor] sem leitor ainda... tentativa 1
  ...
  [produtor] não há leitor (consumer); abortando após 10 tentativas.
  ```

  (Número de retentativas/timeout configurados no código.)

**5.b) Pipe cheio (backpressure) — escrita não-bloqueante**

* **Como testar**:

  1. Rode o consumidor.
  2. **Pausar** o consumidor (ver Requisito 6 para sinais):

     ```bash
     kill -USR1 $(pgrep -f bin/consumer)   # pausa
     ```
  3. Em outro terminal, gere muitas mensagens rapidamente:

     ```bash
     yes "sum 1,2,3,4,5,6,7,8,9,10" | head -n 50000 | ./bin/producer
     ```
  4. Observe o produtor:

     * Mensagens como:
       `[produtor] pipe cheio; aguardando disponibilidade...`
       `[produtor] ainda bloqueado (tentativa X/5)`
     * Se o bloqueio persistir por muito tempo, verá:
       `Timeout::Error ... descartando esta linha` (comportamento esperado).
  5. **Retomar** o consumidor:

     ```bash
     kill -USR1 $(pgrep -f bin/consumer)   # retoma
     ```
* **Esperado**:

  * O produtor **não trava**; ele detecta bloqueio e espera com timeout.
  * Ao retomar o consumidor, o fluxo normal se restabelece.

**5.c) Buffer do consumidor (quando pausado)**

* **Como testar**:

  1. Pausar com `USR1`.
  2. Enviar várias mensagens.
  3. **Limpar buffer** com `USR2` (ver Requisito 6).
* **Esperado**: loga `buffered` enquanto pausado e, ao limpar,
  `[signal] USR2: buffer limpo (N mensagens descartadas)`
  (tamanho máximo do buffer configurado em `$max_buffer`).

---

### ✅ Requisito 6 — Sinais (2+ tipos com comportamentos distintos)

**Sinais implementados:**

* `USR1` → **pausar/retomar** o consumidor (mensagens novas vão para o **buffer**).
* `USR2` → **limpar** o buffer (descarta mensagens **em memória**).
* `TERM` e `INT` → **encerrar gracioso** (faz **flush** final se não estiver pausado e imprime contador processado).

**Como testar:**

```bash
PID=$(pgrep -f bin/consumer)

# Pausar
kill -USR1 $PID
# Envie algumas linhas no producer; observe que o consumer registra 'buffered'.

# Retomar
kill -USR1 $PID
# Observe o consumer drenando o buffer ('[proc] ...').

# Limpar buffer (estando pausado)
kill -USR2 $PID
# Mensagem: "[signal] USR2: buffer limpo (N mensagens descartadas)"

# Encerrar gracioso (sem precisar de nova mensagem)
kill -TERM $PID   # ou kill -INT $PID
# Mensagem final: "Consumer finalizado. Processados: X"
```

> Observação: o consumidor usa `IO.select` com timeout curto, então `TERM/INT` encerram **sem** depender de nova mensagem no pipe.

---

## 5) Logs e métricas básicas

* Arquivo de log: `logs/consumer.log` (rotaciona: 10 arquivos de 1 MB).
* Cada processamento incrementa `$processed`; no encerramento é impresso:

  ```
  Consumer finalizado. Processados: <N>
  ```

---

## 6) Dicas, parâmetros e troubleshooting

* **Sem FIFO**: crie com `mkfifo tmp/ipc_fifo`.
* **Permissões**: `chmod +x bin/producer bin/consumer`.
* **Descobrir PID**: `pgrep -af bin/consumer`.
* **Pipe cheio**: é esperado ver logs de backpressure no produtor; ajuste `timeout`/`max_retries` em `write_json_line` se quiser tolerar bloqueios mais longos.
* **Divisão por zero**: retorna `NaN` (teste com `5 / 0`).
* **Windows**: use **WSL** (Ubuntu). No Windows “puro”, named pipes funcionam diferente.

---

## 7) Critérios de aceite (checklist rápido)

* [ ] **(1)** Dois processos independentes (`producer` e `consumer`) rodando em terminais separados.
* [ ] **(2)** ≥ 10 mensagens processadas (verificado no log).
* [ ] **(3)** Mensagens variáveis (conteúdo e `rnd` aleatório).
* [ ] **(4)** Consumidor processa e **loga** resultados (aritmética, `sum`, strings).
* [ ] **(5)** Tratamento de **leitor ausente** e de **pipe cheio** (mensagens de backpressure/timeout no produtor).
* [ ] **(6)** Sinais: `USR1` pausa/retoma (com buffer), `USR2` limpa buffer, `TERM/INT` encerram gracioso.

---