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

Aqui está o documento atualizado. Adicionei exemplos práticos e concisos para cada um dos conceitos, focando em **quando** e **como** utilizá-los no seu código.

---

# Exemplos

---

## 1. Box<T> (O Dono Único)

O `Box` aloca memória no **Heap**. É o ponteiro inteligente mais simples, usado para tipos com tamanho desconhecido em compilação ou para evitar cópias de grandes estruturas na Stack.

### Exemplo Simples:

```rust
fn main() {
    // O valor 5 é colocado no Heap
    let b = Box::new(5);
    
    // Podemos usar b como se fosse um i32 comum graças ao Deref
    println!("Valor no heap: {}", *b + 10); 
} // A memória no heap é liberada aqui automaticamente (Drop)

```

---

## 2. Rc<T> (Propriedade Compartilhada)

O `Rc` (Reference Counted) permite que um dado tenha vários donos. Ele é ideal para grafos ou estruturas onde a vida útil do dado não é linear.

### Exemplo Simples:

```rust
use std::rc::Rc;

fn main() {
    let original = Rc::new(String::from("Dados Compartilhados"));
    
    // Incrementa o contador 'strong' para 2
    let clone_1 = Rc::clone(&original);
    
    {
        // Incrementa o contador para 3
        let clone_2 = Rc::clone(&original);
        println!("Contagem: {}", Rc::strong_count(&original));
    } // clone_2 sai de escopo, contagem volta para 2

    println!("Contagem final: {}", Rc::strong_count(&original));
}

```

---

## 3. RefCell<T> (Mutabilidade Interior Dinâmica)

Permite modificar dados mesmo através de referências imutáveis, verificando as regras de empréstimo em tempo de execução.

### Exemplo Simples:

```rust
use std::cell::RefCell;

fn main() {
    let dado = RefCell::new(10);

    // Podemos criar uma referência mutável mesmo 'dado' não sendo 'let mut'
    let mut referencia_mut = dado.borrow_mut();
    *referencia_mut += 5;
    
    println!("Novo valor: {:?}", referencia_mut);
    
    // IMPORTANTE: Se tentássemos um .borrow() aqui sem soltar o mut, o programa daria PANIC.
} 

```

---

## 4. Cell<T> (Mutabilidade por Cópia)

Semelhante ao `RefCell`, mas sem contadores de runtime. Ele funciona substituindo o valor inteiro, por isso exige que o tipo seja `Copy`.

### Exemplo Simples:

```rust
use std::cell::Cell;

fn main() {
    let c = Cell::new(100);

    let valor_antigo = c.get();
    c.set(200); // Altera o valor interno

    println!("Antigo: {}, Novo: {}", valor_antigo, c.get());
}

```

---

## 5. Unique<T> (O Bloco de Construção Interno)

O `Unique` é uma abstração interna usada para construir coleções. Ele garante ao compilador que o ponteiro é o dono da memória e nunca é nulo.

### Exemplo Conceitual (Uso em Structs do Sistema):

```rust
// Exemplo de como o Rust define um Box internamente (simplificado)
// Requer #![feature(ptr_internals)] em versões nightly
struct MyBox<T> {
    ptr: std::ptr::Unique<T>,
}

// O Unique garante que MyBox<T> é o dono exclusivo de T,
// permitindo que o Drop Checker saiba que deve limpar T.

```

---

## Tabela Comparativa de Implementação

| Ponteiro | Local do Dado | Mutabilidade | Risco de Pânico | Overhead |
| --- | --- | --- | --- | --- |
| **Box<T>** | Heap | Estática | Não | Zero |
| **Rc<T>** | Heap | Imutável | Não | Contagem Ref. |
| **RefCell<T>** | Local | Dinâmica | **Sim** | Verificação Runtime |
| **Cell<T>** | Local | Por Cópia | Não | Zero |

---

## Padrão Comum: O "Super Ponteiro" `Rc<RefCell<T>>`

Muitas vezes você precisará de **múltiplos donos** (Rc) e que todos eles possam **modificar** o dado (RefCell).

```rust
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let compartilhado = Rc::new(RefCell::new(5));

    let dono_1 = Rc::clone(&compartilhado);
    let dono_2 = Rc::clone(&compartilhado);

    *dono_1.borrow_mut() += 10;
    *dono_2.borrow_mut() += 20;

    println!("Valor final: {:?}", compartilhado.borrow()); // Resultado: 35
}

```

---

