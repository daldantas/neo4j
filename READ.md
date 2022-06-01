# Demanda & Desenvolvimento
Demanda: [Jira](https://jira.tce.sc.gov.br/browse/ADV)
# Pré-requisitos
* GIT
* Docker
# Instalação
> Na URL, altere para o seu usuário
```sh
git clone http://gabriel.dantas@bitbucket.tce.sc.gov.br/scm/por/vinculos-neo4j.git
cd vinculos-neo4j
docker compose up -d
```
Veja se o sistema está rodando
```sql
docker ps
```
Inicie, reinicie ou pare o sistema
```sql
docker start neo4j
docker restart neo4j
docker stop neo4j
```
Acesse `http://localhost:7474/browser/`
* Usuário e senha: **neo4j**  
# O que é Neo4j
É um banco de dados gráfico que usa a Cypher para armazenar estruturas complexas de dados.
# O que é Cypher
É a linguagem de consulta do Neo4j, reutiliza a sintaxe do SQL e ascii-art para representar gráficos.  
>[Getting Started with Neo4j](https://neo4j.com/developer/get-started/)  
[Documentation](https://neo4j.com/docs/cypher-manual/current/)  
## Conceitos básicos & Sintaxe
### Nodes ou Nós
São registros de entidades representados com `()`.
* `(n)` É um node referido pela variável `n`, _camelCase*_.
* `(p:Pessoa)` Você pode adicionar um label ao seu node,  _PascalCase**_.
* `(p:Pessoa:Terceirizado)` Um node pode ter muitos labels.
* `(p:Pessoa {nome: 'Gabriel', idade: 29})` Um node pode ter propriedades, _camelCase*_.

As propriedades podem ser dos tipos:  
* Numeric
* Boolean
* String
### Relationships ou Arestas
São relacionamentos representados com `[]`, conexão entre diferentes Nós.
* `[:COLEGAS]` É um relacionamento com o label COLEGAS, _SCREEMING_SNAKE_CASE***_.
* `[c:COLEGAS]`  O mesmo relacionamento, referido pela variável c, camelCase*.
* `[c:COLEGAS {desde: 2020}]` O mesmo relacionamento, com propriedades, camelCase*.
### Comandos básicos
Criando Nós:
```sql
CREATE  (:Person:Coordinator {name: 'Rafael'}),
        (:Person:Manager     {name: 'Leonardo'}),
        (:Person:Outsourced  {name: 'Luciano'}),
        (:Person:Outsourced  {name: 'Jeferson'}),
        (:Person:Outsourced  {name: 'Gabriel'}),
        (:Person:Outsourced  {name: 'Bruno'}),
        (:Person:Outsourced  {name: 'Sabrina'}),
        (:Person:Outsourced  {name: 'Davidson'}),
        (:Person:Outsourced  {name: 'Michele'})
```
A criação do index otimiza a consulta melhorando o desempenho da aplicação
```sql
CREATE INDEX ON :Person(id)
```
Criando Relacionamentos:
```sql
MATCH (c:Coordinator)
MATCH (m:Manager)
CREATE (c)-[:COORDINATES]->(m)
```
```sql
MATCH (c:Coordinator)
MATCH (o:Outsourced)
CREATE (c)-[:COORDINATES]->(o)
```
```sql
MATCH (m:Manager)
MATCH (o:Outsourced)
CREATE (m)-[:MANAGES]->(o)
```
Consultando a entidade e seus relacionamentos
```sql
MATCH (p:Person) RETURN p
```
### Comandos Intermediários
#### Criando entidades a partir de documentos **CSV**.  
Lista de Medicamentos
```sql
LOAD CSV WITH HEADERS FROM  
'file:///drugs.csv' AS drug  
CREATE (:Drug {
    code: drug.code,
    name: drug.name
})
```
```sql
CREATE INDEX ON :Drug(id)
```
```sql
MATCH (n:Drug) RETURN n LIMIT 50
```
Lista de Patologias
```sql
LOAD CSV WITH HEADERS FROM  
'file:///pathologies.csv' AS pathology  
CREATE (:Pathology {
    code: pathology.code,
    name: pathology.name
})
```
```sql
CREATE INDEX ON :Pathology(id)
```
```sql
MATCH (n:Pathology) RETURN n LIMIT 50
```
Aqui usamos um documento **CSV** e entidades previamente geradas para criar um Relacionamento que mostra medicamentos que foram usados para tratar doenças.
Onde a Aresta é o paciente.
```sql
LOAD CSV WITH HEADERS FROM 'file:///drug-use.csv' AS line
MATCH (d:Drug {code: line.codedrug})
MATCH (p:Pathology {code: line.codepathology})
CREATE (d)-[:TREATS {person: line.idperson}]->(p)
```
Consulta o Relacionamento
```sql
MATCH (d:Drug)-[:TREATS]->(p:Pathology)
RETURN d, p
LIMIT 50
```
Deleta o Relacionamento
```sql
MATCH ()-[t:TREATS]->()
DELETE t
```
### Comandos Avançados
No caso acima, não contamos o número de vezes que a relação acontece. Então usaremos uma sentença `MERGE` para o Relacionamento baseado na incidência em que um medicamento tratou uma patologia.
```sql
LOAD CSV WITH HEADERS FROM 'file:////drug-use.csv' AS line
MATCH (d:Drug {code: line.codedrug})
MATCH (p:Pathology {code: line.codepathology})
MERGE (d)-[t:TREATS]->(p)
ON CREATE SET t.weight=1
ON MATCH SET t.weight=t.weight+1
```
Medicamentos que trataram patologias, com mais de 50 ocorrências
```sql
MATCH (d)-[t:TREATS]->(p)
WHERE t.weight > 50
RETURN d,p
```
Todos Medicamentos que trataram dor
```sql
MATCH r=(d:Drug)-[t:TREATS]->(p:Pathology)
WHERE p.name = 'Pain'
RETURN r
```
Medicamentos que trataram dor, com mais de 10 ocorrências
```sql
MATCH r=(d:Drug)-[t:TREATS]->(p:Pathology)
WHERE t.weight > 10 AND p.name = 'Pain'
RETURN r
```
#### **Criando uma Projeção**
A sentença anterior produziu um grafo bipartido, vamos aplicar uma projeção para ligar drogas que tratam a mesma patologia. A patologia é transformada em aresta.
```sql
MATCH (d1:Drug)-[a]->(p:Pathology)<-[b]-(d2:Drug)
WHERE a.weight > 20 AND b.weight > 20 AND d1.name <> d2.name AND p.name <> 'Product used for unknown indication'
MERGE (d1)<-[r:RELATES]->(d2)
ON CREATE SET r.weight=1, r.pathology=p.name
ON MATCH SET r.weight=r.weight+1
```
Consulta o Relacionamento
```sql
MATCH (d1:Drug)<-[r:RELATES]->(d2:Drug)
RETURN d1, d2
```
```sql
MATCH (d1:Drug)<-[r:RELATES]->(d2:Drug)
WHERE r.weight > 3
RETURN d1, d2
```
<small>*_minúsculas iniciadas com maiúsculas, exceto primeira_  
**_minúsculas iniciadas com maiúsculas_  
***_maiúsculas separadas com underline_</small>