# egg-ts-sequelize

`Sequelize@5` plugin for Egg.js.

[![NPM version][npm-image]][npm-url]
[![build status][travis-image]][travis-url]
[![Test coverage][codecov-image]][codecov-url]
[![David deps][david-image]][david-url]
[![Known Vulnerabilities][snyk-image]][snyk-url]
[![npm download][download-image]][download-url]

[npm-image]: https://img.shields.io/npm/v/egg-ts-sequelize.svg?style=flat-square
[npm-url]: https://npmjs.org/package/egg-ts-sequelize
[travis-image]: https://img.shields.io/travis/eggjs/egg-ts-sequelize.svg?style=flat-square
[travis-url]: https://travis-ci.org/eggjs/egg-ts-sequelize
[codecov-image]: https://img.shields.io/codecov/c/github/eggjs/egg-ts-sequelize.svg?style=flat-square
[codecov-url]: https://codecov.io/github/eggjs/egg-ts-sequelize?branch=master
[david-image]: https://img.shields.io/david/eggjs/egg-ts-sequelize.svg?style=flat-square
[david-url]: https://david-dm.org/eggjs/egg-ts-sequelize
[snyk-image]: https://snyk.io/test/npm/egg-ts-sequelize/badge.svg?style=flat-square
[snyk-url]: https://snyk.io/test/npm/egg-ts-sequelize
[download-image]: https://img.shields.io/npm/dm/egg-ts-sequelize.svg?style=flat-square
[download-url]: https://npmjs.org/package/egg-ts-sequelize

<!--
Description here.
-->

## Install

```bash
$ npm i @humandetail/egg-ts-sequelize --save
$ npm i mysql2 --save # For both mysql and mariadb dialects

# Or use other database backend.
$ npm install --save pg pg-hstore # PostgreSQL
$ npm install --save tedious # MSSQL
```

## Usage

```js
// {app_root}/config/plugin.js
exports.tsSequelize = {
  enable: true,
  package: '@humandetail/egg-ts-sequelize',
};
```

## Configuration

Edit your own configurations in `config/config.{env}.ts`

```js
// {app_root}/config/config.default.js
exports.tsSequelize = {
  dialect: 'mysql', // support: mysql, mariadb, postgres, mssql
  database: 'test',
  host: 'localhost',
  port: 3306,
  username: 'root',
  password: '',
  // delegate: 'model', // load all models to `app[delegate]` and `ctx[delegate]`, default to `model`
  // baseDir: 'model', // load all files in `app/${baseDir}` as models, default to `model`
  // exclude: 'index.js', // ignore `app/${baseDir}/index.js` when load models, support glob and array
  // more sequelize options
};
```

You can also use the `connection uri` to configure the connection:

```js
exports.sequelize = {
  dialect: 'mysql', // support: mysql, mariadb, postgres, mssql
  connectionUri: 'mysql://root:@127.0.0.1:3306/test',
  // delegate: 'myModel', // load all models to `app[delegate]` and `ctx[delegate]`, default to `model`
  // baseDir: 'my_model', // load all files in `app/${baseDir}` as models, default to `model`
  // exclude: 'index.js', // ignore `app/${baseDir}/index.js` when load models, support glob and array
  // more sequelize options
};
```

egg-ts-sequelize has a default sequelize options below

```js
{
  delegate: 'model',
  baseDir: 'model',
  logging(...args) {
    // if benchmark enabled, log used
    const used = typeof args[1] === 'number' ? `(${args[1]}ms)` : '';
    app.logger.info('[egg-ts-sequelize]%s %s', used, args[0]);
  },
  host: 'localhost',
  port: 3306,
  username: 'root',
  password: 'root',
  benchmark: true,
  define: {
    freezeTableName: false,
    underscored: true,
  },
};
```

see [Sequelize.js](http://docs.sequelizejs.com/manual/installation/usage.html) for more detail.

## Model files

Please put models under `app/model` dir by default.

## Conventions

|model file|class name|
|-|-|
|user.ts|app.model.User|
|project.ts|app.model.Project|
|address.ts|app.model.Address|

## Example

Define a model first.

> NOTE: `options.delegate` default to `model`, so `app.model` is an [Instance of Sequelize](http://docs.sequelizejs.com/class/lib/sequelize.js~Sequelize.html#instance-constructor-constructor), so you can use methods like: `app.model.sync, app.model.query ...`.

```js
// app/model/user.ts
import {
  Sequelize,
  Model,
  Optional,
  DataTypes,
  HasManyGetAssociationsMixin,
  HasManyAddAssociationMixin,
  HasManyHasAssociationMixin,
  HasManyCountAssociationsMixin,
  HasManyCreateAssociationMixin,
  Association
} from 'sequelize';

import { Project } from './Project';
import { Address } from './Address';

// These are all the attributes in the User model
interface UserAttributes {
  id: number;
  name: string;
  preferredName: string | null;
}

// Some attributes are optional in `User.build` and `User.create` calls
interface UserCreationAttributes extends Optional<UserAttributes, 'id'> {}

export class User extends Model<UserAttributes, UserCreationAttributes>
  implements UserAttributes {
  public id!: number; // Note that the `null assertion` `!` is required in strict mode.
  public name!: string;
  public preferredName!: string | null; // for nullable fields

  // timestamps!
  public readonly createdAt!: Date;
  public readonly updatedAt!: Date;

  // Since TS cannot determine model association at compile time
  // we have to declare them here purely virtually
  // these will not exist until `Model.init` was called.
  public getProjects!: HasManyGetAssociationsMixin<Project>; // Note the null assertions!
  public addProject!: HasManyAddAssociationMixin<Project, number>;
  public hasProject!: HasManyHasAssociationMixin<Project, number>;
  public countProjects!: HasManyCountAssociationsMixin;
  public createProject!: HasManyCreateAssociationMixin<Project>;

  // You can also pre-declare possible inclusions, these will only be populated if you
  // actively include a relation.
  public readonly projects?: Project[]; // Note this is optional since it's only populated when explicitly requested in code

  public static associations: {
    projects: Association<User, Project>;
  };
}

// required
export function init (sequelize: Sequelize) {
  User.init(
    {
      id: {
        type: DataTypes.INTEGER.UNSIGNED,
        autoIncrement: true,
        primaryKey: true,
      },
      name: {
        type: new DataTypes.STRING(128),
        allowNull: false,
      },
      preferredName: {
        field: 'preferred_name',
        type: new DataTypes.STRING(128),
        allowNull: true,
      },
    },
    {
      tableName: 'users',
      sequelize, // passing the `sequelize` instance is required
      createdAt: 'created_at',
      updatedAt: 'updated_at'
    },
  );
}

export function associate () {
  // Here we associate which actually populates out pre-declared `association` static and other methods.
  User.hasMany(Project, {
    sourceKey: 'id',
    foreignKey: 'ownerId',
    as: 'projects', // this determines the name in `associations`!
  });
  
  User.hasOne(Address, { sourceKey: 'id' });
}
```

```js
// app/model/project.ts
import {
  Optional,
  DataTypes,
  Model,
  Sequelize
} from 'sequelize';


interface ProjectAttributes {
  id: number;
  ownerId: number;
  name: string;
}

interface ProjectCreationAttributes extends Optional<ProjectAttributes, 'id'> {}

export class Project extends Model<ProjectAttributes, ProjectCreationAttributes>
  implements ProjectAttributes {
  public id!: number;
  public ownerId!: number;
  public name!: string;

  public readonly createdAt!: Date;
  public readonly updatedAt!: Date;
}

export function init (sequelize: Sequelize): void {
  Project.init(
    {
      id: {
        type: DataTypes.INTEGER.UNSIGNED,
        autoIncrement: true,
        primaryKey: true,
      },
      ownerId: {
        field: 'owner_id',
        type: DataTypes.INTEGER.UNSIGNED,
        allowNull: false,
      },
      name: {
        type: new DataTypes.STRING(128),
        allowNull: false,
      }
    },
    {
      sequelize,
      tableName: 'projects',
      createdAt: 'created_at',
      updatedAt: 'updated_at'
    },
  );
}
```

```js
// app/model/address.ts
import { Application } from 'egg';
import {
  Model,
  Sequelize,
  DataTypes,
} from 'sequelize';

interface AddressAttributes {
  userId: number;
  address: string;
}

// You can write `extends Model<AddressAttributes, AddressAttributes>` instead,
// but that will do the exact same thing as below
export class Address extends Model<AddressAttributes> implements AddressAttributes {
  public userId!: number;
  public address!: string;

  public readonly createdAt!: Date;
  public readonly updatedAt!: Date;
}

export function init (sequelize: Sequelize): void {
  Address.init(
    {
      userId: {
        field: 'user_id',
        type: DataTypes.INTEGER.UNSIGNED,
      },
      address: {
        type: new DataTypes.STRING(128),
        allowNull: false,
      },
    },
    {
      tableName: 'address',
      sequelize, // passing the `sequelize` instance is required
      createdAt: 'created_at',
      updatedAt: 'updated_at'
    },
  );
}

export function associate (app: Application): void {
  app.model.Address.belongsTo(app.model.User, { targetKey: 'id' });
}
```

```js
// app/route.ts
import { Application } from 'egg';

export default (app: Application) => {
  const { controller, router } = app;

  router.resources('test', '/api', controller.home);
};
```

Now you can use it in your controller:

```js
// app/controller/home.ts
import { Controller } from 'egg';

export default class HomeController extends Controller {
  public async index() {
    const { ctx } = this;

    const users = await ctx.model.User.findAll();

    ctx.body = {
      users
    };
  }

  public async show () {
    const { ctx } = this,
      { id } = ctx.params;

    const user = await ctx.model.User.findByPk(id);

    ctx.body = {
      user
    };
  }

  public async create () {
    const { ctx } = this;

    const user = await ctx.models.User.create({
      name: 'Zhangsan',
      preferredName: '张三'
    });

    ctx.body = {
      user
    };
  }
}
```

<!-- example here -->

## Questions & Suggestions

Please open an issue [here](https://github.com/humandetail/egg-ts-sequelize/issues).

## License

[MIT](LICENSE)
