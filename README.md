<h1 align="center">
    <a href="https://github.com/sellerartifact/accesscontrol"><img width="465" height="170" src="https://camo.githubusercontent.com/b87f2d6e5bff84ae927950f754580cfe3270540e8635aaa0236f4f9c6f369476/68747470733a2f2f7261772e6769746875622e636f6d2f6f6e7572792f616363657373636f6e74726f6c2f6d61737465722f61632d6c6f676f2e706e67" alt="AccessControl.js" /></a>
</h1>
<p align="center">
    <a href="https://snyk.io/test/github/sellerartifact/accesscontrol"><img src="https://snyk.io/test/github/sellerartifact/accesscontrol/badge.svg?style=flat-square" alt="Known Vulnerabilities" /></a>
    <a href="https://github.com/sellerartifact/accesscontrol/graphs/commit-activity"><img src="https://img.shields.io/maintenance/yes/2024.svg?style=flat-square" alt="Maintained" /></a>
    <br />
    <a href="https://www.npmjs.com/package/@sellerartifact/accesscontrol"><img src="http://img.shields.io/npm/v/accesscontrol.svg?style=flat-square" alt="npm" /></a>
    <a href="https://github.com/sellerartifact/accesscontrol"><img src="https://img.shields.io/github/release/sellerartifact/accesscontrol.svg?style=flat-square" alt="Release" /></a>
    <a href="https://www.npmjs.com/package/@sellerartifact/accesscontrol"><img src="https://img.shields.io/npm/dm/@sellerartifact/accesscontrol.svg?style=flat-square" alt="Downloads/mo." /></a>
    <a href="https://github.com/sellerartifact/accesscontrol/blob/master/LICENSE"><img src="http://img.shields.io/npm/l/accesscontrol.svg?style=flat-square" alt="License" /></a>
    <a href="https://www.typescriptlang.org"><img src="https://img.shields.io/badge/written%20in-%20TypeScript%20-6575ff.svg?style=flat-square" alt="TypeScript" /></a>
    <a href="https://accesscontrol-psi.vercel.app/?api=ac"><img src="https://img.shields.io/badge/documentation-click_to_read-c27cf4.svg?documentation=click_to_read&style=flat-square" alt="Documentation" /></a>
    <br />
    <sub>© 2024, invalid w (<b><a href="https://github.com/wangjue666">@wangjue666</a></b>).</sub>
</p>
<br />

# Why this repo ?

This repo is a fork of the original [accesscontrol](https://github.com/onury/accesscontrol), and the original repo is no longer maintained. I will try to maintain this repo and fix the issues and add new features.

This repo was rebuilt using [modern.js](https://github.com/web-infra-dev/modern.js)

### Role and Attribute based Access Control for Node.js

Many [RBAC][rbac] (Role-Based Access Control) implementations differ, but the basics is widely adopted since it simulates real life role (job) assignments. But while data is getting more and more complex; you need to define policies on resources, subjects or even environments. This is called [ABAC][abac] (Attribute-Based Access Control).

With the idea of merging the best features of the two (see this [NIST paper][nist-paper]); this library implements RBAC basics and also focuses on _resource_ and _action_ attributes.

<table>
  <thead>
    <tr>
      <th><a href="#installation">Install</a></th>
      <th><a href="#guide">Examples</a></th>
      <th><a href="#roles">Roles</a></th>
      <th><a href="#actions-and-action-attributes">Actions</a></th>
      <th><a href="#resources-and-resource-attributes">Resources</a></th>
      <th><a href="#checking-permissions-and-filtering-attributes">Permissions</a></th>
      <th><a href="#defining-all-grants-at-once">More</a></th>
      <th><a href="https://github.com/sellerartifact/accesscontrol/blob/master/docs/FAQ.md">F.A.Q.</a></th>
      <th><a href="https://onury.io/accesscontrol?api=ac">API Reference</a></th>
    </tr>
  </thead>
</table>

## Core Features

- Chainable, friendly API.
  e.g. `ac.can(role).create(resource)`
- Role hierarchical **inheritance**.
- Define grants **at once** (e.g. from database result) or **one by one**.
- Grant/deny permissions by attributes defined by **glob notation** (with nested object support).
- Ability to **filter** data (model) instance by allowed attributes.
- Ability to control access on **own** or **any** resources.
- Ability to **lock** underlying grants model.
- No **silent** errors.
- **Fast**. (Grants are stored in memory, no database queries.)
- Brutally **tested**.
- TypeScript support.

_In order to build on more solid foundations, this library (v1.5.0+) is completely re-written in TypeScript._

## Installation

with [**npm**](https://www.npmjs.com/package/@sellerartifact/accesscontrol): `npm i @sellerartifact/accesscontrol --save`
with [**yarn**](https://yarn.pm/@sellerartifact/accesscontrol): `yarn add @sellerartifact/accesscontrol`

## Guide

```js
const { AccessControl } = require('@sellerartifact/accesscontrol');
// or:
// import { AccessControl } from '@sellerartifact/accesscontrol';
```

### Basic Example

Define roles and grants one by one.

```js
const ac = new AccessControl();
ac.grant('user') // define new or modify existing role. also takes an array.
  .createOwn('video') // equivalent to .createOwn('video', ['*'])
  .deleteOwn('video')
  .readAny('video')
  .grant('admin') // switch to another role without breaking the chain
  .extend('user') // inherit role capabilities. also takes an array
  .updateAny('video', ['title']) // explicitly defined attributes
  .deleteAny('video');

let permission = ac.can('user').createOwn('video');
console.log(permission.granted); // —> true
console.log(permission.attributes); // —> ['*'] (all attributes)

permission = ac.can('admin').updateAny('video');
console.log(permission.granted); // —> true
console.log(permission.attributes); // —> ['title']
```

### Express.js Example

Check role permissions for the requested resource and action, if granted; respond with filtered attributes.

```js
const ac = new AccessControl(grants);
// ...
router.get('/videos/:title', function (req, res, next) {
  const permission = ac.can(req.user.role).readAny('video');
  if (permission.granted) {
    Video.find(req.params.title, function (err, data) {
      if (err || !data) return res.status(404).end();
      // filter data by permission attributes and send.
      res.json(permission.filter(data));
    });
  } else {
    // resource is forbidden for this user/role
    res.status(403).end();
  }
});
```

## Roles

You can create/define roles simply by calling `.grant(<role>)` or `.deny(<role>)` methods on an `AccessControl` instance.

- Roles can extend other roles.

```js
// user role inherits viewer role permissions
ac.grant('user').extend('viewer');
// admin role inherits both user and editor role permissions
ac.grant('admin').extend(['user', 'editor']);
// both admin and superadmin roles inherit moderator permissions
ac.grant(['admin', 'superadmin']).extend('moderator');
```

- Inheritance is done by reference, so you can grant resource permissions before or after extending a role.

```js
// case #1
ac.grant('admin')
  .extend('user') // assuming user role already exists
  .grant('user')
  .createOwn('video');

// case #2
ac.grant('user').createOwn('video').grant('admin').extend('user');

// below results the same for both cases
const permission = ac.can('admin').createOwn('video');
console.log(permission.granted); // true
```

Notes on inheritance:

- A role cannot extend itself.
- Cross-inheritance is not allowed.
  e.g. `ac.grant('user').extend('admin').grant('admin').extend('user')` will throw.
- A role cannot (pre)extend a non-existing role. In other words, you should first create the base role. e.g. `ac.grant('baseRole').grant('role').extend('baseRole')`

## Actions and Action-Attributes

[CRUD][crud] operations are the actions you can perform on a resource. There are two action-attributes which define the **possession** of the resource: _own_ and _any_.

For example, an `admin` role can `create`, `read`, `update` or `delete` (CRUD) **any** `account` resource. But a `user` role might only `read` or `update` its **own** `account` resource.

<table>
    <thead>
        <tr>
            <th>Action</th>
            <th>Possession</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td rowspan="2">
            <b>C</b>reate<br />
            <b>R</b>ead<br />
            <b>U</b>pdate<br />
            <b>D</b>elete<br />
            </td>
            <td>Own</td>
            <td>The C|R|U|D action is (or not) to be performed on own resource(s) of the current subject.</td>
        </tr>
        <tr>
            <td>Any</td>
            <td>The C|R|U|D action is (or not) to be performed on any resource(s); including own.</td>
        </tr>
    </tbody>
</table>

```js
ac.grant('role').readOwn('resource');
ac.deny('role').deleteAny('resource');
```

_Note that **own** requires you to also check for the actual possession. See [this](https://github.com/sellerartifact/accesscontrol/issues/14#issuecomment-328316670) for more._

## Resources and Resource-Attributes

Multiple roles can have access to a specific resource. But depending on the context, you may need to limit the contents of the resource for specific roles.

This is possible by resource attributes. You can use Glob notation to define allowed or denied attributes.

For example, we have a `video` resource that has the following attributes: `id`, `title` and `runtime`.
All attributes of _any_ `video` resource can be read by an `admin` role:

```js
ac.grant('admin').readAny('video', ['*']);
// equivalent to:
// ac.grant('admin').readAny('video');
```

But the `id` attribute should not be read by a `user` role.

```js
ac.grant('user').readOwn('video', ['*', '!id']);
// equivalent to:
// ac.grant('user').readOwn('video', ['title', 'runtime']);
```

You can also use nested objects (attributes).

```js
ac.grant('user').readOwn('account', ['*', '!record.id']);
```

## Checking Permissions and Filtering Attributes

You can call `.can(<role>).<action>(<resource>)` on an `AccessControl` instance to check for granted permissions for a specific resource and action.

```js
const permission = ac.can('user').readOwn('account');
permission.granted; // true
permission.attributes; // ['*', '!record.id']
permission.filter(data); // filtered data (without record.id)
```

See [express.js example](#expressjs-example).

## Defining All Grants at Once

You can pass the grants directly to the `AccessControl` constructor.
It accepts either an `Object`:

```js
// This is actually how the grants are maintained internally.
const grantsObject = {
  admin: {
    video: {
      'create:any': ['*', '!views'],
      'read:any': ['*'],
      'update:any': ['*', '!views'],
      'delete:any': ['*'],
    },
  },
  user: {
    video: {
      'create:own': ['*', '!rating', '!views'],
      'read:own': ['*'],
      'update:own': ['*', '!rating', '!views'],
      'delete:own': ['*'],
    },
  },
};
const ac = new AccessControl(grantsObject);
```

... or an `Array` (useful when fetched from a database):

```js
// grant list fetched from DB (to be converted to a valid grants object, internally)
const grantList = [
  {
    role: 'admin',
    resource: 'video',
    action: 'create:any',
    attributes: '*, !views',
  },
  { role: 'admin', resource: 'video', action: 'read:any', attributes: '*' },
  {
    role: 'admin',
    resource: 'video',
    action: 'update:any',
    attributes: '*, !views',
  },
  { role: 'admin', resource: 'video', action: 'delete:any', attributes: '*' },

  {
    role: 'user',
    resource: 'video',
    action: 'create:own',
    attributes: '*, !rating, !views',
  },
  { role: 'user', resource: 'video', action: 'read:any', attributes: '*' },
  {
    role: 'user',
    resource: 'video',
    action: 'update:own',
    attributes: '*, !rating, !views',
  },
  { role: 'user', resource: 'video', action: 'delete:own', attributes: '*' },
];
const ac = new AccessControl(grantList);
```

You can set grants any time...

```js
const ac = new AccessControl();
ac.setGrants(grantsObject);
console.log(ac.getGrants());
```

...unless you lock it:

```js
ac.lock().setGrants({}); // throws after locked
```

## Documentation

You can read the full [**API reference**][docs] with lots of details, features and examples.
And more at the [F.A.Q. section][faq].

## Change-Log

See [CHANGELOG][changelog].

## Contributing

Clone original project:

```sh
git clone https://github.com/sellerartifact/accesscontrol.git
```

Install dependencies:

```sh
npm install
```

Add tests to relevant file under [/test](test/) directory and run:

```sh
npm run build && npm run test
```

## License

[**MIT**][license].

[docs]: https://accesscontrol-psi.vercel.app/?api=ac
[faq]: https://accesscontrol-psi.vercel.app/?content=faq
[rbac]: https://en.wikipedia.org/wiki/Role-based_access_control
[abac]: https://en.wikipedia.org/wiki/Attribute-Based_Access_Control
[crud]: https://en.wikipedia.org/wiki/Create,_read,_update_and_delete
[nist-paper]: http://csrc.nist.gov/groups/SNS/rbac/documents/kuhn-coyne-weil-10.pdf
[changelog]: https://github.com/sellerartifact/accesscontrol/blob/master/CHANGELOG.md
[license]: https://github.com/sellerartifact/accesscontrol/blob/master/LICENSE
[modern.js]: https://github.com/web-infra-dev/modern.js/
