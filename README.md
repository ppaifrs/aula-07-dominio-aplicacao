# Domínio

Camadas
Layered Architecture - Camada Lógica
Tiered Architecture - Camada "Física"

Tier
+-------------------+
Usuário final, outros sistemas
+-------------------+
| Apresentação/UI   | uma máquina
+-------------------+
| Negócio           | outra máquina
+-------------------+
| Persistência      | ...
+-------------------+
Banco de dados, outros meios
+-------------------+

Microsserviço: todas as camadas repetidas em várias "máquinas"
+-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ 
+-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ 
+-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ 
+-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ +-+ 

Escala de hardware, escala vertical e horizontal
Vertical: máquinas melhores (mais RAM, mais CPU, etc)
Horizontal: mais máquinas

## Negócio

Business Rules (regras de negócio)
Domain Rules (regras de domínio)
Business Logic (lógica de negócio)

Como organizar o negócio/domínio?

Como não organizar o negócio/domínio?

Ex.: considere um sistema acadêmico,
considere a ideia de turma que tem 12 vagas,
e começam as matrículas, quando o 13º estudante
tentar se matricular. Em algum lugar, terá
```java
if (!ehRepetente && matriculados + 1 > vagas) {
    throw new TurmaCheiaException();
}
```

**Não deve estar no front** (JS que vai para o navegador, código da interface do usuário Swing, WPF, Forms, Tk, etc).

**Não deve estar na persistência**, ou seja, nos repositórios, nos DAOs, ou seja o nome, quer dizer, onde está o SQL, não deve ter regras. Por exemplo:

```java
class TurmaRepository {

    void save(Turma turma) {
        if (turma.alunos.size() > turma.vagas) ...
        // código SQL
    }

}
```

Persistência agnóstica, deve ser apenas submeter as informações ao banco de dados.

**Não deve estar no banco (ou deve?!)**, pois daí é um banco de regras e não de dados. Bancos comuns que têm regras: Oracle, IBM DB2, MSSQL Server, etc, banco "enterprise". No Banco:

```
// 2023-11-23
  // alterado isso
// 2024-02-04
  // fulano de tal, mudou taltaltal
// 2024-07-29 Gabriel
 // taltal
procuredure matricular() {
    // fetch rows (select)
    // count
    // if matriculados > vagas
    //   raise error
}
```

```java
class TurmaRepository {
    void matricular(int aluno, int turma) {
        ps = preparedStatement("CALL matricular(?, ?)");

        try {
            ps.execute();
        } catch (SQLException e) {
            e.getMessage().startWith("Erro:");
        }
    }
}
```

Considere a o padrão de arquitetura, MVC, MVVM, MVP, etc, onde não deve estar a lógica de negócio?
- View (lógica de apresentação), ex.: JSF, .NET WebForms, classe que representa a interface do usuário
- Controller, ele deve "pegar" os dados da requisição e repassar ao domínio (VM - viewModel - Presenter).

Onde ficam as regras?

Existem dois Padrões de Arquitetura dominantes:

- Transaction Script (roteiro da transação)
- Domain-Driven (guiado pelo domínio)

View
Controller
Model

```java
class TurmaService { // TurmaBusiness, TurmaBiz, ...
// Transaction Script
    Resultado matricular(int idAluno, int idTurma) {
        Aluno aluno = repoAluno.findById(idAluno);
        Turma turma = repoTurma.findById(idTurma);
        List<Aluno> alunos = repoTurma.findAlunos(idTurma);
        if (alunos.size() + 1 > turma.getVagas()) {
            throw TurmaException();
        }
        Matricula matricula = new Matricula(idAluno, idTurma);
        repoTurma.save(matricula);
        // ...
        return resultado;
    }
}
```


```java
// DTO
class Conta { // modelo (objeto) anêmico (não tem comportamento)
    int numero;
    BigDecimal saldo;
    // getter/setters/toString
}

class ContaService { // Transaction Script (induz ao modelo anêmico)
    void transferir(int numOrigem, int numDestino, BigDecimal valor) {
        // busca dado
        Conta origem = contaRepo.findByNumero(numOrigem);
        Conta destino = contaRepo.findByNumero(numDestino);
        // transforma dado
        if (origem.getSaldo().lessThan(valor)) {
            throw SaldoInsuficienteException();
        }
        origem.setSaldo(origem.getSaldo() - valor);
        destino.setSaldo(destino.getSaldo() + valor);
        // persiste dado
        contaRepo.saveAll(origem, destino);
    }  
}
```

```java
class Conta {
    void sacar(BigDecimal valor) {
        if (this.getSaldo().lessThan(valor)) {
            throw SaldoInsuficienteException();
        }
    }
}
class ContaService { // Transaction Script, mas com um Modelo/Domínio
    void transferir(int numOrigem, int numDestino, BigDecimal valor) {
        Conta origem = contaRepo.findByNumero(numOrigem);
        Conta destino = contaRepo.findByNumero(numDestino);
        origem.sacar(valor); // existe o método sacar e depositar no domínio
        destino.depositar(valor);
        contaRepo.saveAll(origem, destino);
    }  
}
```


```java
// Application Service (serviço que apóia a aplicação, mas não o domínio)
class TransferenciaFundoService {
    void transferir(int numOrigem, int numDestino, BigDecimal valor) {
        Conta origem = contaRepo.findByNumero(numOrigem);
        Conta destino = contaRepo.findByNumero(numDestino);
        origem.transferir(destino);   
        contaRepo.saveAll(origem, destino);
    }
}
// Domain Model
class Conta { // A Conta é um domínio autocontido,
// toda a lógica do estado resultante deve estar no domínio
    void transferir(Conta destino, BigDecimal valor) {
        this.sacar(valor);
        destino.depositar(valor);
    }

    void depositar(BigDecimal valor) {
        this.saldo += valor;
    }
}
```
// Escala -> Um dia isto será alterado, incrementado, mudado.

MatriculaController -> MatriculaService 
                            -> MatriculaDomainService -> Matricula  
                            -> MatriculaRepositorio