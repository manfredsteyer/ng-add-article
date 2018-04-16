# Seamlessly Updating your Angular Libraries with ng update

Updating libraries within your npm/yarn-based project can be a nightmare. Once you've dealt with all the peer dependencies, you have to make sure your source code doesn't run into breaking changes. 

The new command ``ng update`` provides a remedy: It goes trough all updated dependencies -- including the transitive ones -- and calls schematics to update the current project for them. Together with [ng add described in my blog article here](http://www.softwarearchitekt.at/post/2018/03/20/custom-schematics-part-iv-frictionless-library-setup-with-the-angular-cli-and-schematics.aspx), it is the foundation for an eco system allowing a more frictionless package management.

In this post, I'm showing how to make use of ``ng update`` within an existing library by extending the [simple logger used in my article about ng add](http://www.softwarearchitekt.at/post/2018/03/20/custom-schematics-part-iv-frictionless-library-setup-with-the-angular-cli-and-schematics.aspx). 

If you want to look at the [completed example](https://github.com/manfredsteyer/schematics-ng-add.git), you find it in my [GitHub repo](https://github.com/manfredsteyer/schematics-ng-add.git).

## Introducing a Breaking Change

To showcase ``ng update``, I'm going to modify my logger library here. For this, I'm renaming the ``LoggerModule``'s ``forRoot`` method into ``configure``:

```typescript
// logger.module.ts

[...]

@NgModule({
  [...]
})
export class LoggerModule { 
  // Old:
  // static forRoot(config: LoggerConfig): ModuleWithProviders {

  // New:
  static configure(config: LoggerConfig): ModuleWithProviders {
    [...]
  }
}
```

As this is just an example, please see this change just as a proxy for all the other breaking changes one might introduce with a new version.

## Creating the Migration Schematic

To adopt existing projects to my breaking change, I'm going to create a schematic for it. It will be placed into an new ``update`` folder within the library's ``schematics`` folder:

<img src="./img/update-schematic.png" width="250" alt="Folder update for new schematic">


 This new folder gets an ``index.ts`` when a rule factory:

```typescript
import { Rule, SchematicContext, Tree } from '@angular-devkit/schematics';

export function update(options: any): Rule {
  return (tree: Tree, _context: SchematicContext) => {

    _context.logger.info('Running update schematic ...');

    // Hardcoded path for the sake of simplicity
    const appModule = './src/app/app.module.ts';

    const buffer = tree.read(appModule);
    if (!buffer) return tree;
    const content = buffer.toString('utf-8');

    // One more time, this is for the sake of simplicity
    const newContent = content.replace('LoggerModule.forRoot(', 'LoggerModule.configure(');
    tree.overwrite(appModule, newContent);

    return tree;
  };
}
```

For the sake of simplicity, I'm taking two short cuts here. First, the rule assumes that the ``AppModule`` is located in the file ``./src/app/app.module.ts``. While this might be the case in a traditional Angular CLI project, one could also use a completely different folder structure. One example is a monorepo workspace containing several applications and libraries. I will present a solution for this in an other post but for now, let's stick with this simple solution.

To simplify things further, I'm directly modifying this file using a string replacement. A more safe way to change existing code is going with the TypeScript Compiler API. If you're interested into this, you'll find an example for this in my [blog post here](TODO!).

## Configuring the Migration Schematic

To configure migration schematics, let's follow the advice from the underlying [design document](TODO) and create an own collection. This collection is described by an ``migration-collection.json`` file:

<img src="./img/migration-collection.png" width="250" alt="Collection for migration schematics">

For each migration, it gets a schematic. The name of this schematic doesn't matter but what matters is the ``version`` property:

```json
{
  "schematics": {
    "migration-01": {
      "version": "4",
      "factory": "./update/index#update",
      "description": "updates to v4"
    }
  }
}
```

This collection tells the CLI to execute the current schematic when migrating to version 4. Let's assume we had such an schematic for version 5 too. If we migrated directly from version 3 to 5, the CLI would execute both. 

Instead of just pointing to a major version, we could also point to a minor or a patch version using version numbers like ``4.1`` or ``4.1.1``.

We also need to tell the CLI that this very file describes the migration schematics. For this, let's add an entry point ``ng-update`` to our ``package.json``. As in our example the ``package.json`` located in the project root is used by the library built, we have to modify this one. In other project setups the library could have an ``package.json`` of its own:

```json
[...]
"version": "4.0.0",
"schematics": "./schematics/collection.json",
"ng-update": {
  "migrations": "./schematics/migration-collection.json"
},
[...]
```

While the known ``schematics`` field is pointing to the traditional collection, ``ng-update`` shows which collection to use for migration.

We also need to increase the version within the ``package.json``. As my schematic is indented for version 4, I've set the ``version`` field to this very version above.

## Test, Publish, and Update

To test the migration schematic, we need a demo Angular application using the old version of the logger-lib. Some information about this can be found in my [last blog post]((http://www.softwarearchitekt.at/post/2018/03/20/custom-schematics-part-iv-frictionless-library-setup-with-the-angular-cli-and-schematics.aspx). This post also describes, how to setup a simple npm registry that provides the logger-lib and how to use it in your demo project.

Make sure to use the latest versions of ``@angular/cli`` and its dependency ``@angular-devkit/schematics``. When I wrote this up, I've used version ``6.0.0-rc.4`` of the CLI and version ``0.5.6`` of the schematics package. However, this came with some issues especially on Windows. Nether the less, I expect those issues to vanish, once we have version 6.

To ensure having the latest versions, I've installed the latest CLI and created a new application with it.

Sometimes during testing, it might be useful to install a former/ a specific version of the library. You can just use ``npm install`` for this:

    npm install @my/logger-lib@^0 --save

When everything is in place, we can build and publish the new version of our ``logger-lib``. For this, let's use the following commands in the library's root directory:

```
npm run build:lib
cd dist
cd lib
npm publish --registry http://localhost:4873
```

As in the [previous article](http://www.softwarearchitekt.at/post/2018/03/20/custom-schematics-part-iv-frictionless-library-setup-with-the-angular-cli-and-schematics.aspx), I'm using the npm registry verdaccio which is available at port 4863 by default.

## Updating the Library 

To update the logger-lib within our demo application, we can use the following command in it's root directory:

    ```
    ng update @my/logger-lib --registry http://localhost:4873 --force
    ```

The switch ``force`` makes ``ng update`` proceed even if there are unresolved peer dependencies. 

This command npm installs the newest version of the logger-lib and executes the registered migration script. After this, you should see the modifications within your ``app.module.ts`` file.

As an alternative, you could also npm install it by hand:

```
npm i @my/logger-lib@^4 --save
```

After this, you could run all the necessary migration schematics using ``ng update`` with the ``migrate-only`` switch:

```
ng update @my/logger-lib --registry http://localhost:4873 
--migrate-only --from=0.0.0 --force
```

This will execute all migration schematics to get from version 0.0.0 to the currently installed one. To just execute the migration schematics for a specific (former) version, you could make use of the ``--to`` switch:

```
ng update @my/logger-lib --registry http://localhost:4873 
--migrate-only --from=0.0.0 --to=4.0.0 --force
```






