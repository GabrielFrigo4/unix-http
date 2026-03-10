# 💻 **Implementação: Servidor HTTP/1.1 RESTful em C (POSIX / FreeBSD & Linux)**

## 🎯 **Arquitetura e Objetivo Técnico (V1)**
 Implementação de um servidor web e API REST robusta em **C puro (C23)**, operando diretamente sobre a API padrão de **Berkeley Sockets**. Este projeto foi desenhado com foco estrito em **portabilidade UNIX (POSIX)**, garantindo execução nativa tanto em ambientes **FreeBSD** quanto **Linux**.

 O objetivo primário é a exploração da pilha TCP/IP e a superação das armadilhas clássicas de I/O, adotando o padrão de *Clean Architecture* e *Zero-Copy Parsing*.

### ⚙️ **Modelo de Concorrência: Fork-per-Request & Self-Pipe Trick**
 Para garantir que múltiplas requisições sejam atendidas simultaneamente sem bloqueio de I/O, o servidor adota a system call `fork()`, aliada a um sofisticado controle de estado:

 * **Isolamento de Falhas (Workers):** Cada conexão é delegada a um processo filho isolado, protegendo o daemon principal.
 * **O "Self-Pipe Trick" e `poll()`:** Para superar a limitação assíncrona dos sinais POSIX, o servidor utiliza um `pipe` local não-bloqueante. Sinais do Kernel são injetados no cano e monitorados de forma síncrona pelo `poll()` junto com os sockets de rede.
 * **Resiliência a `EINTR`:** O loop de eventos captura e trata *Interrupted System Calls*, garantindo estabilidade durante a limpeza de processos zumbis.

### ⏱️ **Ciclo de Vida da Conexão (V1)**
 ```mermaid
 sequenceDiagram
     participant C as HTTP Client
     participant M as Master (poll)
     participant P as Self-Pipe
     participant OS as Kernel UNIX
     participant W as Child Worker

     M->>P: setup pipe() & sigaction()
     loop Main Event Loop
         M->>OS: poll(server_fd, pipe_read_fd)

         alt Novo Cliente HTTP
             C->>OS: TCP 3-Way Handshake
             OS-->>M: POLLIN no server_fd
             M->>W: fork()
             W->>C: Zero-Copy HTTP Parse & Resposta
             W->>OS: _exit(0)

         else Sinal Assíncrono (SIGCHLD)
             OS->>M: Kernel emite Sinal
             M->>P: write(pipe_write_fd, signo)
             OS-->>M: POLLIN no pipe_read_fd
             M->>OS: waitpid(WNOHANG) -> Zombie Reaped!
         end
     end
 ```

## ⚡ **Definição dos Métodos RESTful Suportados**
 | Método | Comportamento no Servidor | Finalidade Técnica |
 | --- | --- | --- |
 | **GET** | Leitura via I/O padrão (`fread`) | Recursos estáticos ou serialização JSON. |
 | **POST** | Criação via `fopen(..., "w")` | Criação integral de novos recursos. |
 | **PUT** | Escrita Idempotente | Substituição completa de um recurso existente. |
 | **PATCH** | Modificação via `fopen(..., "a")` | Atualização parcial via *append*. |
 | **DELETE** | Remoção via syscall `remove()` | Exclusão definitiva no sistema de arquivos. |

## 🔨 **Compilação e Deploy**
 O projeto utiliza um `.editorconfig` para padronização global e é gerenciado via `Makefile`. Requer **GCC** com suporte a **C23**.

 ```sh
 make clear
 make build
 make run
 ```

---

## 🚀 **Roadmap e Evolução Arquitetural (Versão 2)**
 A Versão 2 focará em **Alta Performance Absoluta** e escalabilidade horizontal, migrando do modelo de processos para um modelo de eventos assíncronos.

### **Proposta Técnica V2: Event-Driven Architecture**
 ```mermaid
 graph TD
     subgraph "Kernel Space"
         K_NET[Network Interface]
         K_SIG[Signal Events]
         K_FS[Filesystem Cache]
     end

     subgraph "User Space (V2 Engine)"
         EP[Event Multiplexer: kqueue / epoll]
         TQ[Task Queue]
        
         subgraph "Thread Pool (Workers)"
             T1[Thread 1]
             T2[Thread 2]
             T3[Thread N]
         end
     end

     K_NET -- "O(1) Readiness" --> EP
     K_SIG -- "Signal Event" --> EP
     EP -- "Task Dispatch" --> TQ
     TQ -- "Pop" --> T1
     TQ -- "Pop" --> T2

     T1 -- "syscall: sendfile()" --> K_FS
     K_FS -- "Direct DMA Transfer" --> K_NET

     style EP fill:#3b82f6,stroke:#fff,color:#fff
     style TQ fill:#1e293b,stroke:#fff,color:#fff
     style K_NET fill:#10b981,stroke:#fff,color:#fff 
 ```

 1. **I/O Assíncrono Nativo:** Substituição do `poll()` por **`kqueue`** (FreeBSD) ou **`epoll`** (Linux), permitindo gerenciar milhares de conexões com complexidade $O(1)$.
 2. **Otimização de Kernel (Zero-Copy):** Implementação da system call **`sendfile()`**, eliminando o *overhead* de copiar bytes do espaço do Kernel para o espaço do usuário antes de enviá-los à rede.
 3. **Thread Pooling:** Transição para um modelo de Threads persistentes, eliminando o custo de criação de processos e reduzindo drasticamente o *Context Switching*.
