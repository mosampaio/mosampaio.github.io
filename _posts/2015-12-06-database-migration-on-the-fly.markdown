---
layout: post
title:  "Database Migration On The Fly"
date:   2015-12-06 18:50:52 -0800
categories: database migration fly demand mongodb java
---
# Database Migration On The Fly

Nos últimos anos, os times agéis tem adotado a técnica chamada [Evolutionary Database Design](http://martinfowler.com/articles/evodb.html), que basicamente consiste em evoluir o schema do banco de dados gradualmente durante a fase de desenvolvimento e automatizar as mudanças através de scripts que geralmente rodam durante o processo de deploy da aplicação.

Já existe um grande número de bibliotecas que facilita esta automatização durante o deploy e que atende muito bem a maioria dos projetos. 

Porém, um dos problemas de rodar os scripts durante o deployment é o downtime. A aplicação precisa ficar indisponível até que todas as migrações sejam executadas para posteriormente o deploy ser efetuado. 

Dependendo da quantidade de dados e migrações, o downtime pode ser grande o suficiente para ser ser considerado inaceitável para algumas situações.

O objetivo deste artigo é oferecer uma alternativa que não precise necessariamente que a aplicação fique fora do ar.

Para exemplificar esta técnica, foi-se utilizada a seguinte tech stack: Java 8, Spring Data MongoDB e o MongoDB.

O fato de usar um banco noSQL como o MongoDB é essencial a execução desta técnica, pois trata-se de um banco de dados schemaless, o que nos permite ter na mesma coleção documentos com estruturas diferentes.

Digamos que existe uma coleção chamada people com a seguinte estrutura:

```javascript
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Marcos Sampaio",
  "telephone": "14159363485"
}
```

```java
class Person {
  UUID id;
  String name;
  String telephone;
}
```
Inicialmente o cliente achou que cada pessoa deveria ter apenas um telefone. Depois decidiu-se que ao invés de armazenar apenas um telefone, pode-se armazenar múltiplos. 

Para atender a mudança, o time decidiu que deveria escrever uma migração para mudar o "schema lógico" e o estrutura ficará assim:

```javascript
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Marcos Sampaio",
  "telephones": ["14159363485"]
}
```
```java
class Person {
  UUID id;
  String name;
  List<String> telephones;
}
```
Como as coisas serão feitas "on the fly", cada documento terá que guardar uma versão de si mesmo, para sabermos se a migração deverá ser aplicada. Então será adicionado o campo "migrationVersion" em cada um deles.

```javascript
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Marcos Sampaio",
  "telephones": ["14159363485"],
  "migrationVersion": 1
}
```

A classe de domínio (Person) deve entender apenas a "ultima versão" do schema lógico de dados. Por tanto, ela é agnostica em relação ao campo migrationVersion e ao fato de existir múltiplas versões no banco.

Neste padrão, deve-se introduzir uma camada entre o repositório e o banco de dados. O objetivo desta camada é garantir a conversão do documento para o objeto de domínio, não importando em qual “versão” este documento esteja.

```java
public class PersonMongodbListener extends AbstractMongoEventListener<Person> {

    private MigrationOnTheFly migrationOnTheFly;
    private int currentMigrationVersion;

    public PersonMongodbListener(MigrationOnTheFly migrationOnTheFly, int currentMigrationVersion) {
        this.migrationOnTheFly = migrationOnTheFly;
        this.currentMigrationVersion = currentMigrationVersion;
    }

    @Override
    public void onBeforeSave(BeforeSaveEvent<Person> event) {
        this.migrationOnTheFly.migrateVersion(event.getDBObject());
    }

    @Override
    public void onAfterLoad(AfterLoadEvent<Person> event) {
        this.migrationOnTheFly.migrate(event.getDBObject());
    }
}
```

O evento onAfterLoad é um evento que ocorre entre o processo de leitura do banco e a conversão do dbobject em um objeto de domínio. É nessa fase que será executada todas as migrações que ainda não foram aplicadas para aquele documento/dbobject pois o objeto de domínio só entende a estrutura final. É importante frisar que essas migrações não serão salvas no banco nesse momento.

No método onBeforeSave, atribui-se a versão final naquele documento. Como o objeto de domínio só entende a estrutura final, quando vamos salvar, faz sentido que sempre salve com migrationVersion={latest}.

Com isso, garantimos que toda vez que o objeto for lido, ele vai ser convertido para algo que o domínio entende e toda vez que for salvo, já estará no schema lógico final. Deste modo, a migração é feita de maneira suave, sem impactar o downtime do sistema e sem poluir as classes de domínio.

Abaixo segue a implementação da classe MigrationOnTheFly.
```java
public class MigrationOnTheFly {

    public static final String MIGRATION_VERSION_FIELD_NAME = "migrationVersion";

    private SortedMap<Integer, Migration> migrations;
    private int currentMigrationVersion;

    public MigrationOnTheFly(SortedMap<Integer, Migration> migrations, int currentMigrationVersion) {
        this.migrations = migrations;
        this.currentMigrationVersion = currentMigrationVersion;
    }

    public void migrate(final DBObject dbObject) {
        final Integer migrationVersion = (Integer) Optional
                .ofNullable(dbObject.get(MIGRATION_VERSION_FIELD_NAME))
                .orElse(0);

        migrations.entrySet()
                .stream()
                .filter(input -> input.getKey() > migrationVersion)
                .forEachOrdered(migrationEntry -> {
                    migrationEntry.getValue().migrate(dbObject);

                });
    }

    public void migrateVersion(DBObject dbObject) {
        dbObject.put(MIGRATION_VERSION_FIELD_NAME, currentMigrationVersion);
    }
}
```


Porém nem tudo são flores. 

Existe alguns cenários que também deve-se estar atento. Caso a aplicação faça alguma consulta que utiliza o campo migrado, deve-se alterar estas consultas para contemplar múltiplas versões. 

##### Exemplo:
Antes da migração 1
```javascript
db.people.find({
  'telephone': '14159363485'
})
```
Depois da migração 1
```javascript
db.people.find({
  $or:[
    {'telephone': '14159363485', 'migrationVersion': {$exists: false}},
    {'telephones': {$in: ['14159363485']}, 'migrationVersion': {$gte: 1}}
  ]
})
```
Isto pode aumentar a complexidade a medida em aumenta o número de migrações e caso tenha mudanças em campos usados nas consultas.

Uma solução é, além de ter essa migração on the fly, ter um job seja iniciado após o deploy e faça as migrações em paralelo para garantir que todos os dados eventualmente estarão migrados.

Assim, pode-se remover as migrações antigas posteriormente e consequentemente a complexidade das consultas.

Uma coisa que deve-se estar atento em relação a este job é que ele pode acabar atualizando um dado esteja sendo salvo pelo usuário. Por isso, recomendo que o uso de optimistic locking nas entidades migradas.

Outra coisa a se considerar é, caso queira manter as duas versões da aplicação (antiga e a nova) rodando ao mesmo tempo durante um curto período, é importante que a migração não remova campos existentes para manter a retrocompatibilidade.

Neste [link](https://github.com/mosampaio/migration-on-the-fly/) pode-se encontrar uma implementação simples deste padrão.
