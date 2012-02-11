---
title: Base de données sur les workers de Travis
kind: content
---

Tout les workers de Travis CI ont des logiciels pré-installés souvent utilisés par les développeurs de la communauté open source. Quelques uns des services disponibles sont:

* Bases de données – MySQL, PostgreSQL, SQLite3, MongoDB
* Sauvegarde clé-valeur – Redis, Riak, memcached
* Systèmes de messagerie – RabbitMQ
* node.js, ImageMagick

Vous pourrez trouver la liste complète des logiciels installés, comme les versions de Ruby, dans le [fichier de configuration du worker][config]. Ce fichier spécifie quelles [recettes du cookbook][cookbook] sont utilisées lors de la construction de l'instance du worker.

Certains d'entre eux e.g. node.js et memcached, sont accessibles dans le PATH où s'exécute sur un port par défaut, par conséquent, aucunes informations supplémentaires sont requises. Les autres, autrement dit les base de données, peuvent avoir besoin d'une authentification.

Nous allons voir ici comment configurer votre projet pour utiliser une base de données dans vos tests. Ceci suppose que vous avez déjà visité la documentation [configurer une build][build configuration].

### SQLite3

C'est probablement la solution la plus facile et la plus rapide pour vos besoins en base de données relationnelle. Si vous ne voulez pas tester le comportement de votre code avec d'autres base de données, SQLite en mémoire peut être la meilleur option et la plus rapide à mettre en place.

Pour les projets Ruby, assurez-vous d'avoir une liason sqlite3 de Ruby dans votre bundle: 

    # Gemfile
    gem 'sqlite3'

Si votre projet est une application Rails, vous n'avez qu'a définir ceci:

    # config/database.yml in Rails
    test:
      adapter: sqlite3
      database: ":memory:"
      timeout: 500

Sinon, dans le cas où votre projet est une librairie où un plugin, vous devez configurer la connexion à la base de données vous même dans vos tests. Par exemple, voici comment faire avec ActiveRecord:

    ActiveRecord::Base.establish_connection :adapter => 'sqlite3',
                                            :database => ':memory:'

### MySQL

MySQL ne requiert aucune authentification sur Travis. Définissez un utilisateur et un mot de passe vide:

    mysql:
      adapter: mysql2
      database: myapp_test
      username: 
      encoding: utf8

Vous devez ensuite créer la base de données `myapp_test`. Exécuter ceci avec votre script de build:

    # .travis.yml
    before_script:
      - "mysql -e 'create database myapp_test;'"

### PostgreSQL

PostgreSQL requiert une authentification avec l'utilisateur "postgres" sans mot de passe:

    postgres:
      adapter: postgresql
      database: myapp_test
      username: postgres

Vous devez ensuite créer la base de données avec votre script de build:

    # .travis.yml
    before_script:
      - "psql -c 'create database myapp_test;' -U postgres"

### MongoDB

MongoDB ne requiert aucune authentification où de création d'une base de données:

    require 'mongo'
    Mongo::Connection.new('localhost').db('myapp')

Dans de cas où vous avez besoin de créer des utilisateurs pour votre base de données, vous pouvez faire quelque chose comme ça:

    # .travis.yml
    before_script:
      - mongo myapp --eval 'db.addUser("travis", "test");'

    # then, in ruby:
    uri = "mongodb://travis:test@localhost:27017/myapp"
    Mongo::Connection.from_uri(uri)

### Les bases de données multiples

Si les tests de votre projet ont besoin de s'exécuter plusieurs fois en utilisant des base de données différentes, vous pouvez le configurer sur Travis de plusieurs façons différentes. Une première approche consiste à mettre chaque configuration des bases de données dans un seul fichier yaml:  

    # test/database.yml
    sqlite:
      adapter: sqlite3
      database: ":memory:"
      timeout: 500
    mysql:
      adapter: mysql2
      database: myapp_test
      username: 
      encoding: utf8
    postgres:
      adapter: postgresql
      database: myapp_test
      username: postgres

Ensuite, dans vos suites de test, récupérer toutes ces informations dans un hash contenant alors les configurations:

    configs = YAML.load_file('test/database.yml')
    ActiveRecord::Base.configurations = configs

    db_name = ENV['DB'] || 'sqlite'
    ActiveRecord::Base.establish_connection(db_name)
    ActiveRecord::Base.default_timezone = :utc

Ici, nous avons utilisé la variable d'environnement "DB" pour définir le nom de la configuration de base de données que vous voulez utiliser. Vous pourrez exécuter ceci en local:

    $ DB=postgres bundle exec rake

Pour tester avec vos 3 base de données de façon permanente sur Travis CI, vous pouvez utiliser l'option "env":

    # .travis.yml
    env:
      - DB=sqlite
      - DB=mysql
      - DB=postgres

Lorsque vous faites ceci, veuillez lire tout ce qui concerne la matrice de builds détaillé dans la section [configurer une build][build configuration].

Pour finir, assurez-vous d'avoir créer les bases de données mysql et postgres dans votre script de build conformément aux notes de ce document écrites plus tôt.


[cookbook]: https://github.com/travis-ci/travis-cookbooks
[config]: https://github.com/travis-ci/travis-worker/blob/master/config/worker.production.yml
[build configuration]: /docs/user/build-configuration/
