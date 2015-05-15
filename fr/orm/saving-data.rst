Sauvegarder les Données
#######################

.. php:namespace:: Cake\ORM

.. php:class:: Table

Après avoir :doc:`chargé vos données</orm/retrieving-data-and-resultsets>` vous
voudrez probablement mettre à jour & sauvegarder les changements.

Coup d'Oeil sur Enregistrement des Données
==========================================

Les applications ont habituellement deux façons d'enregistrer les données.
La première est évidemment via des formulaires web et l'autre en générant ou
modifiant directement les données dans le code pour l'envoyer à la base de
données.

Insérer des Données
-------------------

Le moyen le plus simple d'insérer des données dans une base de données est de
créer une nouvelle entity et de la passer à la méthode ``save()`` de la classe
``Table``::

    use Cake\ORM\TableRegistry;

    $articlesTable = TableRegistry::get('Articles');
    $article = $articlesTable->newEntity();

    $article->title = 'A New Article';
    $article->body = 'Ceci est le contenu de cet article';

    if ($articlesTable->save($article)) {
        // L'entity $article contient maintenant l'id
        $id = $article->id;
    }

Mettre à jour des Données
-------------------------

La mise à jour est aussi simple et la méthode ``save()`` sert également ce
but::

    use Cake\ORM\TableRegistry;

    $articlesTable = TableRegistry::get('Articles');
    $article = $articlesTable->get(12); // article avec l'id 12

    $article->title = 'Un nouveau titre pour cet article';
    $articlesTable->save($article);

CakePHP saura s'il doit faire un ajout ou une mise à jour en se basant sur le
résultat de la méthode ``isNew()``. Les entities qui sont récupérées via
``get()`` ou  ``find()`` renverrons toujours ``false`` lorsque la méthode
``isNew()`` est appelée sur eux.

Enregistrements avec Associations
---------------------------------

Par défaut, la méthode ``save()`` ne sauvegardera qu'un seul niveau
d'association::

    $articlesTable = TableRegistry::get('Articles');
    $author = $articlesTable->Authors->findByUserName('mark')->first();

    $article = $articlesTable->newEntity();
    $article->title = 'Un article par mark';
    $article->author = $author;

    if ($articlesTable->save($article)) {
        // La valeur de la clé étrangère a été ajoutée automatiquement.
        echo $article->author_id;
    }

La méthode ``save()`` est également capable de créer de nouveaux
enregistrements pour les associations::

    $firstComment = $articlesTable->Comments->newEntity();
    $firstComment->body = 'Un super article';

    $secondComment = $articlesTable->Comments->newEntity();
    $secondComment->body = 'J aime lire ceci!';

    $tag1 = $articlesTable->Tags->findByName('cakephp')->first();
    $tag2 = $articlesTable->Tags->newEntity();
    $tag2->name = 'Génial';

    $article = $articlesTable->get(12);
    $article->comments = [$firstComment, $secondComment];
    $article->tags = [$tag1, $tag2];

    $articlesTable->save($article);

Associer des Enregistrements Many to Many
-----------------------------------------

Dans le code ci-dessus il y a déjà un exemple de liaison d'un article vers
deux tags. Il y a un autre moyen de faire la même chose en utilisant la
méthode ``link()`` dans l'association::

    $tag1 = $articlesTable->Tags->findByName('cakephp')->first();
    $tag2 = $articlesTable->Tags->newEntity();
    $tag2->name = 'Génial';

    $articlesTable->Tags->link($article, [$tag1, $tag2]);

Sauvegarder des Données dans la Table de Jointure
-------------------------------------------------

L'enregistrement de données dans la table de jointure est réalisé en utilisant
la propriété spéciale ``_joinData``. Cette propriété doit être une instance
d'Entity de la table de jointure::

    $tag1 = $articlesTable->Tags->findByName('cakephp')->first();
    $tag1->_joinData = $articlesTable->ArticlesTags->newEntity();
    $tag1->_joinData->tagComment = 'Je pense que cela est lié à CakePHP';

    $ArticlesTags->link($article, [$tag1]);

Délier les Enregistrements Many To Many
---------------------------------------

Délier des enregistrements Many to Many (plusieurs à pluiseurs) est réalisable
via la méthode ``unlink()``::

    $tags = $articlesTable
        ->Tags
        ->find()
        ->where(['name IN' => ['cakephp', 'awesome']])
        ->toArray();

    $articlesTable->Tags->unlink($article, $tags);

Lors de la modification d'enregistrements en définissant ou modifiant
directement leurs propriétés il n'y aura pas de validation, ce qui est
problématique pour l'acceptation de données de formulaire. La section suivante
va vous expliquer comment convertir efficacement les données de formulaire
en entities afin qu'elles puissent être validées et sauvegardées.

.. _converting-request-data:

Convertir les Données Requêtées en Entities
===========================================

Avant de modifier et sauvegarder à nouveau les données dans la base de données,
vous devrez convertir les données requêtées (qui se trouvent dans
$this->request->data) à partir du format de tableau
qui se trouvent dans la requête, et les entities que l'ORM utilise. La classe
Table facilite la conversion d'une ou de plusieurs entities à partir des
données requêtées. Vous pouvez convertir une entity unique en utilisant::

    // Dans un controller.
    $articles = TableRegistry::get('Articles');
    $entity = $articles->newEntity($this->request->data());

Les données requêtées doivent suivre la structure de vos entities. Par
exemple si vous avez un article qui appartient à un utilisateur, et si vous
avez plusieurs commentaires, vos données requêtées devraient ressembler
à ceci::

    $data = [
        'title' => 'My title',
        'body' => 'The text',
        'user_id' => 1,
        'user' => [
            'username' => 'mark'
        ],
        'comments' => [
            ['body' => 'First comment'],
            ['body' => 'Second comment'],
        ]
    ];

Lors de la construction de formulaires qui sauvegardent des associations
imbriquées, vous devez définir quelles associations doivent être marshalled::

    // Dans un controller
    $articles = TableRegistry::get('Articles');
    $entity = $articles->newEntity($this->request->data(), [
        'associated' => [
            'Tags', 'Comments' => ['associated' => ['Users']]
        ]
    ]);

Ce qui est au-dessus indique que les 'Tags', 'Comments' et 'Users' pour les
Comments doivent être marshalled. D'une autre façon, vous pouvez utiliser
la notation par point pour être plus bref::

    // Dans un controller.
    $articles = TableRegistry::get('Articles');
    $entity = $articles->newEntity($this->request->data(), [
        'associated' => ['Tags', 'Comments.Users']
    ]);

Convertir des Données BelongsToMany
-----------------------------------

Si vous sauvegardez des associations belongsToMany, vous pouvez soit utiliser
une liste de données d'entity ou une liste d'ids. Quand vous utilisez une
liste de données d'entity, vos données requêtées devraient ressembler à ceci::

    $data = [
        'title' => 'My title',
        'body' => 'The text',
        'user_id' => 1,
        'tags' => [
            ['tag' => 'CakePHP'],
            ['tag' => 'Internet'],
        ]
    ];

Le code ci-dessus créera 2 nouveaux tags. Si vous voulez créer un lien d'un
article  vers des tags existants, vous pouvez utiliser une lite des ids.
Vos données de requête doivent ressembler à ceci::

    $data = [
        'title' => 'My title',
        'body' => 'The text',
        'user_id' => 1,
        'tags' => [
            '_ids' => [1, 2, 3, 4]
        ]
    ];

If you need to link against some existing belongsToMany records, and create new
ones at the same time you can use an expanded format::

    $data = [
        'title' => 'My title',
        'body' => 'The text',
        'user_id' => 1,
        'tags' => [
            ['name' => 'A new tag'],
            ['name' => 'Another new tag'],
            ['id' => 5],
            ['id' => 21]
        ]
    ];

When the above data is converted into entities, you will have 4 tags. The first
two will be new objects, and the second two will be references to existing
records.

Convertir des Données HasMany
-----------------------------

Si vous sauvegardez des associations hasMany et voulez lier des enregistrements
existants à un nouveau parent, vous pouvez utiliser le format ``_ids``::

    $data = [
        'title' => 'My new article',
        'body' => 'The text',
        'user_id' => 1,
        'comments' => [
            '_ids' => [1, 2, 3, 4]
        ]
    ];

Convertir des Enregistrements Multiples
---------------------------------------

Lorsque vous créez des formulaires de création/mise à jour d'enregistrements
multiples en une seule opération vous pouvez utiliser ``newEntities()``::

    // Dans un controller.
    $articles = TableRegistry::get('Articles');
    $entities = $articles->newEntities($this->request->data());

Dans cette situation, les données de requête pour plusieurs articles doivent
ressembler à ceci::

    $data = [
        [
            'title' => 'First post',
            'published' => 1
        ],
        [
            'title' => 'Second post',
            'published' => 1
        ],
    ];

.. _changing-accessible-fields:

Changer les Champs Accessibles
------------------------------

Il est également possible de permettre à ``newEntity()`` d'écrire dans des
champs non accessibles. Par exemple, ``id`` est généralement absent de la
propriété ``_accessible``. Dans ce cas, vous pouvez utiliser l'option
``accessibleFields``. Cela est particulièrement intéressant pour conserver les
associations existantes entre certaines entities::

    // Dans un controller.
    $articles = TableRegistry::get('Articles');
    $entity = $articles->newEntity($this->request->data(), [
        'associated' => [
            'Tags', 'Comments' => [
                'associated' => [
                    'Users' => [
                        'accessibleFields' => ['id' => true]
                    ]
                ]
            ]
        ]
    ]);

Le code ci-dessus permet de conserver l'association entre Comments et Users pour
l'entity concernée.

Une fois que vous avez converti les données requêtées dans des entities, vous
pouvez leur faire un ``save()`` ou un ``delete()``::

    // Dans un controller.
    foreach ($entities as $entity) {
        // Save entity
        $articles->save($entity);

        // Supprime l'entity
        $articles->delete($entity);
    }

Ce qui est au-dessus va lancer une transaction séparée pour chaque entity
sauvegardée. Si vous voulez traiter toutes les entities en transaction unique,
vous pouvez utiliser ``transactional()``::

    // Dans un controller.
    $articles->connection()->transactional(function () use ($articles, $entities) {
        foreach ($entities as $entity) {
            $articles->save($entity, ['atomic' => false]);
        }
    });

.. note::

    Si vous utilisez newEntity() et qu'il manque quelques unes ou toutes les
    données des entities résultantes, vérifiez deux fois que les colonnes que
    vous souhaitez définir sont listées dans la propriété ``$_accessible``
    de votre entity.

Fusionner les Données Requêtées dans les Entities
-------------------------------------------------

Afin de mettre à jour les entities, vous pouvez choisir d'appliquer les données
requêtées directement dans une entity existante. Ceci a l'avantage que seuls les
champs qui changent réellement seront sauvegardés, au lieu d'envoyer tous les
champs à la base de données, même ceux qui sont identiques. Vous pouvez
fusionner un tableau de données brutes dans une entity existante en utilisant la
méthode ``patchEntity()``::

    // Dans un controller.
    $articles = TableRegistry::get('Articles');
    $article = $articles->get(1);
    $articles->patchEntity($article, $this->request->data());
    $articles->save($article);

Comme expliqué dans la section précédente, les données requêtées doivent suivre
la structure de votre entity. La méthode ``patchEntity()`` est également capable
de fusionner les associations, par défaut seul les premiers niveaux
d'associations sont fusionnés mais si vous voulez contrôler la liste des
associations à fusionner ou fusionner des niveaux de plus en plus profonds, vous
pouvez utiliser le troisième paramètre de la méthode::

    // Dans un controller.
    $article = $articles->get(1);
    $articles->patchEntity($article, $this->request->data(), [
        'associated' => ['Tags', 'Comments.Users']
    ]);
    $articles->save($article);

Les associations sont fusionnées en faisant correspondre le champ de clé
primaire dans la source entities avec les champs correspondants dans le tableau
de données. Pour des associations belongsTo et hasOne, les nouvelles entities
seront construites si aucune entity précédente n'est trouvé pour la propriété
cible.

Pa exemple, prenons les données requêtées comme ce qui suit::

    $data = [
        'title' => 'My title',
        'user' => [
            'username' => 'mark'
        ]
    ];

Essayer de faire un patch d'une entity sans entity dans la propriété user va
créer une nouvelle entity user::

    // Dans un controller.
    $entity = $articles->patchEntity(new Article, $data);
    echo $entity->user->username; // Echoes 'mark'

La même chose peut être dite pour les associations hasMany et belongsToMany,
mais une note importante doit être faîte.

.. note::

    For belongsToMany associations, ensure the relevant entity has
    a property accessible for the associated entity.


If a Product belongsToMany Tag::

    // in the Product Entity
    protected $_accessible = [
        // .. other properties
       'tags' => true,
    ];

.. note::

    Pour les associations hasMany et belongsToMany, s'il y avait des entities
    qui ne pouvaient pas correspondre avec leur clé primaire à aucun
    enregistrement dans le tableau de données, alors ces enregistrements
    seraient annulés de l'entity résultante.

    Rappelez-vous que l'utilisation de ``patchEntity()`` ou de
    ``patchEntities()`` ne fait pas persister les données, il modifie juste
    (ou créé) les entities données. Afin de sauvegarder l'entity, vous devrez
    appeler la méthode ``save()``.

Par exemple, considérons le cas suivant::

    $data = [
        'title' => 'My title',
        'body' => 'The text',
        'comments' => [
            ['body' => 'First comment', 'id' => 1],
            ['body' => 'Second comment', 'id' => 2],
        ]
    ];
    $entity = $articles->newEntity($data);

    $newData = [
        'comments' => [
            ['body' => 'Changed comment', 'id' => 1],
            ['body' => 'A new comment'],
        ]
    ];
    $articles->patchEntity($entity, $newData);
    $articles->save($article);

A la fin, si l'entity est à nouveau convertie en tableau, vous obtiendrez le
résultat suivant::

    [
        'title' => 'My title',
        'body' => 'The text',
        'comments' => [
            ['body' => 'Changed comment', 'id' => 1],
            ['body' => 'A new comment'],
        ]
    ];

Comme vous l'avez vu, le commentaire avec l'id 2 n'est plus ici, puisqu'il ne
correspondait à rien dans le tableau ``$newData``. Ceci est fait ainsi pour
mieux capturer l'intention du post des données requêtées. Les données envoyées
reflètent le nouvel état que l'entity doit avoir.

Des avantages supplémentaires à cette approche sont qu'elle réduit le nombre
d'opérations à exécuter quand on fait persister l'entity à nouveau.

Notez bien que ceci ne signifie pas que le commentaire avec l'id 2 a été
supprimé de la base de données, si vous souhaitez retirer les commentaires pour
cet article qui ne sont pas présents dans l'entity, vous pouvez collecter
les clés primaires et exécuter une suppression batch pour celles qui ne sont
pas dans la liste::

    // Dans un controller.
    $comments = TableRegistry::get('Comments');
    $present = (new Collection($entity->comments))->extract('id')->toArray();
    $comments->deleteAll([
        'article_id' => $article->id,
        'id NOT IN' => $present
    ]);

Comme vous pouvez le voir, ceci permet aussi de créer des solutions lorsqu'une
association a besoin d'être implémentée comme un ensemble unique.

Vous pouvez aussi faire un patch de plusieurs entities en une fois. Les
considérations faîtes pour les associations hasMany et belongsToMany
s'appliquent pour le patch de plusieurs entities: Les correspondances sont
faites avec la valeur du champ de la clé primaire et les correspondances
manquantes dans le tableau original des entities seront retirées et non
présentes dans les résultats::

    // Dans un controller.
    $articles = TableRegistry::get('Articles');
    $list = $articles->find('popular')->toArray();
    $patched = $articles->patchEntities($list, $this->request->data());
    foreach ($patched as $entity) {
        $articles->save($entity);
    }

De la même façon que pour l'utilisation de ``patchEntity()``, vous pouvez
utiliser le troisième argument pour contrôler les associations qui seront
fusionnées dans chacune des entities du tableau::

    // Dans un controller.
    $patched = $articles->patchEntities(
        $list,
        $this->request->data(),
        ['associated' => ['Tags', 'Comments.Users']]
    );

.. _before-marshal:

Modifier les Données Requêtées Avant de Construire les Entities
---------------------------------------------------------------

Si vous devez modifier les données requêtées avant qu'elles ne soient
converties en entities, vous pouvez utiliser l'event ``Model.beforeMarshal``.
Cet event vous laisse manipuler les données requêtées juste avant que les
entities ne soient créées::

    // Dans une classe table ou behavior
    public function beforeMarshal(Event $event, ArrayObject $data, ArrayObject $options)
    {
        $data['username'] .= 'user';
    }

Le paramètre ``$data`` est une instance ``ArrayObject``, donc vous n'avez pas
à la retourner pour changer les données utilisées pour créer les entities.

Validating Data Before Building Entities
----------------------------------------

The :doc:`/orm/validation` chapter has more information on how to use the
validation features of CakePHP to ensure your data stays correct and consistent.

Eviter les Attaques d'Assignement en Masse de Propriétés
--------------------------------------------------------

Lors de la création ou la fusion des entities à partir de données requêtées,
vous devez faire attention à ce que vous autorisez à changer ou à ajouter
dans les entities à vos utilisateurs. Par exemple, en envoyant un tableau
dans la requête contenant ``user_id``, un pirate pourrait changer le
propriétaire d'un article, ce qui entraînerait des effets indésirables::

    // Contient ['user_id' => 100, 'title' => 'Hacked!'];
    $data = $this->request->data;
    $entity = $this->patchEntity($entity, $data);
    $this->save($entity);

Il y a deux façons de se protéger pour ce problème. La première est de définir
les colonnes par défaut qui peuvent être définies en toute sécurité à partir
d'une requête en utilisant la fonctionnalité d':ref:`entities-mass-assignment`
dans les entities.

La deuxième façon est d'utiliser l'option ``fieldList`` lors de la création ou
la fusion de données dans une entity::

    // Contient ['user_id' => 100, 'title' => 'Hacked!'];
    $data = $this->request->data;

    // Permet seulement de changer le title
    $entity = $this->patchEntity($entity, $data, [
        'fieldList' => ['title']
    ]);
    $this->save($entity);

Vous pouvez aussi contrôler les propriétés qui peuvent être assignées pour les
associations::

    // Permet seulement le changement de title et de tags
    // et le nom du tag est la seule colonne qui peut être définie
    $entity = $this->patchEntity($entity, $data, [
        'fieldList' => ['title', 'tags'],
        'associated' => ['Tags' => ['fieldList' => ['name']]]
    ]);
    $this->save($entity);

Utiliser cette fonctionnalité est pratique quand vous avez différentes fonctions
auxquelles vos utilisateurs peuvent accéder et que vous voulez laisser vos
utilisateurs modifier différentes données basées sur leurs privilèges.

L'option ``fieldList`` est aussi acceptée par les méthodes ``newEntity()``,
``newEntities()`` et ``patchEntities()``.

.. _saving-entities:

Sauvegarder les Entities
========================

.. php:method:: save(Entity $entity, array $options = [])

Quand vous sauvegardez les données requêtées dans votre base de données, vous
devez d'abord hydrater une nouvelle entity en utilisant ``newEntity()`` pour
passer dans ``save()``. Pare exemple::

  // Dans un controller
  $articles = TableRegistry::get('Articles');
  $article = $articles->newEntity($this->request->data);
  if ($articles->save($article)) {
      // ...
  }

L'ORM utilise la méthode ``isNew()`` sur une entity pour déterminer si oui ou
non une insertion ou une mise à jour doit être faite. Si la méthode
``isNew()`` retourne ``true`` et que l'entity a une valeur de clé primaire,
une requête 'exists' sera faîte. La requête 'exists' peut être supprimée en
passant ``'checkExisting' => false`` à l'argument ``$options`` ::

    $articles->save($article, ['checkExisting' => false]);

Une fois que vous avez chargé quelques entities, vous voudrez probablement les
modifier et les mettre à jour dans votre base de données. C'est un exercice
simple dans CakePHP::

    $articles = TableRegistry::get('Articles');
    $article = $articles->find('all')->where(['id' => 2])->first();

    $article->title = 'My new title';
    $articles->save($article);

Lors de la sauvegarde, CakePHP va
:ref:`appliquer vos règles de validation <application-rules>`, et
entourer l'opération de sauvegarde dans une transaction de base de données.
Cela va aussi seulement mettre à jour les propriétés qui ont changé. Le
``save()`` ci-dessus va générer le code SQL suivant::

    UPDATE articles SET title = 'My new title' WHERE id = 2;

Si vous avez une nouvelle entity, le code SQL suivant serait généré::

    INSERT INTO articles (title) VALUES ('My new title');

Quand une entity est sauvegardée, voici ce qui se passe:

1. La vérification des règles commencera si elle n'est pas désactivée.
2. La vérification des règles va déclencher l'event
   ``Model.beforeRules``. Si l'event est stoppé, l'opération de
   sauvegarde va connaitre un échec et retourner ``false``.
3. Les règles seront vérifiées. Si l'entity est en train d'être créée, les
   règles ``create`` seront utilisées. Si l'entity est en train d'être mise à
   jour, les règles ``update`` seront utilisées.
4. L'event ``Model.afterRules`` sera déclenché.
5. L'event ``Model.beforeSave`` est dispatché. S'il est stoppé, la
   sauvegarde sera annulée, et save() va retourner ``false``.
6. Les associations parentes sont sauvegardées. Par exemple, toute association
   belongsTo listée sera sauvegardée.
7. Les champs modifiés sur l'entity seront sauvegardés.
8. Les associations Enfant sont sauvegardées. Par exemple, toute association
   hasMany, hasOne, ou belongsToMany listée sera sauvegardée.
9. L'event ``Model.afterSave`` sera dispatché.

Consultez la section :ref:`application-rules` pour plus d'informations sur la
création et l'utilisation des règles.

.. warning::

    Si aucun changement n'est fait à l'entity quand elle est sauvegardée, les
    callbacks ne vont pas être déclenchés car aucune sauvegarde n'est faîte.

La méthode ``save()`` va retourner l'entity modifiée en cas de succès, et
``false`` en cas d'échec. Vous pouvez désactiver les règles et/ou les
transactions en utilisant l'argument ``$options`` pendant la sauvegarde::

    // Dans un controller ou une méthode de table.
    $articles->save($article, ['atomic' => false]);

Sauvegarder les Associations
----------------------------

Quand vous sauvegardez une entity, vous pouvez aussi choisir d'avoir quelques
unes ou toutes les entities associées. Par défaut, toutes les entities de
premier niveau seront sauvegardées. Par exemple sauvegarder un Article, va
aussi automatiquement mettre à jour tout entity modifiée qui n'est pas
directement liée à la table articles.

Vous pouvez régler finement les associations qui sont sauvegardées en
utilisant l'option ``associated``::

    // Dans un controller.

    // Sauvegarde seulement l'association avec les commentaires
    $articles->save($entity, ['associated' => ['Comments']]);

Vous pouvez définir une sauvegarde distante ou des associations imbriquées
profondément en utilisant la notation par point::

    // Sauvegarde la company, les employees et les addresses liées pour chacun d'eux.
    $companies->save($entity, ['associated' => ['Employees.Addresses']]);

Si vous avez besoin de lancer un ensemble de règle de validation différente pour
une association, vous pouvez le spécifier dans un tableau d'options pour
l'association::

    // Dans un controller.

    // Sauvegarde la company, les employees et les addresses liées pour chacun d'eux.
    $companies->save($entity, [
      'associated' => [
        'Employees' => [
          'associated' => ['Addresses'],
        ]
      ]
    ]);

En plus, vous pouvez combiner la notation par point pour les associations avec
le tableau d'options::

    $companies->save($entity, [
      'associated' => [
        'Employees',
        'Employees.Addresses'
      ]
    ]);

Vos entities doivent être structurées de la même façon qu'elles l'étaient
quand elles ont été chargées à partir de la base de données.
Consultez la documentation du helper Form pour savoir comment
:ref:`associated-form-inputs`.

Si vous construisez ou modifiez une donnée d'association après avoir construit
vos entities, vous devrez marquer la propriété d'association comme étant
modifiée avec ``dirty()``::

    $company->author->name = 'Master Chef';
    $company->dirty('author', true);

Sauvegarder les Associations BelongsTo
--------------------------------------

Lors de la sauvegarde des associations belongsTo, l'ORM s'attend à une entity
imbriquée unique avec le nom de l'association au singulier, en underscore.
Par exemple::

    // Dans un controller.
    $data = [
        'title' => 'First Post',
        'user' => [
            'id' => 1,
            'username' => 'mark'
        ]
    ];
    $articles = TableRegistry::get('Articles');
    $article = $articles->newEntity($data, [
        'associated' => ['Users']
    ]);

    $articles->save($article);

Sauvegarder les Associations HasOne
-----------------------------------

Lors de la sauvegarde d'associations hasOne, l'ORM s'attend à une entity
imbriquée unique avec le nom de l'association au singulier et en underscore.
Par exemple::

    // Dans un controller.
    $data = [
        'id' => 1,
        'username' => 'cakephp',
        'profile' => [
            'twitter' => '@cakephp'
        ]
    ];
    $users = TableRegistry::get('Users');
    $user = $users->newEntity($data, [
        'associated' => ['Profiles']
    ]);
    $users->save($user);

Sauvegarder les Associations HasMany
------------------------------------

Lors de la sauvegarde d'associations hasMany, l'ORM s'attend à une entity
imbriquée unique avec le nom de l'association au pluriel et en underscore.
Par exemple::

    // Dans un controller.
    $data = [
        'title' => 'First Post',
        'comments' => [
            ['body' => 'Best post ever'],
            ['body' => 'I really like this.']
        ]
    ];
    $articles = TableRegistry::get('Articles');
    $article = $articles->newEntity($data, [
        'associated' => ['Comments']
    ]);
    $articles->save($article);

Lors de la sauvegarde d'associations hasMany, les enregistrements associés
seront soit mis à jour, soit insérés. L'ORM ne va pas retirer ou 'synchroniser'
une association hasMany. Peu importe quand vous ajoutez de nouveaux
enregistrements dans une association existante, vous devez toujours marquer la
propriété de l'association comme 'dirty'. Ceci dit à l'ORM que la propriété de
l'association doit persister::

    $article->comments[] = $comment;
    $article->dirty('comments', true);

Sans l'appel à ``dirty()``, les commentaires mis à jour ne seront pas
sauvegardés.

Sauvegarder les Associations BelongsToMany
------------------------------------------

Lors de la sauvegarde d'associations hasMany, l'ORM s'attend à une entity
imbriquée unique avec le nom de l'association au pluriel et en underscore.
Par exemple::

    // Dans un controller.

    $data = [
        'title' => 'First Post',
        'tags' => [
            ['tag' => 'CakePHP'],
            ['tag' => 'Framework']
        ]
    ];
    $articles = TableRegistry::get('Articles');
    $article = $articles->newEntity($data, [
        'associated' => ['Tags']
    ]);
    $articles->save($article);

Quand vous convertissez les données requêtées en entities, les méthodes
``newEntity()`` et ``newEntities()`` vont gérer les deux tableaux de propriétés,
ainsi qu'une liste d'ids avec la clé ``_ids``. Utiliser la clé ``_ids``
facilite la construction d'un box select ou d'un checkbox basé sur les
contrôles pour les associations belongs to many. Consultez la section
:ref:`converting-request-data` pour plus d'informations.

Lors de la sauvegarde des associations belongsToMany, vous avez le choix entre
2 stratégies de sauvegarde:

append
    Seuls les nouveaux liens seront créés de chaque côté de cette
    association. Cette stratégie ne va pas détruire les liens existants même
    s'ils ne sont pas présents dans le tableau d'entities à sauvegarder.
replace
    Lors de la sauvegarde, les liens existants seront retirés et les nouveaux
    liens seront créés dans la table de jointure. S'il y a des liens existants
    dans la base de données vers certaines entities que l'on souhaite
    sauvegarder, ces liens seront mis à jour, non supprimés et re-sauvegardés.

Par défaut la stratégie ``replace`` est utilisée. Quand vous avez de nouveaux
enregistrements dans une association existante, vous devez toujours marquer
la propriété de l'association en 'dirty'. Ceci dit à l'ORM que la propriété
de l'association doit persister::

    $article->tags[] = $tag;
    $article->dirty('tags', true);

Sans appel à ``dirty()``, les tags mis à jour ne seront pas sauvegardés.

Often you'll find yourself wanting to make an association between two existing
entities, eg. a user coauthoring an article. This is done by using the method
``link()``, like this::

    $article = $this->Articles->get($articleId);
    $user = $this->Users->get($userId);

    $this->Articles->Users->link($article, [$user]);

When saving belongsToMany Associations, it can be relevant to save some
additional data to the Joint Table.  In the previous example of tags, it could
be the ``vote_type`` of person who voted on that article.  The ``vote_type`` can
be either ``upvote`` or ``downvote`` and is represented by a string.  The
relation is between Users and Articles.

Saving that association, and the ``vote_type`` is done by first adding some data
to ``_joinData`` and then saving the association with ``link()``, example::

    $article = $this->Articles->get($articleId);
    $user = $this->Users->get($userId);

    $user->_joinData = new Entity(['vote_type' => $voteType, ['markNew' => true]]);
    $this->Articles->Users->link($article, [$user]);

Sauvegarder des Données Supplémentaires à la Table de Jointure
--------------------------------------------------------------

Dans certaines situations, la table de jointure de l'association BelongsToMany,
aura des colonnes supplémentaires. CakePHP facilite la sauvegarde des
propriétés dans ces colonnes. Chaque entity dans une association belongsToMany
a une propriété ``_joinData`` qui contient les colonnes supplémentaires sur la
table de jointure. Ces données peuvent être soit un tableau, soit une instance
Entity. Par exemple si les Students BelongsToMany Courses, nous pourrions
avoir une table de jointure qui ressemble à ceci::

    id | student_id | course_id | days_attended | grade

Lors de la sauvegarde de données, vous pouvez remplir les colonnes
supplémentaires sur la table de jointure en définissant les données dans la
propriété ``_joinData``::

    $student->courses[0]->_joinData->grade = 80.12;
    $student->courses[0]->_joinData->days_attended = 30;

    $studentsTable->save($student);

La propriété ``_joinData`` peut être soit une entity, soit un tableau de données
si vous sauvegardez les entities construites à partir de données
requêtées. Lorsque vous sauvegardez des données de tables jointes depuis les données
requêtées, vos données POST doivent ressembler à ceci::

    $data = [
        'first_name' => 'Sally',
        'last_name' => 'Parker',
        'courses' => [
            [
                'id' => 10,
                '_joinData' => [
                    'grade' => 80.12,
                    'days_attended' => 30
                ]
            ],
            // d'autres cours (courses).
        ]
    ];
    $student = $this->Students->newEntity($data, [
        'associated' => ['Courses._joinData']
    ]);

Regardez le chapitre sur les :ref:`inputs pour les données associées
<associated-form-inputs>` pour savoir comment construire des inputs avec
le ``FormHelper`` correctement.

.. _saving-complex-types:

Sauvegarder les Types Complexes
-------------------------------

Les tables peuvent stocker des données représentées dans des types basiques,
comme les chaînes, les integers, floats, booleans, etc... Mais elles peuvent
aussi être étendues pour accepter plus de types complexes comme les tableaux
ou les objets et sérialiser ces données en types plus simples qui peuvent
être sauvegardés dans la base de données.

Cette fonctionnalité se fait en utilisant le système personnalisé de types.
Consulter la section :ref:`adding-custom-database-types` pour trouver comment
construire les Types de colonne personnalisés::

    // Dans config/bootstrap.php
    use Cake\Database\Type;
    Type::map('json', 'App\Database\Type\JsonType');

    // Dans src/Model/Table/UsersTable.php
    use Cake\Database\Schema\Table as Schema;

    class UsersTable extends Table
    {

        protected function _initializeSchema(Schema $schema)
        {
            $schema->columnType('preferences', 'json');
            return $schema;
        }

    }

Le code ci-dessus correspond à la colonne ``preferences`` pour le type
personnalisé ``json``.
Cela signifie que quand on récupère des données pour cette colonne, elles seront
désérialisées à partir d'une chaîne JSON dans la base de données et mises
dans une entity en tant que tableau.

Comme ceci, lors de la sauvegarde, le tableau sera transformé à nouveau en sa
représentation JSON::

    $user = new User([
        'preferences' => [
            'sports' => ['football', 'baseball'],
            'books' => ['Mastering PHP', 'Hamlet']
        ]
    ]);
    $usersTable->save($user);

Lors de l'utilisation de types complexes, il est important de vérifier que les
données que vous recevez de l'utilisateur final sont valides. Ne pas
gérer correctement les données complexes va permettre à des
utilisateurs mal intentionnés d'être capable de stocker des données qu'ils ne
pourraient pas stocker normalement.

Mises à Jour en Masse
=====================

.. php:method:: updateAll($fields, $conditions)

Il peut arriver que la mise à jour de lignes individuellement n'est pas
efficace ou pas nécessaire. Dans ces cas, il est plus efficace d'utiliser une
mise à jour en masse pour modifier plusieurs lignes en une fois::

    // Publie tous les articles non publiés.
    function publishAllUnpublished()
    {
        $this->updateAll(['published' => true], ['published' => false]);
    }

Si vous devez faire des mises à jour en masse et utiliser des expressions SQL,
vous devrez utiliser un objet expression puisque ``updateAll()`` utilise
des requêtes préparées sous le capot::

    function incrementCounters()
    {
        $expression = new QueryExpression('view_count = view_count + 1');
        $this->updateAll([$expression], ['published' => true]);
    }

Une mise à jour en masse sera considérée comme un succès si 1 ou plusieurs
lignes sont mises à jour.

.. warning::

    updateAll *ne* va *pas* déclencher d'events beforeSave/afterSave. Si
    vous avez besoin de ceux-ci, chargez d'abord une collection
    d'enregistrements et mettez les à jour.
