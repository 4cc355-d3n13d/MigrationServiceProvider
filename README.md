#Introduction
This is one fork from [KnpLabs/MigrationServiceProvider](https://github.com/KnpLabs/MigrationServiceProvider) which is altered for working in silex 2 and add migrate to a specific version
# Migrations

This is a simple homebrew schema migration system for silex and doctrine.

## Install

As usual, just include `knplabs/migration-service-provider` in your `composer.json` (don't tell me you don't have one, it's 2012 already), and register the service. You will have to pass the `migration.path` option, which should contain the path to your migrations files:

```php
$app->register(new \Knp\Provider\MigrationServiceProvider(), array(
    'migration.path' => __DIR__.'/../src/Resources/migration'
));
```

## Enough small talk, I want to write migrations!

And I am too lazy to write a comprehensive documentation right now, so you will have to rely on two external resources:

1. [The marketplace's migrations](https://github.com/KnpLabs/marketplace/tree/master/src/Resources/migrations)
2. [The official documentation for Doctrine's DBAL Schema Manager](http://readthedocs.org/docs/doctrine-dbal/en/latest/reference/schema-manager.html)

## Running migrations

There are two ways of running migrations

### Using the `before` handler

If you pass a `migration.register_before_handler` (set to `true`) when registering the service, then a `before` handler will be registered for migration to be run. It means that the migration manager will be run for each hit to your application.

You might want to enable this behavior for development mode, but please don't do that in production!

### Using the `knp:migration:migrate` command

If you installed the console service provider right, you can use the `knp:migration:migrate` command.

## Writing migrations

A migration consist of a single file, holding a migration class. By design, the migration file must be named something like `<version>_<migration_name>Migration.php` and located in `src/Resources/migrations`, and the class `<migration_name>Migration`. For example, if your migration adds a `bar` field to the `foo` table, and is the 5th migration of your schema, you should name your file `05_FooBarMigration.php`, and the class would be named `FooBarMigration`.

In addition to these naming conventions, your migration class must extends `Knp\Migration\AbstractMigration`, which provides a few helping method such as `getVersion` and default implementations for migration methods.

The migration methods consist of 4 methods:

* `schemaUp`
* `schemaDown`
* `appUp`
* `appDown`

The names are pretty self-explanatory. Each `schema*` method is fed a `Doctrine\DBAL\Schema\Schema` instance of which you're expected to work to add, remove or modify fields and/or tables. The `app*` method are given a `Silex\Application` instance, actually your very application. You can see an example of useful `appUp` migration in the marketplace's [CommentMarkdownCacheMigration](https://github.com/knplabs/marketplace/blob/master/src/Resources/migrations/04_CommentMarkdownCacheMigration.php).

## Migration infos

There's one last method you should know about: `getMigrationInfo`. This method should return a self-explanatory description of the migration (it is optional though, and you can skip its implementation). When a migration implementing the `getMigrationInfo` method is run, and if you use twig, a global variable is set in your twig environment containing an array of all run migration informations.

You can then use it with something like that:

```html
      {% if migration_infos is defined %}
        <div class="alert alert-success">
          <p>Some migrations have been run:</p>
          <ul>
          {% for info in migration_infos %}
            <li>{{ info }}</li>
          {% endfor %}
        </div>
      {% endif %}
```
## Examples
Add your migrate command: MigrationToCommand.php
```php
...
use Knp\Command\Command;

use Symfony\Component\Console\Input\InputDefinition;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class MigrationToCommand extends Command
{
    public function configure()
    {
        $this->setName('mhl:migration:migrate')
        ->addArgument('version', InputArgument::OPTIONAL, 'The version number (1 or 2 or ...) to migrate to.', null);
    }

    public function execute(InputInterface $input, OutputInterface $output)
    {
        $app        = $this->getSilexApplication();
        $manager    = $app['migration'];

        if (!$manager->hasVersionInfo()) {
            $manager->createVersionInfo();
        }

        $targetVersion = $input->getArgument('version');
        if (isset($targetVersion)){
        	$res = $manager->migrateTo($targetVersion);
        } else {
        	$res = $manager->migrate();
        }
        switch($res) {
            case true:
                $output->writeln(sprintf('Succesfully executed <info>%d</info> migration(s)!', $manager->getMigrationExecuted()));
                foreach ($manager->getMigrationInfos() as $info) {
                    $output->writeln(sprintf(' - <info>%s</info>', $info));
                }
                break;
            case null:
                $output->writeln('No migrations to execute, you are up to date!');
                break;
        }
    }
}
```
Then create database version migrate files: 1_ProjectV1Migration.php. The name must be followed `version_*Migration`. If you have muntiple version for one table, the `*` must be unique for each table.
```php
<?php

namespace Migration;

use Doctrine\DBAL\Schema\Schema;
use Silex\Application;

class ProjectV1Migration
{
    public function schemaUp(Schema $schema)
    {
        $projectTable = $schema->createTable('project');
        $projectTable->addColumn('id', 'integer', array(
            'unsigned'      => true,
            'autoincrement' => true
        ));
        $projectTable->addColumn('name', 'string');
        $projectTable->addColumn('description', 'text');
        $projectTable->addColumn('username', 'string');
        $projectTable->setPrimaryKey(array('id'));
        $projectTable->addUniqueIndex(array('name'));
    }
    public function getMigrationInfo()
    {
        return 'Added project table';
    }
    public function getVersion()
    {
        $rc = new \ReflectionClass($this);
        if (preg_match('/^(\d+)/', basename($rc->getFileName()), $matches)) {
            return (int) ltrim($matches[1], 0);
        }
        throw new RuntimeError(sprintf('Could not get version from "%"', basename($rc->getFileName())));
    }
    public function schemaDown(Schema $schema) {}
    public function appUp(Application $app) {}
    public function appDown(Application $app) {}
}
```
