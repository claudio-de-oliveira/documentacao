# Ponteiros em RUST

Este guia detalhado explora a implementação interna dos principais ponteiros e invólucros (*wrappers*) de memória no Rust, detalhando como eles gerenciam a segurança e a performance.

---

## 1. Box<T> (O Dono Único)

O `Box` é a forma mais simples de alocação no **Heap**. Ele oferece propriedade exclusiva sobre o dado alocado.

* **Estrutura Interna:** Contém apenas um ponteiro para o endereço de memória no Heap.
* **Tamanho na Stack:** **1 word** (8 bytes em 64-bit).
* **Implementação:**
* **Deref:** Permite acessar o conteúdo como se fosse um valor comum.
* **Drop:** Quando o `Box` sai de escopo, ele chama o destruidor do dado e libera a memória do Heap imediatamente.


* **Custo:** Zero overhead. É o equivalente ao `std::unique_ptr` do C++.

---

## 2. Rc<T> (Propriedade Compartilhada)

O `Rc` (Reference Counted) permite que um dado tenha múltiplos donos no mesmo thread.

* **Estrutura Interna:** O ponteiro na Stack aponta para uma struct no Heap chamada `RcBox`.
* **Layout no Heap (`RcBox`):**
* `strong`: Contador de referências fortes (determina quando o dado é destruído).
* `weak`: Contador de referências fracas (determina quando a `RcBox` é desalocada).
* `value`: O dado real `T`.


* **Funcionamento:** * `clone()` apenas incrementa o contador `strong`.
* `drop()` decrementa o contador. Se `strong == 0`, o dado é destruído.


* **Restrição:** Não é thread-safe porque os incrementos não são atômicos.

---

## 3. RefCell<T> (Mutabilidade Interior Dinâmica)

O `RefCell` move as regras de empréstimo (*borrow checking*) do tempo de compilação para o **tempo de execução**.

* **Estrutura Interna:** * `value`: Um `UnsafeCell<T>` que contém o dado.
* `borrow`: Um `isize` que atua como contador de estados (**0**: livre, **>0**: leitores, **-1**: um escritor).


* **Segurança:** Se você chamar `.borrow_mut()` enquanto houver um `.borrow()` ativo, o programa sofrerá um **panic**.
* **Uso Comum:** Frequentemente usado dentro de um `Rc` (`Rc<RefCell<T>>`) para permitir que múltiplos donos modifiquem o mesmo dado.

---

## 4. Cell<T> (Mutabilidade por Cópia)

O `Cell` permite mutabilidade interior sem o custo de um contador de referências (ao contrário do `RefCell`), mas com restrições.

* **Estrutura Interna:** Apenas um invólucro em volta de `UnsafeCell<T>`.
* **Funcionamento:** Ele não fornece referências (`&T`). Em vez disso, ele move valores para dentro e para fora usando os métodos `.get()` e `.set()`.
* **Requisito:** No caso de `.get()`, o tipo `T` deve implementar o trait `Copy`.
* **Vantagem:** Possui **zero overhead** em tempo de execução. O compilador simplesmente permite a alteração do valor através de uma referência imutável.

---

## 5. Unique<T> (O Bloco de Construção)

O `Unique` é um tipo interno da biblioteca padrão (`std::ptr::Unique`), não sendo usado diretamente no código estável do dia a dia, mas é a base de `Box<T>` e `Vec<T>`.

* **Função:** É um invólucro sobre um ponteiro bruto `*mut T`.
* **Propriedades Semânticas para o Compilador:**
1. **Propriedade:** Indica que a struct "possui" o dado (ajuda o *Drop Checker*).
2. **Não-Nulo:** Garante que o ponteiro nunca é nulo (permite otimização de `Option<T>`).
3. **Covariância:** Permite que o sistema de tipos lide corretamente com subtipos e tempos de vida (lifetimes).



---

## Tabela Comparativa de Implementação

| Ponteiro | Local do Dado | Mutabilidade | Thread-Safe | Overhead |
| --- | --- | --- | --- | --- |
| **Box<T>** | Heap | Estática (Exclusiva) | Sim (se T for Send) | Zero |
| **Rc<T>** | Heap | Imutável | Não | Contagem de Ref. |
| **RefCell<T>** | Local (Stack/Heap) | Dinâmica | Não | Contador `isize` |
| **Cell<T>** | Local (Stack/Heap) | Por Cópia | Não | Zero |
| **Unique<T>** | Heap | N/A (Interno) | N/A | Zero |

---

### Resumo Visual do Layout de Memória

* **Box<T>**: `[ Ponteiro ]`  `[ Dado T ]`
* **Rc<T>**: `[ Ponteiro ]`  `[ Strong | Weak | Dado T ]`
* **RefCell<T>**: `[ Contador | Dado T ]` (O dado está "dentro" dele)
* **Unique<T>**: `[ Ponteiro Bruto ]` (Com metadados de tipagem para o compilador)

Seria útil para você ver um exemplo de código que combine **Rc** e **RefCell** para criar uma estrutura de dados compartilhada e mutável?
