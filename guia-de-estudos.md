# Guia de estudos: Estruturas concorrentes e lock-free

Programa de Mestrado em Ciência da Computação · Universidade Federal do Ceará (UFC)

Este guia acompanha os slides da aula, um a um, na mesma numeração. Para cada slide há uma explicação do conteúdo, os termos que você deve dominar e, quando existir, um exercício no simulador interativo deste repositório (`simulador/simulador-lock-lockfree.html`, abra no navegador).

**Livro-base:** HERLIHY, M.; SHAVIT, N.; LUCHANGCO, V.; SPEAR, M. *The Art of Multiprocessor Programming*. 2. ed. Cambridge: Morgan Kaufmann, 2020.

## Sumário

- Parte 0 · Motivação (slides 1 a 7)
- Parte 1 · Locks e correção (slides 8 a 14)
- Parte 2 · CAS (slides 15 a 18)
- Parte 3 · Estruturas lock-free (slides 19 a 25)
- Parte 4 · Prática e encerramento (slides 26 a 32)
- Questões para revisão
- Glossário

## Como usar o simulador

| Slide | Modo ou cenário | O que observar |
| --- | --- | --- |
| 3 e 4 | Sem sincronização, 2 threads, Passo a passo | Cada passo é uma linha da tabela, até o depósito perdido |
| 8 | Com lock, 3 threads | A fila de espera e o saldo sempre correto |
| 11 | Cenário Deadlock; depois Com lock + Suspender no pior momento | Passos 1 a 5 formam o ciclo; 6 a 12 aplicam a ordem global; depois o convoying |
| 14 | Lock x lock-free + Suspender no pior momento | O lock trava todas as threads; com CAS as outras seguem |
| 18 | Lock-free (CAS), 3 threads | O CAS falha em vermelho e a thread tenta de novo |
| 21 | Cenário ABA: Sem versão, depois Com versão | Passo 6: o CAS aceita o endereço reciclado; com versão, falha |
| 23 | Cenário Fila Michael-Scott | Passo 4: T1 suspensa; passo 6: ajuda mútua; passo 9: CAS falha sem prejuízo |
| 25 | Lock-free (CAS), Contínuo, 8 threads | As falhas de CAS crescem com a disputa |

Prefira o modo **Passo a passo**: cada clique em "Próximo passo" executa uma única ação e mostra o evento em "Último evento".

## Parte 0 · Motivação (slides 1 a 7)

### Slide 1 · Estruturas concorrentes e lock-free

**Tema da aula:** como construir estruturas de dados (filas, pilhas, contadores, mapas) que várias threads podem usar ao mesmo tempo sem corromper os dados e, em alguns casos, sem usar locks.

**Por que o assunto importa:** um bug que só aparece às vezes e some quando se coloca um `print` ou se usa o depurador costuma ser de concorrência. O `print` muda o tempo de execução e, com isso, a ordem em que as threads se intercalam. Esses bugs são chamados de *heisenbugs*.

**A imagem da capa:** uma lista encadeada (círculos são nós, linhas são ponteiros). O nó laranja é o ponto de disputa, o lugar da memória onde várias threads querem mexer ao mesmo tempo.

### Slide 2 · Roteiro da aula

A aula segue uma escada:

1. O problema: a condição de corrida.
2. O que é uma estrutura de dados concorrente e qual é o critério de correção.
3. A solução clássica (locks) e seus custos.
4. A alternativa (lock-free), que depende de uma instrução de hardware chamada CAS.
5. Uma pilha e uma fila lock-free reais, as armadilhas e quando vale a pena usá-las.

**Fio condutor:** o mesmo contador de saldo aparece três vezes. No começo ele quebra (bug). Na parte de locks, é consertado com `synchronized`. Na parte de CAS, é consertado sem lock. Comparar as soluções sobre o mesmo exemplo é a forma mais rápida de entender a diferença entre elas.

### Slide 3 · Condição de corrida: depósitos perdidos

**O experimento:** uma conta com saldo zero, duas threads, cada uma deposita 1 real um milhão de vezes (`depositar1M` é um laço que chama `conta.depositar()` um milhão de vezes). A intuição diz que o saldo final será 2.000.000.

**O resultado:** o valor impresso é menor que 2.000.000 e muda a cada execução. O valor exato depende da máquina.

**O que está acontecendo:** cada thread executa `saldo++` milhões de vezes, em núcleos diferentes, ao mesmo tempo. Às vezes uma sobrescreve o trabalho da outra. Isso é uma **condição de corrida** (*race condition*): o resultado depende da ordem em que as threads se intercalam, e essa ordem não é controlada pelo programador.

**Pontos importantes:**

- O compilador não reclama e não há exceção. O programa só dá a resposta errada, e por isso passa em testes simples.
- Com uma thread só, funciona sempre. A concorrência quebra código que estava correto.
- Em Java, além da atomicidade, há o problema de **visibilidade**: sem sincronização, uma thread nem tem garantia de ver as escritas da outra (tema do slide 24).

**Por que um banco:** perder depósitos é obviamente grave. Em sistemas reais, o mesmo bug aparece como saldo divergente, ingresso vendido duas vezes ou estoque negativo (slide 5).

**No simulador:** modo Sem sincronização, 2 threads, Passo a passo. Avance até aparecer em vermelho o depósito perdido e compare "Depósitos feitos" com "Saldo na memória".

### Slide 4 · Condição de corrida passo a passo

`saldo++` é uma linha em Java, mas o processador executa três passos:

1. **Ler:** copiar o valor de `saldo` da memória para um registrador da CPU.
2. **Calcular:** somar 1 no registrador.
3. **Gravar:** copiar o registrador de volta para a memória.

O sistema operacional (ou o próprio hardware, com threads em núcleos diferentes) pode intercalar esses passos de qualquer jeito. A tabela do slide mostra uma intercalação ruim:

- **t1 e t2:** A lê 41; B também lê 41, porque A ainda não gravou.
- **t3 e t4:** cada uma calcula 42 no seu próprio registrador.
- **t5 e t6:** A grava 42; B grava 42 por cima.

**Resultado:** dois depósitos foram feitos, mas o saldo subiu só 1. Esse fenômeno é a **atualização perdida** (*lost update*).

**Vocabulário essencial:**

- **Seção crítica:** trecho de código que acessa dado compartilhado e não pode ser executado por duas threads ao mesmo tempo sem cuidado.
- **Operação atômica:** operação indivisível. Nenhuma outra thread consegue ver ou interferir no meio dela.
- **Read-modify-write:** o padrão ler, modificar, gravar. É o padrão que causa o problema.

**As duas famílias de solução estudadas na aula:**

- **Exclusão mútua (locks):** impedir que outra thread entre no meio. Trava, faz os três passos, destrava.
- **Sincronização otimista:** deixar todas tentarem, mas detectar que alguém entrou no meio e tentar de novo. Faz o trabalho e só confirma se ninguém mexeu.

**Com um único núcleo o bug acontece?** Sim. O sistema operacional pode trocar de thread entre o passo 1 e o passo 3 (preempção). Com vários núcleos, só fica mais frequente.

### Slide 5 · Condições de corrida em sistemas reais

A condição de corrida não é um detalhe de programador: é um problema de qualquer recurso compartilhado. Cada caso do slide reaparece mais adiante:

1. **Último ingresso:** dois sistemas leem "1 disponível" (ler), decidem vender (calcular) e gravam "0 disponível, vendido para mim" (gravar). É exatamente o `saldo++`. Solução comum: reserva atômica do assento ("marque o assento, desde que ainda esteja livre"), a ideia do CAS (slide 16).
2. **Corrida de aplicativo:** um pedido de corrida só pode ter um motorista. Dois aceites chegam quase juntos e o sistema precisa garantir que exatamente um vence. É um problema de consenso (slide 17).
3. **Curtidas:** um contador extremamente disputado vira gargalo. É o caso de uso do `LongAdder`, que espalha a soma em várias células (slide 25).
4. **Black Friday:** a corrida do saldo aplicada ao estoque. Mostra que o problema também existe entre servidores diferentes acessando o mesmo banco de dados.
5. **Wiki e git:** aqui o sistema não perde a atualização. A Wikipedia avisa que a página mudou; o git recusa o push se o remoto avançou. É controle otimista: ler, trabalhar localmente e só confirmar se nada mudou. É o padrão do CAS.
6. **Fila de pedidos:** produtores e consumidores. Vários atendentes inserem pedidos (enqueue) e várias cozinhas retiram (dequeue) ao mesmo tempo. É a fila de Michael-Scott (slide 23).

**Regra geral:** sempre que mais de um agente lê, decide e escreve sobre o mesmo recurso, há risco de corrida.

### Slide 6 · Lei de Amdahl: o limite do paralelismo

Todo programa tem uma parte que pode ser dividida entre núcleos e uma parte que precisa rodar em série. A parte serial limita o ganho:

S(n) = 1 / ((1 − p) + p / n)

em que p é a fração paralelizável e n o número de núcleos. Com 95% de código paralelo, 64 núcleos rendem cerca de 15 vezes; com 5% de código serial, nem infinitos núcleos passam de 20 vezes. Com 50%, o ganho nunca passa de 2 vezes.

**Ligação com a aula:** um lock disputado faz as threads passarem uma de cada vez, ou seja, cria trecho serial. Quanto mais threads disputam o mesmo lock, maior o trecho serial e menor o ganho com mais núcleos. Estruturas concorrentes melhores (locks finos ou lock-free) existem para diminuir esse trecho.

As curvas do gráfico saem diretamente da fórmula; não são medições. A Lei de Gustafson trata de problemas que crescem com o número de núcleos.

**Leitura:** AMDAHL (1967); livro-base, capítulo 1.

### Slide 7 · O que é uma estrutura de dados concorrente

**Definição:** estrutura de dados concorrente (no livro-base, *concurrent object*) é uma estrutura compartilhada que várias threads podem acessar simultaneamente, com operações que continuam corretas qualquer que seja a forma como as threads se intercalam.

A estrutura sequencial (`ArrayList`, `HashMap`, `LinkedList` do Java) só é correta se uma thread por vez a usar. A versão concorrente (`ConcurrentHashMap`, `ConcurrentLinkedQueue`, `AtomicLong`) foi projetada para o acesso simultâneo. Uma estrutura sequencial envolvida por um único lock (`Collections.synchronizedList`, por exemplo) já é concorrente e correta, mas é a solução mais simples possível e fica lenta sob disputa.

**Toda estrutura concorrente é avaliada por três perguntas:**

1. **Segurança (*safety*):** ela está correta? "Nada de ruim acontece." O critério formal é a linearizabilidade (slide 12).
2. **Progresso (*liveness*):** quem consegue terminar suas operações? "Algo de bom acaba acontecendo." Bloqueante, lock-free ou wait-free (slide 13).
3. **Desempenho:** fica mais rápida com mais threads ou vira um gargalo serial?

**Exemplo real:** o mapa de assentos de um voo, alterado por milhares de clientes ao mesmo tempo. Segurança: nenhum assento vendido duas vezes. Progresso: se um servidor trava, as outras vendas continuam? Desempenho: o sistema aguenta a abertura das vendas?

**Leitura:** livro-base, capítulo 3 (objetos concorrentes, linearizabilidade e progresso) e capítulos 9 a 11 (listas, filas e pilhas concorrentes).

## Parte 1 · Locks e correção (slides 8 a 14)

### Slide 8 · Locks e correção

A solução óbvia para o contador é um lock: `synchronized` em Java, `std::mutex` em C++, `Mutex` em Rust ou Go. Um **lock** (ou **mutex**, de *mutual exclusion*) é um objeto que só uma thread pode segurar por vez: quem tenta adquiri-lo enquanto outra o segura fica esperando.

```java
synchronized void depositar() { saldo++; }
```

Com o lock, os três passos (ler, calcular e gravar) formam uma seção crítica que só uma thread executa por vez, e o programa imprime sempre 2.000.000.

**O que vem nesta parte:** como usar locks em estruturas de dados (granularidade), os problemas que locks trazem, os dois critérios de uma boa estrutura concorrente (linearizabilidade e garantias de progresso) e a definição precisa de lock-free.

**No simulador:** modo Com lock, 3 threads, Passo a passo. O saldo sempre bate com os depósitos feitos; quem não tem o lock entra na fila (linha tracejada até o cadeado).

### Slide 9 · Granularidade: coarse-grained x fine-grained

Granularidade é a primeira decisão de projeto de uma estrutura baseada em locks: quanto da estrutura cada lock protege.

**Coarse-grained locking (lock grosso):** um único lock protege a estrutura inteira.

- Vantagem: é trivial de implementar e de provar correto, porque dentro do lock o código é sequencial.
- Desvantagem: sob disputa, todas as threads fazem fila no mesmo lock e a estrutura se comporta como se tivesse uma thread só.
- Exemplos: `synchronized` em todos os métodos, `Collections.synchronizedMap`, `Hashtable`.
- Exemplo real: um site de reservas que trava o voo inteiro enquanto um cliente escolhe o assento. Funciona, mas na abertura de uma promoção todos os clientes do voo fazem fila.

**Fine-grained locking (lock fino):** a estrutura é dividida em partes, cada uma com seu lock (um por grupo de buckets num mapa, técnica chamada *lock striping*, ou um por nó numa lista).

- Vantagem: threads em partes diferentes trabalham em paralelo.
- Desvantagens: o código fica bem mais complexo; operações que envolvem várias partes adquirem vários locks e, se duas threads os adquirirem em ordens diferentes, ocorre deadlock; adquirir e liberar muitos locks tem custo.
- Exemplo real: travar só o assento escolhido. Clientes que escolhem assentos diferentes reservam em paralelo, mas trocar de lugar mexe em dois assentos, precisa de dois locks e passa a correr risco de deadlock.

**Evolução real:** o `ConcurrentHashMap` do Java 5 a 7 dividia a tabela em 16 segmentos, cada um com seu lock. No Java 8 foi reescrito com CAS para inserir em bucket vazio e `synchronized` apenas no primeiro nó do bucket, uma granularidade ainda mais fina que mistura locks e lock-free.

**Pergunta para pensar:** num mapa com lock por segmento, qual operação fica difícil? Resposta: redimensionar a tabela e calcular `size()`, porque precisam olhar todos os segmentos, ou seja, adquirir todos os locks.

### Slide 10 · Cinco técnicas: do coarse-grained ao lock-free

Este slide resume o capítulo 9 do livro-base: lock-free não é um salto mágico, é o último degrau de uma escada de técnicas, cada uma trocando simplicidade por paralelismo. O exemplo do livro é uma **lista encadeada ordenada que implementa um conjunto** (`add`, `remove`, `contains`).

1. **Coarse-grained:** um lock para a lista inteira. Correto e simples; nenhum paralelismo.
2. **Fine-grained (hand-over-hand ou lock coupling):** cada nó tem seu lock. Para percorrer a lista, a thread segura o lock do nó atual e do próximo e avança "de mão em mão", sempre segurando pelo menos um. Várias threads cabem na lista, mas andam em fila indiana, e travar cada nó é caro.
3. **Optimistic (otimista):** percorre a lista sem locks, encontra o lugar, trava só os dois nós envolvidos e então **valida**: percorre de novo para confirmar que os nós ainda estão na lista e ainda são vizinhos. A validação é necessária porque, durante a busca sem lock, outra thread pode ter removido um desses nós ou inserido outro entre eles. Se a validação falhar, recomeça. Ganha quando conflitos são raros.
4. **Lazy (preguiçosa):** a remoção tem duas etapas: primeiro o nó é marcado como removido (remoção lógica), depois é desligado da lista (remoção física). Com isso, `contains()` nem precisa de lock: basta olhar a marca.
5. **Lock-free:** nenhum lock. Todas as mudanças de ponteiro são feitas com CAS. A lista lock-free de Harris e Michael marca antes de desligar, como a lazy, mas guarda a marca no próprio ponteiro, para que marca e ponteiro mudem juntos num único CAS (em Java, `AtomicMarkableReference`).

**Mensagem:** da técnica 1 para a 5, aumenta o paralelismo e aumenta muito a dificuldade de acertar. Na prática, estruturas de produção misturam técnicas, como o `ConcurrentHashMap` do Java 8.

### Slide 11 · Os quatro problemas dos locks

Locks garantem correção, mas criam problemas que motivam a busca por estruturas lock-free.

1. **Deadlock (impasse):** a thread A segura o lock X e espera o lock Y; a thread B segura Y e espera X. Ninguém avança, para sempre. Ocorre quando há mais de um lock adquirido em ordens diferentes, o que é comum em fine-grained locking.
   - Condições de Coffman (todas precisam valer para haver deadlock): exclusão mútua, segurar e esperar, não preempção e espera circular.
   - **Prevenção mais usada: ordem global de aquisição.** Todos os locks recebem uma ordem fixa (por exemplo, o número da conta) e toda thread os adquire sempre nessa ordem, qualquer que seja a operação. Assim a espera circular, a quarta condição, não pode se formar.
   - Exemplo real: um PIX de Ana para Bruno e outro de Bruno para Ana no mesmo instante. Se cada transferência trava a conta de origem e depois espera a de destino, as duas ficam esperando uma pela outra. Com a ordem global (sempre a conta de menor número primeiro), as duas tentam travar a mesma conta primeiro; uma espera a outra terminar, e não há ciclo. Bancos de dados detectam esse ciclo e abortam uma transação; um programa com locks comuns simplesmente trava.
2. **Inversão de prioridade:** uma tarefa de baixa prioridade segura um lock; uma de alta prioridade precisa dele e espera; uma de prioridade média, que não precisa do lock, ocupa a CPU e impede a de baixa de terminar. Na prática, a média passa na frente da alta. Caso histórico: a sonda Mars Pathfinder (1997) reiniciava sozinha por causa disso.
3. **Convoying (comboio):** se a thread que segura o lock é tirada da CPU (fim da fatia de tempo, falta de página, coleta de lixo), todas as que precisam do lock ficam paradas, mesmo com núcleos livres. Exemplo real: na abertura da venda de ingressos de um show, o servidor com o lock do estoque é pausado e todas as compras param atrás dele.
4. **Não composição:** `sacar()` e `depositar()` são thread-safe, mas `transferir()` = `sacar(a)` seguido de `depositar(b)` não é atômico: entre as duas chamadas, outra thread pode ver o dinheiro fora das duas contas (um relatório de saldo total, por exemplo). Para compor, é preciso conhecer os locks internos dos dois objetos, o que quebra o encapsulamento e convida o deadlock.

**Ideia-chave:** as três primeiras armadilhas têm a mesma causa. **Com lock, o progresso de todas as threads depende de uma thread, a dona do lock, continuar executando.** Lock-free elimina exatamente essa dependência.

**No simulador:**

1. Cenário Deadlock, 12 passos. Nos passos 1 a 5, os dois PIX travam as contas em ordens opostas e o ciclo se forma. Nos passos 6 a 12, a ordem global (sempre A antes de B) impede o ciclo.
2. Modo Com lock, 3 threads, Passo a passo, depois "Suspender no pior momento": o dono do lock é suspenso dentro da seção crítica e nenhuma outra thread avança (convoying). Use "Retomar todas as threads" para seguir.

### Slide 12 · Linearizabilidade: o critério de correção

**A pergunta:** o que quer dizer uma fila concorrente estar certa? Numa fila sequencial é fácil: o primeiro a entrar é o primeiro a sair. Com threads, as operações se sobrepõem no tempo. Se `enq(x)` e `enq(y)` acontecem ao mesmo tempo, quem entrou primeiro?

**Definição (Herlihy e Wing, 1990):** um objeto é linearizável se toda operação parece ter efeito instantâneo em algum ponto entre a chamada e o retorno, o **ponto de linearização**. Ordenando as operações por esses pontos, o resultado precisa ser uma execução válida da estrutura sequencial.

**Como ler o diagrama:**

- Cada barra é uma operação: começa quando a thread chama o método e termina quando o método retorna.
- A thread A chama `enq(x)` enquanto a thread B chama `enq(y)`: as barras se sobrepõem. Depois A chama `deq()` e recebe x.
- Os círculos laranja são os pontos de linearização. Como o de `enq(x)` vem antes do de `enq(y)`, a ordem equivalente é `enq(x)`, `enq(y)`, `deq() → x`, uma execução FIFO válida.
- Se `deq()` tivesse devolvido y, também seria aceitável: as barras se sobrepõem e o ponto de `enq(y)` poderia vir antes. Quando duas operações são concorrentes, qualquer ordem entre elas é válida. Mas se `enq(x)` terminasse antes de `enq(y)` começar, `deq()` teria de devolver x.

**Exemplo real:** dois PIX enviados no mesmo segundo, de celulares diferentes. O extrato mostra uma ordem, e qualquer uma é aceitável, mas o saldo final precisa ser consistente com alguma ordem.

**Por que é o critério mais usado:**

1. Permite raciocinar como se o código fosse sequencial: basta saber que cada operação "acontece de uma vez".
2. É composicional: se cada objeto do sistema é linearizável, o sistema todo é.
3. Respeita o tempo real: se uma operação termina antes de outra começar, a ordem é preservada.

**Onde fica o ponto de linearização:** com lock, em qualquer instante dentro da seção crítica; em lock-free, quase sempre no CAS bem-sucedido.

**Comparação:** a consistência sequencial é mais fraca: não exige respeitar o tempo real entre threads diferentes e não é composicional.

### Slide 13 · Garantias de progresso

Linearizabilidade diz **se** o resultado está certo. Progresso diz **quem** consegue terminar. São propriedades independentes: uma estrutura pode ser correta e travar (deadlock).

**As quatro garantias, da mais fraca para a mais forte:**

1. **Bloqueante:** o progresso de uma thread pode depender de outra. Se a dona do lock for suspensa, todas as que precisam do lock esperam indefinidamente. Toda estrutura com lock é bloqueante. Exemplo real: estoque de ingressos com lock; se o servidor que segura o lock parar, ninguém compra.
2. **Obstruction-free:** sem locks, mas só garante que uma thread termina se rodar isolada por tempo suficiente. Com várias se atrapalhando, pode ocorrer **livelock**: todas trabalham e nenhuma conclui. Exemplo real: edição de uma página de wiki; você salva se ninguém editou no meio, mas dois editores insistentes podem se derrubar indefinidamente.
3. **Lock-free:** a cada momento, pelo menos uma thread completa sua operação em número finito de passos. O sistema sempre avança, mas uma thread específica pode perder sempre (**starvation**, inanição). Exemplo real: reserva de assento com CAS; algum cliente sempre confirma, embora um azarado possa perder várias vezes. Em código: `ConcurrentLinkedQueue` e `AtomicLong`.
4. **Wait-free:** toda thread completa sua operação em número limitado de passos, faça o que fizerem as outras. É a garantia mais forte e, em geral, a mais cara. Exemplo real: contador de visualizações com fetch-and-add de hardware (`LOCK XADD` no x86).

**Relação:** todo wait-free é lock-free, e todo lock-free é obstruction-free. Lock-free e wait-free juntas são chamadas de **não bloqueantes** (*nonblocking*).

**Termos que costumam ser confundidos:** *deadlock-free* (alguma thread consegue adquirir o lock) e *starvation-free* (toda thread consegue) são as versões bloqueantes de lock-free e wait-free.

|  | Alguém avança | Todos avançam |
| --- | --- | --- |
| Bloqueante | deadlock-free | starvation-free |
| Não bloqueante | lock-free | wait-free |

### Slide 14 · Definição de lock-free e três mitos

**Definição:** uma implementação é lock-free se o sistema nunca fica parado: em qualquer instante, alguma thread conclui sua operação em um número finito de passos, mesmo que outras tenham sido suspensas para sempre.

**Teste mental:** se uma thread qualquer for suspensa no pior momento possível, as outras ainda conseguem terminar? Com lock: não, se a suspensa segurava o lock. Com lock-free: sim. Por isso lock-free elimina deadlock, convoying e inversão de prioridade.

**Como se obtém:** ler o estado, calcular o novo localmente e tentar publicá-lo com uma única instrução atômica que só grava se ninguém mexeu no meio (essa instrução, o CAS, é o assunto da Parte 2). Se a publicação falhar, é porque outra thread publicou antes, ou seja, alguém avançou.

**Três mitos:**

1. **"É código sem a palavra lock."** Errado. Lock-free é propriedade de progresso, não de sintaxe. Um spin lock feito com CAS (`while (!flag.compareAndSet(false, true)) {}`) não tem a palavra lock e é bloqueante: se a thread que pôs a flag em true for suspensa, as outras giram para sempre.
2. **"É sempre mais rápido."** Errado. Um lock sem disputa custa praticamente o mesmo que um CAS. Sob alta disputa, CAS falham e são repetidos. A vantagem principal é robustez e latência previsível, não necessariamente vazão.
3. **"Ninguém espera."** Errado. Uma thread pode perder a disputa repetidas vezes. A garantia de que todas terminam é wait-free.

**Exemplo real:** na venda de ingressos, se o servidor que segura o lock do estoque é pausado, todas as compras param. Com a reserva feita sem lock, o servidor pausado não segura nada e as outras compras continuam.

**No simulador:** modo Lock x lock-free, 3 threads, Passo a passo. Avance alguns passos e clique em "Suspender no pior momento". À esquerda, o dono do lock é suspenso e todas as threads param. À direita, a thread suspensa não segura nada e as outras continuam gravando. Isso é a definição de lock-free em ação.

## Parte 2 · CAS (slides 15 a 18)

### Slide 15 · CAS: a base do lock-free

O problema da corrida é a sequência ler, calcular, gravar ser interrompida no meio. Com lock, impedimos a interrupção. Para fazer lock-free, precisamos de outra coisa: uma operação que o próprio hardware garanta ser indivisível e que diga se alguém mexeu no dado. Essa operação existe em praticamente todo processador moderno e se chama **compare-and-swap (CAS)**.

A ideia é a mesma do `git push` recusado quando o remoto mudou, agora no nível de uma instrução de máquina.

Esta parte cobre o que a instrução CAS faz, por que ela é a base de todo algoritmo lock-free (números de consenso) e como usá-la para consertar o saldo sem lock.

### Slide 16 · Compare-and-swap (CAS): como funciona

**O que faz:** CAS recebe três argumentos: um endereço de memória, o valor que você **espera** encontrar lá e o valor **novo**. Atomicamente (como uma única operação indivisível, garantida pelo hardware), compara o conteúdo do endereço com o esperado. Se forem iguais, grava o novo e devolve true. Se forem diferentes, não grava nada e devolve false.

```
// executado atomicamente pelo hardware
boolean CAS(endereco, esperado, novo) {
  if (*endereco == esperado) {
    *endereco = novo;
    return true;
  }
  return false;
}
```

**Interpretação:** "eu li o valor V; se ninguém mexeu desde então, troque por N". Se o CAS falhar, houve interferência, e a thread relê e tenta de novo. Isso é sincronização **otimista**: em vez de pedir permissão antes (lock, pessimista), faz-se o trabalho e só se confirma se ninguém mexeu.

**Exemplos reais:**

- **Reserva de assento (cinema, avião, show):** o app só confirma o 14C se ele ainda estiver livre naquele instante. Se outra pessoa reservou antes, a confirmação falha, o mapa é recarregado e você escolhe outro.
- **git push:** o servidor só aceita o push se a ponta do branch remoto ainda for o commit que você tinha quando fez pull. Se alguém fez push antes, o seu é recusado; você faz pull, integra e tenta de novo. É um loop de CAS.
- **Bancos de dados:** controle otimista com coluna de versão: `UPDATE ... SET valor = ?, versao = versao + 1 WHERE id = ? AND versao = ?`. Se nenhuma linha for afetada, alguém alterou antes.

**No hardware:**

- **x86:** instrução `CMPXCHG` com prefixo `LOCK`, que garante atomicidade mesmo com vários núcleos.
- **ARM:** par LL/SC (*load-linked / store-conditional*; `LDXR` e `STXR`). O LL lê e "marca" o endereço; o SC só grava se ninguém escreveu ali desde o LL. O ARMv8.1 acrescentou uma instrução CAS direta.

**Nas linguagens:**

- **Java:** `AtomicInteger`, `AtomicLong`, `AtomicReference`, com `compareAndSet(esperado, novo)`.
- **C++:** `std::atomic`, com `compare_exchange_strong` e `compare_exchange_weak`. A versão weak pode falhar mesmo quando o valor é igual (falha espúria), por isso é usada dentro de laços.
- **Rust:** `compare_exchange` e `compare_exchange_weak`.

**Leitura:** livro-base, capítulo 5 e apêndice B.

### Slide 17 · Números de consenso: o poder do CAS

**O problema do consenso:** várias threads, cada uma com um valor proposto, precisam decidir todas o mesmo valor (um dos propostos), sem que nenhuma fique esperando outra. O **número de consenso** de uma primitiva é a quantidade máxima de threads que ela consegue fazer concordar assim (Herlihy, 1991). Quanto maior, mais poderosa a primitiva.

**A hierarquia do slide:**

- **Leitura e escrita atômicas: 1.** Só com ler e gravar, nem duas threads conseguem concordar sem uma esperar a outra: sempre existe uma intercalação em que uma não sabe se a outra já decidiu.
- **Test-and-set, swap, fetch-and-add: 2.** Resolvem para duas threads (quem chega primeiro vence, e a perdedora adota o valor da vencedora), mas não para três: a perdedora não sabe qual das outras venceu.
- **Fila ou pilha compartilhada: 2.** Consequência surpreendente: não dá para construir uma fila wait-free para três ou mais threads usando apenas filas de duas threads e registradores.
- **Compare-and-swap e LL/SC: infinito.** Todas tentam `CAS(decisao, vazio, meuValor)`; a primeira vence e todas leem o mesmo resultado, para qualquer número de threads.

**Conclusão:** por isso o CAS basta para construir qualquer estrutura lock-free ou wait-free (**universalidade**, capítulo 6), e todo processador moderno o oferece. A construção universal é mais teórica do que prática, mas prova que o CAS é a ferramenta certa.

**Exemplo real:** dois motoristas aceitam a mesma corrida no app quase no mesmo instante. Com CAS no campo "motorista" (só grava se ainda estiver vazio), o primeiro aceite vence e os dois veem o mesmo resultado. Em infraestrutura, a eleição de líder num cluster (etcd, ZooKeeper) usa a mesma ideia.

**Definições rápidas:**

- **Test-and-set:** marca uma flag como 1 e devolve o valor que ela tinha antes.
- **Swap:** troca o valor da memória pelo seu e devolve o antigo.
- **Fetch-and-add:** soma um valor e devolve o anterior (é o `getAndIncrement`).
- **LL/SC:** a forma do ARM de fazer o mesmo que o CAS, em duas instruções: lê e marca o endereço, e só grava se ninguém escreveu ali desde a marca.

**Leitura:** HERLIHY (1991); livro-base, capítulos 5 e 6 (sem se prender às provas).

### Slide 18 · Contador lock-free com CAS

O mesmo saldo do início, agora correto e sem lock:

```java
AtomicLong saldo = new AtomicLong();

void depositar() {
  long atual, novo;
  do {
    atual = saldo.get();     // ler
    novo  = atual + 1;       // calcular
  } while (!saldo.compareAndSet(atual, novo));
}
```

**Linha a linha:**

- `saldo` é um `AtomicLong`, um `long` com operações atômicas.
- `atual = saldo.get()`: **lê** o valor atual (por exemplo, 41).
- `novo = atual + 1`: **calcula** localmente (42). Nenhuma outra thread vê esse valor.
- `saldo.compareAndSet(atual, novo)`: tenta **gravar** 42, mas só se o saldo ainda for 41. Se for, grava e sai do laço. Se outra thread mudou o saldo, o CAS falha e o laço recomeça: relê 42, calcula 43, tenta de novo.

**De volta à corrida do slide 4:** lá, a thread B gravava 42 por cima do 42 da thread A. Agora o CAS da thread B falha (esperava 41 e encontrou 42), ela relê e grava 43. Nenhum depósito se perde: o passo 3 **detecta** a interferência em vez de sobrescrever.

**Este padrão (ler, calcular, CAS, repetir) é a base de praticamente todo algoritmo lock-free.** Na pilha e na fila, só muda o "calcular": em vez de somar 1, monta-se um nó novo e troca-se um ponteiro.

**Por que é lock-free:** o CAS de uma thread só falha se o CAS de outra thread teve sucesso. Toda falha corresponde a um sucesso de alguém, então o sistema sempre avança. Uma thread suspensa no meio do laço não segura nada.

**Por que não é wait-free:** uma thread azarada pode perder a disputa indefinidamente (na prática, é raríssimo).

**Ponto de linearização:** o CAS bem-sucedido, o instante em que o depósito "acontece".

**Na prática do Java:**

- `incrementAndGet()` faz isso por você (no x86, a JVM usa `LOCK XADD`, um fetch-and-add de hardware que nem precisa de laço).
- `LongAdder` (Java 8): mantém várias células; cada thread tende a incrementar uma diferente, e `sum()` soma todas. Elimina a disputa num único endereço. Ideal para métricas e contadores de visualizações, em que se escreve muito e se lê pouco.

**Leitura:** GOETZ et al. (2006), capítulo 15, que usa exatamente este exemplo.

**No simulador:** modo Lock-free (CAS), 3 threads, Passo a passo. Quando duas threads leem o mesmo valor, o CAS da segunda falha em vermelho, ela relê e tenta de novo. "Depósitos feitos" e "Saldo na memória" continuam iguais.

## Parte 3 · Estruturas lock-free (slides 19 a 25)

### Slide 19 · Estruturas lock-free

O contador tinha um único valor numérico. Estruturas de dados de verdade são feitas de nós ligados por ponteiros. A ideia continua a mesma (ler, preparar, CAS), mas agora o CAS troca um ponteiro, e aparecem dois problemas novos: **o problema ABA** e **a pergunta de quando é seguro liberar a memória de um nó removido**.

Roteiro da parte: pilha de Treiber (a mais simples), problema ABA, liberação de memória, fila de Michael-Scott (a mais usada na prática), ordem de memória e o que esperar de desempenho.

### Slide 20 · Pilha lock-free de Treiber

Publicada por R. Kent Treiber (IBM, 1986), é o "hello world" das estruturas lock-free.

**Estrutura:** uma lista encadeada em que o topo da pilha é o primeiro nó. **Toda a sincronização está num único ponteiro atômico, `top`.** Cada nó tem um valor e um ponteiro `next` para o nó de baixo.

**push(v):**

1. Cria um nó novo `n` com o valor v. Nenhuma outra thread conhece esse nó, então ele pode ser alterado à vontade.
2. Lê o topo atual e faz `n.next` apontar para ele. A pilha ainda não mudou.
3. Tenta `CAS(top, topoAntigo, n)`: "se o topo ainda for o que eu li, passe a apontar para n".
4. Se o CAS falhar, outra thread mudou o topo. Volta ao passo 2.

**pop():**

1. Lê o topo `t`. Se for null, a pilha está vazia: devolve null.
2. Tenta `CAS(top, t, t.next)`: "se o topo ainda for t, passe a apontar para o nó de baixo".
3. Se falhar, recomeça. Se der certo, devolve `t.value`.

**Ponto de linearização:** o CAS bem-sucedido, em push e em pop.

**Exemplo de execução:** pilha C, B, A. Duas threads fazem `push(D)` e `push(E)` ao mesmo tempo e ambas leem `top = C`. A thread do D faz o CAS primeiro e vence. O CAS da thread do E falha (esperava C, encontrou D); ela relê `top = D`, faz `E.next = D` e tenta de novo, com sucesso. Resultado: E, D, C, B, A. Nenhum push se perdeu.

**Por que é lock-free:** toda falha de CAS significa que o CAS de outra thread teve sucesso. Uma thread suspensa entre a leitura e o CAS não bloqueia ninguém.

**Exemplo real:** revenda de ingressos devolvidos. Cada devolução entra no topo e o próximo comprador recebe o ingresso de cima; devoluções e compras simultâneas disputam o topo. Pools de objetos e alocadores de memória usam a mesma pilha.

**Limitação:** `top` é um ponto único de disputa. Com muitas threads, quase todos os CAS falham. A **pilha com eliminação** (capítulo 11) deixa um push e um pop que se encontram trocarem o valor diretamente, sem tocar em `top`.

**Detalhe importante:** o código funciona em Java porque cada push cria um nó novo e o coletor de lixo não recicla nós ainda referenciados. Em C ou C++, esse mesmo código tem um bug sério: o problema ABA.

### Slide 21 · Problema ABA: quando o CAS se engana

O nome vem da sequência de valores: o ponteiro vale A, muda para B e volta para A.

**A raiz do problema:** o CAS pergunta "o valor ainda é A?", e não "alguém mexeu aqui desde que eu li?". Se o valor saiu de A e voltou para A, o CAS acha que nada mudou.

**Os quatro passos:**

1. **Thread 1 lê o topo.** A pilha é A, B, C. A thread 1 começa um pop: lê `top = A` e `A.next = B`, e pretende fazer `CAS(top, A, B)`. Antes disso, é suspensa.
2. **Thread 2 mexe na pilha.** Remove A, remove B e empilha um valor novo; o alocador (ou um pool de nós) devolve exatamente o mesmo endereço que A tinha. Agora a pilha é A, C, e B foi liberado.
3. **Thread 1 volta e tenta.** Executa `CAS(top, A, B)`. O topo tem o endereço A, então o CAS tem sucesso.
4. **Pilha corrompida.** `top` aponta para B, um nó que já saiu da pilha e pode ter sido liberado. O nó recém-empilhado sumiu, e o próximo push que reaproveitar B vai sobrescrever um nó que a pilha ainda usa. A falha aparece muito depois, longe da causa.

**Por que é tão perigoso:** depende de uma intercalação rara e de reuso de memória; passa em testes e aparece em produção sob carga.

**Exemplo real:** um sistema reaproveita o número de um pedido cancelado para um pedido novo. Um estorno atrasado que só confere "o pedido ainda é o 1042?" encontra o mesmo número e estorna o pedido errado.

**Quando ocorre:**

- **C, C++ e Rust:** sempre que memória liberada é reaproveitada, ou quando se usa um pool de nós.
- **Java com nós sempre novos: não ocorre**, porque o coletor de lixo não reaproveita o endereço de A enquanto a thread 1 ainda guarda referência a ele. Ocorre se o programador usar pool de objetos.
- **Num contador de saldo, é inofensivo:** se o valor sai de 100 e volta a 100, 100 está correto. O problema é com ponteiros e identificadores reaproveitados.

**Correções (slide 22):** ponteiro com contador de versão, hazard pointers, reclamação por épocas ou coleta de lixo. LL/SC também ajuda: o store-conditional falha se o endereço foi escrito, mesmo com o mesmo valor.

**No simulador:** cenário Problema ABA. Primeiro em "Sem versão", 8 passos: no passo 6, o CAS de T1 aceita o endereço 0x10 reciclado; no passo 7, o topo aponta para um nó livre e a transação D sumiu. Depois mude para "Com versão": no passo 6 o mesmo CAS falha, porque a versão mudou, e T1 tenta de novo corretamente.

**Leitura:** livro-base, capítulo 10 (seção sobre reclamação de memória e o problema ABA).

### Slide 22 · Liberação de memória sem lock

O ABA é sintoma de um problema mais geral: sem lock, uma thread pode estar lendo um nó que outra acabou de remover. Se o nó for liberado imediatamente, a leitora acessa memória inválida (*use-after-free*). Com lock isso não acontece, porque quem remove segura o lock. Sem lock, é preciso saber **quando é seguro liberar**.

**Quatro técnicas:**

1. **Ponteiro com versão:** o campo atômico guarda (ponteiro, versão), e toda alteração incrementa a versão. Mesmo que o ponteiro volte a ser A, a versão é diferente e o CAS falha. Em Java, `AtomicStampedReference`. Custo: CAS de 128 bits (`CMPXCHG16B` no x86). Resolve o ABA, mas não diz quando liberar.
2. **Hazard pointers (Maged Michael, 2004):** antes de acessar um nó, a thread publica o endereço dele no seu ponteiro de perigo ("estou usando este, não liberem") e confere se o nó ainda está na estrutura. Quem remove guarda o nó numa lista de aposentados e só libera os que não aparecem em nenhum ponteiro de perigo. Custo: uma barreira de memória por nó lido. Vantagem: memória pendente limitada.
3. **Reclamação por épocas (Keir Fraser, 2004):** um contador global de época. Cada thread anuncia a época ao começar uma operação e deixa de anunciar ao terminar. Um nó removido só é liberado quando todas as threads ativas já estão em épocas posteriores. Custo baixo, mas uma thread parada no meio de uma operação impede toda a liberação. É o que a crate crossbeam usa em Rust.
4. **Coleta de lixo:** em Java, Go e C#, o coletor só libera objetos sem referência, o que resolve a liberação e o ABA (com nós novos). Custo: pausas e memória extra.

**Exemplo real:** o app de uma companhia aérea só pode apagar os dados de um voo cancelado quando nenhuma tela aberta ainda os exibe. No kernel Linux, o **RCU** (*read-copy-update*) faz isso: publica a versão nova e só libera a antiga depois do período de graça, quando terminaram os leitores que podiam vê-la.

**Leitura:** MICHAEL (2004); FRASER (2004); livro-base, capítulos 10 e 19.

### Slide 23 · Fila lock-free de Michael-Scott

A fila lock-free mais usada no mundo (Michael e Scott, PODC 1996), base do `ConcurrentLinkedQueue` do Java.

**Por que é mais difícil que a pilha:** a pilha mexe numa ponta só (`top`). A fila mexe nas duas: enqueue no fim (`tail`) e dequeue no começo (`head`). Em compensação, produtores e consumidores quase não disputam entre si.

**Estrutura:**

- **Nó sentinela:** `head` sempre aponta para um nó vazio; o primeiro elemento real é `head.next`. Assim `head` e `tail` nunca são nulos. A fila está vazia quando `head.next == null`.
- `tail` aponta para o último nó ou, temporariamente, para o penúltimo (tail "atrasado").

**enqueue(z):**

1. Cria o nó z.
2. Lê `tail` (último nó, digamos y) e lê `y.next`.
3. Se `y.next == null`, tenta `CAS(y.next, null, z)`. Se der certo, **este é o ponto de linearização**: z está na fila.
4. Tenta `CAS(tail, y, z)` para avançar `tail`. Esse CAS pode falhar, e tudo bem: alguém já avançou.
5. Se no passo 3 `y.next` não era null, outra thread ligou um nó e ainda não avançou `tail`.

**Ajuda mútua (*helping*), o grande truque:** no caso 5, em vez de esperar a outra thread terminar (o que seria bloqueante: e se ela foi suspensa?), a thread atual avança `tail` ela mesma e recomeça. Nenhuma thread depende de outra terminar. É isso que torna lock-free uma operação de dois passos.

**dequeue():**

1. Lê `head` (sentinela), `tail` e `head.next` (primeiro elemento, x).
2. Se `head.next == null`, a fila está vazia.
3. Se `head == tail` mas `head.next` não é null, `tail` está atrasado: ajuda a avançar e recomeça.
4. Lê o valor de x e tenta `CAS(head, sentinela, x)`. Se der certo, x vira o novo sentinela e o valor é devolvido.

**Exemplo real:** a fila de pedidos de um app de delivery. Clientes inserem pedidos no fim e as cozinhas retiram do começo, ao mesmo tempo. A ajuda mútua aparece quando uma cozinha encontra `tail` atrasado e avança o ponteiro ela mesma.

**Observações:**

- O artigo original também propôs uma fila com dois locks (um para `head`, outro para `tail`), simples e muito usada.
- Em C ou C++, a fila precisa de uma técnica de liberação de memória do slide 22.

**No simulador:** cenário Fila Michael-Scott, 10 passos. Passo 3: o primeiro CAS liga tx2 (ponto de linearização). Passo 4: T1 é suspensa com `tail` atrasado. Passo 6: T2 avança `tail` por ela (ajuda mútua). Passo 9: o CAS de T1 falha sem prejuízo, porque o trabalho já foi feito. Passo 10: um dequeue e o novo sentinela.

**Leitura:** MICHAEL; SCOTT (1996), cerca de 9 páginas com pseudocódigo completo; livro-base, capítulo 10.

### Slide 24 · Ordem de memória e reordenação

**O problema:** para ganhar desempenho, o compilador e o processador reordenam leituras e escritas que, numa thread só, não mudam o resultado. Cada núcleo também tem buffers, e uma escrita pode ficar visível para outro núcleo depois de uma escrita posterior. Com uma thread, ninguém percebe; com várias, sim.

**Exemplo ligado à aula:** na pilha de Treiber, a thread A grava o valor do nó e depois publica o nó com CAS em `top`. Se a escrita do valor ficar visível depois da publicação do ponteiro, a thread B pode pegar o nó e ler um valor não inicializado. É como a notificação "PIX recebido" chegar antes de o saldo estar atualizado.

**As ordens, da mais forte para a mais fraca:**

- **seq_cst:** uma única ordem global, igual para todas as threads. É o padrão em C++ e Rust; `volatile` e `Atomic*` no Java se comportam assim.
- **acquire / release:** uma escrita com release "publica" tudo o que a thread escreveu antes; uma leitura com acquire que a vê passa a enxergar tudo aquilo. É o mínimo para publicar um nó e é o que um lock faz por baixo.
- **relaxed:** só a atomicidade da própria variável. Serve para contadores de estatística.

**Recomendação:** use a ordem padrão (seq_cst) até ter um motivo medido. Erros de ordem de memória dependem da arquitetura: código que funciona no x86 pode falhar no ARM (celulares, Macs com Apple Silicon).

**Nas linguagens:** Java: `volatile`, `Atomic*` e `VarHandle`; C++: `std::memory_order`; Rust: `std::sync::atomic::Ordering`.

**Leitura:** BOS (2023); WILLIAMS (2019), capítulo 5.

### Slide 25 · Desempenho: lock x lock-free

O gráfico do slide é um **esboço da tendência**, não uma medição.

**Mensagens principais:**

1. **Lock-free não é automaticamente mais rápido.** Com pouca disputa, adquirir um lock livre custa praticamente o mesmo que um CAS, porque um lock moderno usa CAS no caminho rápido.
2. **Sob alta disputa, as duas abordagens sofrem com a coerência de cache:** cada gravação numa variável invalida a cópia dela nos outros núcleos, e a linha de cache pula de núcleo em núcleo. No lock-free, muitos CAS falham e são repetidos; no lock, threads ficam paradas ou dormem.
3. **Onde o lock-free costuma ganhar:** degrada com mais suavidade, não sofre convoying e tem latência no pior caso mais previsível. Em sistemas de baixa latência, isso importa mais que a vazão média.

**Remédios para a disputa:**

- **Backoff exponencial:** depois de um CAS que falhou, esperar um tempo aleatório que dobra a cada falha.
- **Espalhar a disputa:** `LongAdder` usa várias células em vez de um contador só.
- **Eliminação:** um push e um pop que se encontram trocam o valor fora do topo da pilha.

**Exemplo real:** o contador de visualizações de um vídeo viral. Milhões de cliques disputam o mesmo valor; por isso as plataformas somam em vários contadores parciais, a ideia do `LongAdder`.

**No simulador:** modo Lock-free (CAS), Contínuo, 8 threads: o contador de CAS que falharam cresce com a disputa. É uma ilustração, não uma medição.

## Parte 4 · Prática e encerramento (slides 26 a 32)

### Slide 26 · Lock-free em bibliotecas e sistemas reais

Nada disso é só teoria acadêmica; provavelmente você já usou várias destas estruturas.

- **Java:** o pacote `java.util.concurrent`, criado por Doug Lea. `ConcurrentLinkedQueue` é a fila de Michael-Scott. `AtomicInteger`, `AtomicLong` e `AtomicReference` expõem CAS. `LongAdder` espalha contadores. `ConcurrentHashMap` mistura CAS e locks finos. Em servidores como Tomcat e frameworks como Netty, filas desse tipo estão no caminho de cada requisição.
- **C++:** `std::atomic` faz parte da linguagem desde o C++11. A Boost.Lockfree oferece fila, pilha e uma fila de um produtor e um consumidor (`spsc_queue`), muito usada em áudio e jogos, onde a thread de tempo real não pode bloquear.
- **Rust:** a crate crossbeam oferece estruturas lock-free e usa reclamação de memória por épocas. Rust não tem coletor de lixo, então o problema da memória é real.
- **Kernel Linux:** RCU permite que leitores percorram listas e tabelas sem lock nenhum. O escritor faz uma cópia, altera a cópia, publica com uma escrita atômica de ponteiro e espera o período de graça para liberar a versão antiga. Ideal quando leituras são muito mais frequentes que escritas.
- **LMAX Disruptor:** biblioteca Java criada por uma bolsa de câmbio de Londres para processar milhões de transações por segundo com latência baixíssima. Usa um buffer circular pré-alocado e contadores de sequência atômicos.
- **Go:** além dos canais (que internamente usam locks), o pacote `sync/atomic` oferece CAS; desde o Go 1.19 há `atomic.Int64` e `atomic.Pointer[T]`.

**Ligação com os exemplos do início:** o contador de curtidas é um `LongAdder`; a fila de pedidos é um `ConcurrentLinkedQueue`; a reserva do último ingresso é um `compareAndSet`.

### Slide 27 · Quando usar lock-free e quando usar lock

**Use lock-free quando:**

- Uma estrutura pequena e muito disputada é o gargalo, e isso foi **medido** com um profiler, não suposto.
- **Latência no pior caso importa mais que vazão média:** sistemas financeiros, áudio, jogos, controle industrial. Uma thread suspensa segurando lock gera picos de latência imprevisíveis.
- **Não se pode bloquear de jeito nenhum:** tratadores de sinal ou de interrupção, threads de tempo real.

**Prefira locks quando:**

- A operação precisa alterar várias estruturas de forma atômica (lembre da não composição: transferir entre duas contas).
- **A disputa é baixa:** o lock custa quase o mesmo e é muito mais simples.
- O código precisa ser mantido por uma equipe. Algoritmos lock-free são difíceis de escrever, revisar e testar; já houve algoritmos publicados com erros descobertos anos depois.

**Regra prática:** use a estrutura concorrente pronta da sua linguagem (`java.util.concurrent`, Boost, crossbeam) antes de escrever a sua.

**Se precisar escrever:** use ferramentas que exploram intercalações e detectam corridas: ThreadSanitizer (C, C++, Go, Rust), jcstress (Java), loom (Rust). E peça revisão a alguém que conheça o assunto.

**Exemplo real:** um site de venda de ingressos não implementa a própria fila lock-free; usa as estruturas de `java.util.concurrent` e gasta o esforço no que é específico do negócio.

### Slide 28 · Resumo: seis ideias principais

1. **Estrutura de dados concorrente:** continua correta qualquer que seja a intercalação. Exemplo: a fila de pedidos.
2. **Corrida:** o saldo que perdeu depósitos, o ingresso vendido duas vezes, o estoque negativo. Sempre o mesmo padrão ler, calcular, gravar.
3. **Linearizabilidade e progresso:** cada operação parece acontecer em um instante (como no extrato bancário); bloqueante, lock-free, wait-free.
4. **Lock-free:** a thread suspensa não trava as outras. Ninguém depende de uma thread específica continuar executando.
5. **CAS:** a reserva de assento e o git push recusado. As armadilhas: ABA (o número de pedido reaproveitado) e liberar memória cedo demais (apagar o voo enquanto uma tela ainda o exibe).
6. **Engenharia:** mais threads nem sempre é mais rápido, por causa da disputa. Meça antes de otimizar e prefira bibliotecas prontas.

**Em uma frase:** lock-free não é sobre tirar a palavra lock do código; é sobre garantir que nenhuma thread precise esperar outra para o sistema continuar andando.

### Slide 29 · Referência principal: livro-base

*The Art of Multiprocessor Programming* é o livro de referência da área. Maurice Herlihy e Nir Shavit receberam o Prêmio Gödel em 2004 e o Prêmio Dijkstra em 2012. A 2ª edição (2020), com Victor Luchangco e Michael Spear, mantém a estrutura dos capítulos 1 a 12 e acrescenta o capítulo 19 (otimismo e gerenciamento manual de memória). A tabela do slide usa a numeração da 2ª edição.

**Roteiro de estudo sugerido:**

1. **Capítulo 1:** leia inteiro. É curto e dá a motivação.
2. **Capítulo 3:** linearizabilidade e condições de progresso. É o coração teórico da aula.
3. **Capítulo 9:** introdução e as seções de coarse-grained e lock-free.
4. **Capítulo 10:** fila de Michael-Scott e o problema ABA.
5. **Capítulo 11:** pilha de Treiber e pilha com eliminação.
6. **Capítulo 5:** números de consenso, sem se prender às provas.

**Dica:** no fim de cada capítulo há uma seção de notas com o histórico e os artigos originais.

### Slide 30 · Referências: artigos originais

- **AMDAHL (1967):** Lei de Amdahl.
- **HERLIHY; WING (1990):** definição de linearizabilidade.
- **HERLIHY (1991):** números de consenso, universalidade e wait-free. Rendeu a Herlihy o Prêmio Dijkstra de 2003.
- **TREIBER (1986):** a pilha lock-free.
- **MICHAEL; SCOTT (1996):** a fila lock-free (e a versão com dois locks).
- **MICHAEL (2004):** hazard pointers.
- **FRASER (2004):** reclamação de memória por épocas.

**Leituras recomendadas:** o artigo de Michael e Scott (cerca de 9 páginas, com pseudocódigo completo) e as primeiras seções do artigo de Herlihy e Wing.

### Slide 31 · Referências complementares por linguagem

- **GOETZ et al. (2006):** referência prática em Java; o capítulo 15 trata de variáveis atômicas e sincronização não bloqueante.
- **WILLIAMS (2019):** referência em C++; capítulo 5 (modelo de memória) e capítulo 7 (estruturas lock-free).
- **BOS (2023):** o texto mais didático sobre atômicos e ordem de memória, em Rust; leitura online gratuita no site da autora.
- **McKENNEY:** livro gratuito do principal desenvolvedor do RCU no Linux.

### Slide 32 · Obrigado!

**Para continuar:** capítulos 10 e 11 do livro-base, sobre filas e pilhas concorrentes.

**Perguntas frequentes:**

- **`volatile` resolve o `saldo++`?** Não. Garante visibilidade e ordem, mas ler, calcular e gravar continuam separados. É preciso `AtomicLong` ou `synchronized`.
- **Thread-safe é o mesmo que lock-free?** Não. Thread-safe é correção; lock-free é garantia de progresso.
- **Banco de dados usa lock-free?** Usa as duas coisas: locks para transações e técnicas otimistas (MVCC, coluna de versão), que são a ideia do CAS em escala maior.
- **Lock-free funciona entre máquinas diferentes?** CAS é uma instrução de um processador. Entre máquinas, a ideia análoga é a escrita condicional de sistemas distribuídos (etcd, Redis com WATCH, coluna de versão no banco).
- **Por que não usar wait-free sempre?** Costuma ser bem mais lento e complexo. Lock-free é o meio-termo mais usado.
- **E memória transacional?** Marca um bloco como atômico e o sistema cuida do resto. Resolve a composição dos locks; é o capítulo 20 do livro-base.

## Questões para revisão

Tente responder antes de ler a resposta.

1. **Uma pilha protegida por um mutex pode ser chamada de lock-free?**
   Não. Se a thread que segura o mutex for suspensa, nenhuma outra consegue completar operações: é bloqueante por definição. Mesmo um spin lock feito com CAS continua bloqueante (Mito 1 do slide 14).
2. **Um loop de CAS em que uma thread pode perder a disputa para sempre é lock-free ou wait-free?**
   Lock-free, mas não wait-free. Toda falha do CAS dessa thread corresponde a um sucesso de outra, então o sistema avança. Mas aquela thread pode sofrer starvation, e wait-free exige que toda thread termine em número limitado de passos.
3. **Em Java, criando sempre nós novos, a pilha de Treiber sofre o problema ABA?**
   Não, nessa forma. Enquanto a thread 1 guarda referência ao nó A, o coletor de lixo não pode reciclá-lo, então um "novo A" no mesmo endereço não aparece. O ABA aparece em Java com pool de objetos e é um problema real em C, C++ e Rust.
4. **Se a máquina tiver um único núcleo, a condição de corrida do saldo ainda acontece?**
   Sim. O sistema operacional pode trocar de thread entre a leitura e a gravação (preempção).
5. **Por que a ordem global de aquisição de locks impede o deadlock?**
   Porque elimina a espera circular: se todas as threads adquirem os locks na mesma ordem, nenhuma pode segurar um lock "maior" enquanto espera um "menor".
6. **Onde fica o ponto de linearização do enqueue na fila de Michael-Scott?**
   No CAS bem-sucedido que liga o novo nó ao `next` do último nó, e não no CAS que avança `tail`.

## Glossário

- **Atomicidade:** propriedade de uma operação indivisível; nenhuma outra thread vê um estado intermediário.
- **CAS (compare-and-swap):** instrução que grava um valor novo somente se o atual for igual ao esperado, devolvendo se teve sucesso.
- **Condição de corrida:** resultado que depende da ordem de intercalação das threads.
- **Deadlock:** threads esperando umas pelas outras em ciclo, sem que nenhuma avance.
- **Linearizabilidade:** cada operação parece ter efeito instantâneo entre a chamada e o retorno.
- **Livelock:** threads ativas que se atrapalham indefinidamente sem concluir.
- **Lock (mutex):** objeto que só uma thread pode segurar por vez, usado para exclusão mútua.
- **Lock-free:** garantia de que, a cada momento, alguma thread conclui sua operação.
- **Seção crítica:** trecho que acessa dado compartilhado e exige exclusão mútua.
- **Starvation:** uma thread específica nunca consegue concluir.
- **Wait-free:** garantia de que toda thread conclui em número limitado de passos.

## Referências

AMDAHL, G. M. Validity of the single processor approach to achieving large scale computing capabilities. In: AFIPS SPRING JOINT COMPUTER CONFERENCE, 1967. *Proceedings* [...]. New York: ACM, 1967. p. 483-485.

BOS, M. *Rust Atomics and Locks*. Sebastopol: O'Reilly, 2023.

FRASER, K. *Practical lock-freedom*. 2004. Tese (Doutorado) - University of Cambridge, Cambridge, 2004.

GOETZ, B. et al. *Java Concurrency in Practice*. Boston: Addison-Wesley, 2006.

HERLIHY, M. Wait-free synchronization. *ACM Transactions on Programming Languages and Systems*, v. 13, n. 1, p. 124-149, 1991.

HERLIHY, M.; WING, J. M. Linearizability: a correctness condition for concurrent objects. *ACM Transactions on Programming Languages and Systems*, v. 12, n. 3, p. 463-492, 1990.

HERLIHY, M.; SHAVIT, N.; LUCHANGCO, V.; SPEAR, M. *The Art of Multiprocessor Programming*. 2. ed. Cambridge: Morgan Kaufmann, 2020.

McKENNEY, P. E. *Is Parallel Programming Hard, And, If So, What Can You Do About It?* Disponível online.

MICHAEL, M. M. Hazard pointers: safe memory reclamation for lock-free objects. *IEEE Transactions on Parallel and Distributed Systems*, v. 15, n. 6, p. 491-504, 2004.

MICHAEL, M. M.; SCOTT, M. L. Simple, fast, and practical non-blocking and blocking concurrent queue algorithms. In: ACM SYMPOSIUM ON PRINCIPLES OF DISTRIBUTED COMPUTING (PODC), 15., 1996. *Proceedings* [...]. New York: ACM, 1996. p. 267-275.

TREIBER, R. K. *Systems programming: coping with parallelism*. San Jose: IBM Almaden Research Center, 1986. (Research Report RJ 5118).

WILLIAMS, A. *C++ Concurrency in Action*. 2. ed. Shelter Island: Manning, 2019.
