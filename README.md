# 🌐 **Projeto: Implementação de um Web Server HTTP Concorrente (C/POSIX)**

## 🧠 **A Filosofia e o Desafio Arquitetural**
 No ecossistema de desenvolvimento moderno, a comunicação de rede é frequentemente ofuscada por pesadas abstrações (frameworks web, reverse proxies, runtimes gerenciados). O objetivo primário deste projeto não é competir com o Nginx ou Apache, mas sim **desconstruir a abstração**.

 Desenvolver um servidor HTTP/1.1 em **C puro** exige confrontar diretamente a dura realidade da engenharia de sistemas: a rede é não-confiável, a memória é finita e a concorrência gera condições de corrida. Este projeto é um estudo rigoroso sobre:

 1. **O Modelo OSI na Prática:** A transição do fluxo de bytes brutos da Camada 4 (TCP) para o protocolo semântico da Camada 7 (HTTP), lidando com fragmentação de pacotes, reconstrução de *streams* e latência.
 2. **Gerenciamento de Estado sem Garbage Collector:** Como alocar, rastrear e liberar memória (buffers de requisição e resposta) em um ambiente de alta concorrência sem introduzir *Memory Leaks* ou *Use-After-Free*.
 3. **A Evolução da Concorrência:** O projeto está estruturado em uma jornada arquitetural clara. Começamos estabelecendo uma base sólida com as regras universais do UNIX (Versão 1 - POSIX Multiprocesso) para, em seguida, quebrarmos essas regras em busca de performance extrema utilizando APIs específicas de Kernel (Versão 2 - Event-Driven/kqueue).

---

# 🎯 **O Projeto (Versão 1 - Portabilidade e Sincronismo)**
 Esta primeira iteração foca na construção de um servidor web robusto, determinístico e portável. A implementação utiliza a **API de Sockets de Berkeley** genérica (POSIX) e operações de I/O padrão, garantindo que o código rode nativamente tanto em **FreeBSD** quanto em **Linux**.

 A grande inovação técnica desta versão é o uso do **Self-Pipe Trick**, que permite tratar sinais assíncronos (como a morte de processos filhos) de forma síncrona dentro de um loop de eventos, eliminando condições de corrida clássicas em C. O servidor serve como base para uma aplicação de **Jogo da Velha Multiplayer**, sincronizando o estado via JSON.

## ⚡ **Definição dos Métodos Suportados**
 | Método | Comportamento no Servidor | Finalidade Técnica |
 | --- | --- | --- |
 | **GET** | Leitura via I/O padrão (`fread`) | Recuperação de recursos estáticos ou estado JSON da sala. |
 | **POST** | Escrita integral (`fopen "w"`) | Criação de novos estados (ex: abrir uma nova sala). |
 | **PUT** | Escrita Idempotente | Atualização completa de um recurso (ex: realizar uma jogada). |
 | **PATCH** | Modificação atômica parcial | Atualização segmentada de recursos existentes. |
 | **DELETE** | Remoção via syscall `remove` | Exclusão definitiva de um recurso (ex: fechar uma sala). |

---

# 😈 **O Ambiente de Desenvolvimento: Por que FreeBSD?**
 Embora o servidor na Versão 1 seja multiplataforma, o **FreeBSD** foi escolhido como o ecossistema primário de pesquisa, arquitetura e validação. Para engenharia de software de baixo nível (Systems Programming), o FreeBSD oferece vantagens estruturais e ferramentas analíticas que superam alternativas tradicionais.

## 📜 **1. A Origem Histórica: O Berço dos Sockets**
 O **BSD (Berkeley Software Distribution)** foi o laboratório onde a pilha TCP/IP moderna foi forjada. A **API de Sockets**, adotada universalmente hoje, foi introduzida no **4.2BSD** em 1983. Desenvolver sobre o FreeBSD é trabalhar na implementação "de referência" das redes UNIX.

 > **Referência Oficial:** [FreeBSD Developers Handbook - Sockets Programming](https://docs.freebsd.org/en/books/developers-handbook/sockets/)

## 🏗️ **2. O "Base System" Coeso**
 Diferente do Linux, que é apenas um Kernel combinado com utilitários GNU de terceiros, o FreeBSD desenvolve o Kernel, a biblioteca padrão C (`libc`) e as ferramentas de *User Space* como um **único repositório coeso (Base System)**. Isso significa que o comportamento das *syscalls*, a implementação da memória e a documentação estão sempre em perfeita sincronia, eliminando a ambiguidade comum ao se debugar problemas profundos de integração.

## 🔬 **3. Observabilidade Absoluta com DTrace**
 Para construir servidores de alta performance, "achismo" não funciona. O FreeBSD integra nativamente o **DTrace (Dynamic Tracing Framework)**. Ele permite instrumentar e rastrear o servidor em tempo real e em produção, mapeando exatamente quantos milissegundos o Kernel gasta alocando buffers de rede (mbufs), realizando trocas de contexto (*context switches*) no `fork()`, ou travando em operações de I/O de disco, sem precisar alterar uma linha de código C ou recompilar o servidor.

## 🛡️ **4. Paradigmas Superiores de Arquitetura (Roadmap V2)**
 O FreeBSD expõe primitivas de Kernel consideradas o estado da arte para escalabilidade e segurança de rede:

 * **kqueue vs epoll:** O `kqueue` do FreeBSD não monitora apenas Sockets de rede, mas unifica o monitoramento de processos (`SIGCHLD`), timers, I/O assíncrono e eventos de sistema de arquivos (vnodes) em uma única API elegante.
 * **Segurança Ofensiva/Defensiva (Capsicum):** Enquanto containers dependem de namespaces complexos, o FreeBSD permite que um servidor drope seus próprios privilégios e entre em um "Capability Mode" (`Capsicum`). Se um atacante explorar um *Buffer Overflow* na função de *parsing* HTTP deste servidor, o *Capsicum* bloqueará fisicamente no Kernel qualquer tentativa de abrir novos arquivos ou sockets maliciosos.

---

# 🚀 **Destaques da Implementação Técnica (V1)**
 Para garantir resiliência e portabilidade, o servidor foi construído sobre uma arquitetura de **Event Loop Síncrono** mesmo utilizando multiprocessamento.

## ⚡ **Gerenciamento de Concorrência: Self-Pipe Trick + poll()**
 * **Self-Pipe Trick:** Sinais UNIX são inerentemente assíncronos e perigosos. Para domá-los, o servidor cria um `pipe()` interno. O tratador de sinais (`SIGCHLD`) apenas escreve um byte no cano.
 * **Loop Unificado:** O daemon principal utiliza `poll()` para monitorar simultaneamente o socket do servidor e a ponta de leitura do `pipe`. Isso transforma interrupções de hardware em eventos de I/O síncronos.
 * **Isolamento via Fork:** Cada conexão é processada por um worker isolado. Se um processo filho falha, o Kernel emite o sinal, o `poll()` detecta o evento no cano e o pai realiza o `waitpid()` (reaping) de forma limpa, sem nunca bloquear.

### 🗺️ **Arquitetura de Concorrência Síncrona**
 ```mermaid
 sequenceDiagram
     participant C as Cliente HTTP
     participant M as Master Daemon (poll)
     participant P as Self-Pipe (Internal)
     participant OS as Kernel (POSIX)
     participant W as Worker (Fork)

     M->>P: setup pipe() & sigaction()
     loop Event Loop
         M->>OS: poll(server_fd, pipe_read_fd)

         alt Novo Cliente (Rede)
             OS-->>M: POLLIN no server_fd
             M->>W: syscall: fork()
             W->>C: Zero-Copy Parsing & Response
             W->>OS: _exit(0)

         else Sinal (Worker Morreu)
             OS->>M: Envia SIGCHLD
             M->>P: write(pipe_write_fd, signo)
             OS-->>M: POLLIN no pipe_read_fd
             M->>P: read() consome sinal do cano
             M->>OS: waitpid(WNOHANG) -> Zombie Reaped!
         end
     end
 ```

## 🧩 **Parsing de Protocolo Zero-Copy (In-situ)**
 * **Mecânica de Memória:** O parser opera diretamente no buffer da pilha. Delimitadores HTTP (`\r\n`, espaços, `?`) são substituídos por terminadores nulos (`\0`).
 * **Otimização de Ponteiros:** A estrutura `http_request_t` mapeia seus ponteiros de caminho, versão e cabeçalhos diretamente para os endereços dentro do buffer original, eliminando `malloc` ou duplicação de strings.

---

# 🔮 **Roadmap e Evolução Arquitetural (Versão 2)**
 A Versão 2 abandonará a portabilidade POSIX básica para abraçar a performance extrema "bare-metal", focando em escalabilidade horizontal.

### **Proposta Técnica V2: Event-Driven & Zero-Copy I/O**
 ```mermaid
 graph TD
     subgraph "Kernel Space"
         K_NET[Network Stack]
         K_SIG[Signals]
         K_FS[Filesystem Cache]
     end

     subgraph "V2 Engine (User Space)"
         EP[Event Multiplexer: kqueue / epoll]
         TQ[Task Queue]

         subgraph "Thread Pool"
             T1[Thread 1]
             T2[Thread 2]
             TN[Thread N]
         end
     end

     K_NET -- "O(1) Event" --> EP
     K_SIG -- "Signal Event" --> EP
     EP -- "Dispatch" --> TQ
     TQ -- "Pop" --> T1

     T1 -- "syscall: sendfile()" --> K_FS
     K_FS -- "Direct DMA" --> K_NET

     style EP fill:#3b82f6,stroke:#fff,color:#fff
     style TQ fill:#1e293b,stroke:#fff,color:#fff
 ```

 1. **I/O Multiplexing (kqueue/kevent):** Substituição do modelo `fork()` por um loop de eventos único no FreeBSD, permitindo milhares de conexões simultâneas com custo de memória quase nulo.
 2. **Zero-Copy I/O (`sendfile`):** Uso da syscall `sendfile(2)` para transferir arquivos diretamente do Page Cache para o buffer do socket, saltando o espaço de usuário.
 3. **Thread Pooling:** Transição para threads persistentes, eliminando o custo de criação de processos e reduzindo o *Context Switching*.

---

# 📖 **Recursos e Documentação Oficial**
 Para garantir a integridade do desenvolvimento, utilizamos a documentação oficial do FreeBSD como **Single Source of Truth (SSoT)**.

## 🔎 **Consulta Online (Web)**
 *Melhor para busca indexada e navegação rápida entre capítulos.*

 * **[FreeBSD Books](https://docs.freebsd.org/en/books/)**: O hub central para livros e artigos técnicos.
 * **[FreeBSD Handbook](https://docs.freebsd.org/en/books/handbook/)**: O "guia definitivo" para instalação e administração.
 * **[FreeBSD Developers Handbook](https://docs.freebsd.org/en/books/developers-handbook/)**: Essencial para **programação de sockets**, chamadas de sistema e arquitetura do kernel.
 * **[FreeBSD FAQ](https://docs.freebsd.org/en/books/faq/)**: Respostas para as dúvidas mais comuns sobre o SO.
 * **[FreeBSD Manual Pages](https://man.freebsd.org/)**: Referência direta de comandos e funções da biblioteca C.

## 📥 **Download Offline (PDF)**
 *Ideal para ambientes isolados (air-gapped) ou leitura focada sem distrações.*

 | Recurso | Download PDF | Download Página |
 | --- | --- | --- |
 | **FreeBSD Handbook** | [📄 **Visualizar PDF](https://download.freebsd.org/doc/en/books/handbook/handbook_en.pdf)** | [📥 **Baixar Página](https://download.freebsd.org/doc/en/books/handbook/handbook_en.tar.gz)** |
 | **Developers Handbook** | [📄 **Visualizar PDF](https://download.freebsd.org/doc/en/books/developers-handbook/developers-handbook_en.pdf)** | [📥 **Baixar Página](https://download.freebsd.org/doc/en/books/developers-handbook/developers-handbook_en.tar.gz)** |
 | **FreeBSD FAQ** | [📄 **Visualizar PDF](https://download.freebsd.org/doc/en/books/faq/faq_en.pdf)** | [📥 **Baixar Página](https://download.freebsd.org/doc/en/books/faq/faq_en.tar.gz)** |

---

# 🛠️ **Acesso Rápido: Arquivos do Repositório**
 Além dos links oficiais, este repositório contém cópias locais da documentação e scripts de automação para facilitar o desenvolvimento no ambiente FreeBSD e Linux.

## 📚 **Livros (PDF Offline)**
 Estes arquivos estão localizados na pasta [`./FreeBSD/Books/`](./FreeBSD/Books/).

 | 📄 Documento | 🔗 Link Local | 📝 Descrição |
 | --- | --- | --- |
 | **FreeBSD Handbook** | [`FreeBSD Handbook.pdf`](./FreeBSD/Books/FreeBSD%20Handbook.pdf) | O guia definitivo de instalação, administração e uso geral do sistema. |
 | **FreeBSD Developers' Handbook** | [`FreeBSD Developers' Handbook.pdf`](./FreeBSD/Books/FreeBSD%20Developers%27%20Handbook.pdf) | Guia avançado focado em programação de Kernel, Sockets e IPC. |
 | **FreeBSD Architecture Handbook** | [`FreeBSD Architecture Handbook.pdf`](./FreeBSD/Books/FreeBSD%20Architecture%20Handbook.pdf) | Detalhes profundos sobre a estrutura e os subsistemas do kernel. |
 | **Design and Implementation of 4.4BSD** | [`The Design and Implementation of the 4.4BSD Operating System.pdf`](./FreeBSD/Books/The%20Design%20and%20Implementation%20of%20the%204.4BSD%20Operating%20System.pdf) | Livro clássico sobre a base de design do 4.4BSD (origem do FreeBSD). |
 | **FreeBSD Porter’s Handbook** | [`FreeBSD Porter’s Handbook.pdf`](./FreeBSD/Books/FreeBSD%20Porter%E2%80%99s%20Handbook.pdf) | Guia oficial para criar "ports" e empacotar softwares de terceiros. |
 | **FreeBSD FAQ** | [`Frequently Asked Questions for FreeBSD.pdf`](./FreeBSD/Books/Frequently%20Asked%20Questions%20for%20FreeBSD.pdf) | Perguntas frequentes e soluções rápidas de problemas comuns. |
 | **FreeBSD Documentation Primer** | [`FreeBSD Documentation Project Primer for New Contributors.pdf`](./FreeBSD/Books/FreeBSD%20Documentation%20Project%20Primer%20for%20New%20Contributors.pdf) | Manual para novos contribuidores do projeto de documentação oficial. |
 | **FreeBSD Project Model** | [`A project model for the FreeBSD Project.pdf`](./FreeBSD/Books/A%20project%20model%20for%20the%20FreeBSD%20Project.pdf) | Estudo sobre a estrutura organizacional e governança do projeto. |
 | **FreeBSD Accessibility Handbook** | [`FreeBSD Accessibility Handbook.pdf`](./FreeBSD/Books/FreeBSD%20Accessibility%20Handbook.pdf) | Guia de utilização e configuração de recursos de acessibilidade. |

## ⚙️ **Scripts de Configuração**
 Scripts utilitários localizados na pasta [`./FreeBSD/Scripts/`](./FreeBSD/Scripts/) para auxiliar na preparação do ambiente.

 * **[`install.sh`](./FreeBSD/Scripts/install.sh)**: Script para instalação das dependências (GCC, Gmake) e compilação do projeto.
 * **[`setup.sh`](./FreeBSD/Scripts/setup.sh)**: Script para configuração inicial do ambiente (variáveis e permissões).
 * **[`connect.sh`](./FreeBSD/Scripts/connect.sh)**: Script para testes de conectividade e sockets.
 * **[`download.sh`](./FreeBSD/Scripts/download.sh)**: Script para baixar recursos adicionais ou PDFs atualizados.
 * **[`uninstall.sh`](./FreeBSD/Scripts/uninstall.sh)**: Script para limpeza completa de dependências e binários.
