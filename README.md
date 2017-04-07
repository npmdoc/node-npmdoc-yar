# api documentation for  [yar (v8.1.2)](https://github.com/hapijs/yar#readme)  [![npm package](https://img.shields.io/npm/v/npmdoc-yar.svg?style=flat-square)](https://www.npmjs.org/package/npmdoc-yar) [![travis-ci.org build-status](https://api.travis-ci.org/npmdoc/node-npmdoc-yar.svg)](https://travis-ci.org/npmdoc/node-npmdoc-yar)
#### Cookie jar plugin for Hapi

[![NPM](https://nodei.co/npm/yar.png?downloads=true)](https://www.npmjs.com/package/yar)

[![apidoc](https://npmdoc.github.io/node-npmdoc-yar/build/screenCapture.buildNpmdoc.browser._2Fhome_2Ftravis_2Fbuild_2Fnpmdoc_2Fnode-npmdoc-yar_2Ftmp_2Fbuild_2Fapidoc.html.png)](https://npmdoc.github.io/node-npmdoc-yar/build/apidoc.html)

![npmPackageListing](https://npmdoc.github.io/node-npmdoc-yar/build/screenCapture.npmPackageListing.svg)

![npmPackageDependencyTree](https://npmdoc.github.io/node-npmdoc-yar/build/screenCapture.npmPackageDependencyTree.svg)



# package.json

```json

{
    "bugs": {
        "url": "https://github.com/hapijs/yar/issues"
    },
    "dependencies": {
        "hoek": "4.x.x",
        "statehood": "5.x.x",
        "uuid": "3.x.x"
    },
    "description": "Cookie jar plugin for Hapi",
    "devDependencies": {
        "boom": "4.x.x",
        "code": "4.x.x",
        "hapi": "15.x.x",
        "lab": "11.x.x",
        "npmignore": "0.2.x"
    },
    "directories": {},
    "dist": {
        "shasum": "302912c2e01d0f3be1bf3ff3b5d3d1e215530700",
        "tarball": "https://registry.npmjs.org/yar/-/yar-8.1.2.tgz"
    },
    "engines": {
        "node": ">=4.0.0"
    },
    "gitHead": "42470f853a95bd5bac1190cf8f96f307cbbb2b15",
    "homepage": "https://github.com/hapijs/yar#readme",
    "keywords": [
        "hapi",
        "plugin",
        "cookies",
        "jar",
        "session"
    ],
    "license": "BSD-3-Clause",
    "main": "lib/index.js",
    "maintainers": [
        {
            "name": "hueniverse",
            "email": "eran@hammer.io"
        },
        {
            "name": "markbradshaw",
            "email": "mark.bradshaw@gmail.com"
        }
    ],
    "name": "yar",
    "optionalDependencies": {},
    "readme": "ERROR: No README data found!",
    "repository": {
        "type": "git",
        "url": "git://github.com/hapijs/yar.git"
    },
    "scripts": {
        "test": "lab -a code -t 100 -L",
        "test-cov-html": "lab -a code -r html -o coverage.html",
        "update-npmignore": "npmignore"
    },
    "version": "8.1.2"
}
```



# <a name="apidoc.tableOfContents"></a>[table of contents](#apidoc.tableOfContents)

#### [module yar](#apidoc.module.yar)
1.  [function <span class="apidocSignatureSpan">yar.</span>register (server, options, next)](#apidoc.element.yar.register)



# <a name="apidoc.module.yar"></a>[module yar](#apidoc.module.yar)

#### <a name="apidoc.element.yar.register"></a>[function <span class="apidocSignatureSpan">yar.</span>register (server, options, next)](#apidoc.element.yar.register)
- description and source-code
```javascript
(server, options, next) => {

    // Validate options and apply defaults

    const settings = Hoek.applyToDefaults(internals.defaults, options);
    Hoek.assert(!settings.cookieOptions.encoding, 'Cannot override cookie encoding');
    const rawCookieOptions = Hoek.clone(settings.cookieOptions);
    settings.cookieOptions.encoding = 'iron';
    rawCookieOptions.encoding = 'none';
    if (settings.customSessionIDGenerator) {
        Hoek.assert(typeof settings.customSessionIDGenerator === 'function', 'customSessionIDGenerator should be a function');
    }

    // Configure cookie

    server.state(settings.name, settings.cookieOptions);

    // Decorate the server with yar object.

    const getState = () => {

        return {};
    };
    server.decorate('request', 'yar', getState, {
        apply: true
    });

    // Setup session store

    const cache = server.cache(settings.cache);

    // Pre auth

    server.ext('onPreAuth', (request, reply) => {

        // If this route configuration indicates to skip, do nothing.
        if (Hoek.reach(request, 'route.settings.plugins.yar.skip')) {
            return reply.continue();
        }

        const generateSessionID = () => {

            const id = settings.customSessionIDGenerator ? settings.customSessionIDGenerator(request) : Uuid.v4();
            Hoek.assert(typeof id === 'string', 'Session ID should be a string');
            return id;
        };

        // Load session data from cookie

        const load = () => {

            request.yar = Object.assign(request.yar, request.state[settings.name]);
            if (request.yar.id) {

                request.yar._isModified = false;
                if (!settings.errorOnCacheNotReady && !cache.isReady() && !request.yar._store) {
                    request.log('Cache is not ready: not loading sessions from cache');
                    request.yar._store = {};
                }
                if (request.yar._store) {
                    return decorate();
                }

                request.yar._store = {};
                return cache.get(request.yar.id, (err, value, cached) => {

                    if (err) {
                        return decorate(err);
                    }

                    if (cached && cached.item) {
                        request.yar._store = cached.item;
                    }

                    return decorate();
                });
            }

            request.yar.id = generateSessionID();
            request.yar._store = {};
            request.yar._isModified = settings.storeBlank;

            decorate();
        };

        const decorate = (err) => {

            if (request.yar._store._lazyKeys) {
                request.yar._isLazy = true;                 // Default to lazy mode if previously set
                request.yar._store._lazyKeys.forEach((key) => {

                    request.yar[key] = request.yar._store[key];
                    delete request.yar._store[key];
                });
            }

            request.yar.reset = () => {

                cache.drop(request.yar.id, () => {});
                request.yar.id = generateSessionID();
                request.yar._store = {};
                request.yar._isModified = true;
            };

            request.yar.get = (key, clear) => {

                const value = request.yar._store[key];
                if (clear) {
                    request.yar.clear(key);
                }

                return value;
            };

            request.yar.set = (key, value) => {

                Hoek.assert(key, 'Missing key');
                Hoek.assert(typeof key === 'string' || (typeof key === 'object' && value === undefined), 'Invalid yar.set() arguments
');

                request.yar._isModified = true;

                if (typeof key === 'string') {
                    // convert key of type string into an object, for consistency.
                    const holder = {};
                    holder[key] = value;
                    key = holder;
                }

                Object. ...
```
- example usage
```shell
...
Please note that there are other default cookie options that can impact your security.
Please look at the description of the cookie options below to make sure this is doing
what you expect.
*/

var server = new Hapi.Server();

server.register({
    register: require('yar'),
    options: options
}, function (err) { });
'''

## Password considerations
...
```



# misc
- this document was created with [utility2](https://github.com/kaizhu256/node-utility2)
