---
id: best
title: "Лучшие практики"
---

Это руководство - список лучших практик, которые мы собрали, и которые рекомендуем всем пользователям. Не воспринимайте это руководство как высеченную в камне неделимую истину, вы можете использовать только пару пунктов, если так будет правильно для вас.

**Feel free to suggest your best practices to the Verdaccio community**.

## Приватный репозиторий

Вы можете добавлять пользователей и определять, какие пользователи имеют доступ к пакетам.

Мы рекомендуем определить для ваших приватных пакетов префикс, например `local-*`, или скоуп `@my-company/*`, так что все ваши приватные пакеты будут выглядеть примерно так: `local-foo`. Таким образом вы отделите публичные пакеты от приватных.

    yaml
      packages:
        '@my-company/*':
          access: $all
          publish: $authenticated
         'local-*':
          access: $all
          publish: $authenticated
        '@*/*':
          access: $all
          publish: $authenticated
        '**':
          access: $all
          publish: $authenticated

Always remember, **the order of packages access is important**, packages are matched always top to bottom.

### Использование публичных пакетов с npmjs.org

If a package doesn't exist in the storage, the server will try to fetch it from npmjs.org. If npmjs.org is down, it serves packages from the cache pretending that no other packages exist. **Verdaccio will download only what's needed (requested by clients)**, and this information will be cached, so if the client requests the same thing a second time it can be served without asking npmjs.org for it.

**Пример:**

If you successfully request `express@4.0.1` from the server once, you'll be able to do it again (with all of it's dependencies) any time, even if npmjs.org is down. Though note that `express@4.0.0` will not be downloaded until it's actually needed by somebody. And if npmjs.org is offline, the server will say that only `express@4.0.1` (what's in the cache) is published, but nothing else.

### Переопределение публичных пакетов

Если вы хотите использовать модифицированную версию какого-либо публичного пакета `foo`, то просто опубликуйте его на вашем локальном сервере, и когда вы запустите команду `npm install foo`, **будет становлена именно ваша версия**.

Возможны две ситуации:

1. Вы хотите создать отдельный **форк** и остановить синхронизацию с публичной версией.
    
    If you want to do that, you should modify your configuration file so Verdaccio won't make requests regarding this package to npmjs anymore. Добавьте отдельную запись для этого пакета в `config.yaml` и удалите `npmjs` из списка `proxy`, затем перезапустите сервер.
    
    ```yaml
    packages:
      '@my-company/*':
        access: $all
        publish: $authenticated
        # comment it out or leave it empty
        # proxy:
    ```
    
    When you publish your package locally, **you should probably start with a version string higher than the existing package** so it won't conflict with that package in the cache.

2. You want to temporarily use your version, but return to the public one as soon as it's updated.
    
    Чтобы избежать конфликта версий, **вам нужно использовать свой пре-релизный суффикс для следующей версии**. Например, если публичный пакет имел версию 0.1.2, вам нужно опубликовать `0.1.3-my-temp-fix`.
    
    ```bash
    npm version 0.1.3-my-temp-fix
    npm publish --tag fix --registry http://localhost:4873
    ```
    
    В этом случае ваш пакет будет использоваться до тех пор, пока владелец пакета не опубликует версию `0.1.3`.

## Безопасность

Security starts in your environment. For such things we recommend reading **[10 npm Security Best Practices](https://snyk.io/blog/ten-npm-security-best-practices/)** and following the steps outlined there.

### Доступ к пакетам

By default all packages you publish in Verdaccio are accessible for all users. We recommend protecting your registry from external non-authorized users by updating the `access` property of your packages to `$authenticated`.

```yaml
  packages:
    '@my-company/*':
      access: $authenticated
      publish: $authenticated
    '@*/*':
      access: $authenticated
      publish: $authenticated
    '**':
      access: $authenticated
      publish: $authenticated
   ```

That way, **nobody can access your registry unless they are authorized, and private packages won't be displayed in the web interface**.

## Server

### Secured Connections

Using **HTTPS** is a common recommendation. For this reason we recommend reading the [SSL](ssl.md) section to make Verdaccio secure, or alternatively using an HTTPS [reverse proxy](reverse-proxy.md) on top of Verdaccio.

### Expiring Tokens

In `verdaccio@3.x` the tokens have no expiration date. For such reason we introduced in the next `verdaccio@4.x` the JWT feature [PR#896](https://github.com/verdaccio/verdaccio/pull/896)

```yaml
security:
  api:
    jwt:
      sign:
        expiresIn: 15d
        notBefore: 0
  web:
    sign:
      expiresIn: 7d
```

**Использование этой конфигурации изменит текущее поведение сервера и вы сможете управлять временим жизни токенов**.

Using JWT also improves the performance with authentication plugins. The old system will perform an unpackage and validate the credentials on every request, while JWT will rely on the token signature instead, avoiding the overhead for the plugin.

В качестве примечания, **npmjs токены не имеют ограничений по времени**.