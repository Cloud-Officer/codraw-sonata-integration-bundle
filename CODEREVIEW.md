# Code Review — codraw/sonata-integration-bundle

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **L2 (composer.json)** — added a `suggest` section to `composer.json` documenting the optional feature packages (`codraw/application`, `codraw/console`, `codraw/cron-job`, `codraw/entity-migrator`, `codraw/messenger`, `codraw/security`, `codraw/user-bundle`, `scheb/2fa-bundle`, `scheb/2fa-email`, `scheb/2fa-totp`) and added `codraw/cron-job` + `codraw/entity-migrator` to `require-dev` (their entity classes are imported by `Tests/DependencyInjection/*`). The `"php": ">=8.5"` constraint was deliberately left unbounded — the `>=8.5` style is the convention across all codraw packages. `composer validate --no-check-publish` passes. Open item: no declared dependency was removed; nothing was flagged as unused.
- **H4** — `User/Admin/Extension/RefreshUserLockExtension.php`: replaced the removed single-colon controller notation with `RefreshUserLockController::class.'::refreshUserLocksAction'`, un-breaking the `refresh-user-locks` route (previously a guaranteed 500).
- **H5** — `Console/Admin/ExecutionAdmin.php::createNewInstance()`: an unknown `?command=` query parameter now throws `NotFoundHttpException` (404) instead of fataling with a null dereference (500).
- **M1** — `Configuration/Admin/ConfigAdmin.php`: the reverse JSON model transformer now catches `\JsonException` and rethrows it as `TransformationFailedException`, so malformed JSON produces a form validation error instead of a 500.
- **L3** — `User/Controller/LoginController.php`, `User/Controller/AccountLockedController.php`: switched the deprecated `Symfony\Component\Routing\Annotation\Route` import to `Symfony\Component\Routing\Attribute\Route` (drop-in on Symfony 6.4).
- **L5** — `User/Block/UserCountBlock.php`: filter settings missing a `value` key no longer trigger an undefined-array-key warning (`$data['value'] ?? null`).

Not fixed on purpose (behavior-changing, need design/security decisions): H1, H2, H3 (authorization/CSRF/validation hardening), M2–M6, L1, L4, L6, L7.

### Validation pass (2026-07-20)

- `composer install` (CI flags) resolves cleanly with the updated `composer.json`; no constraint adjustment was needed.
- Full PHPUnit run against a dedicated MySQL database: 72 tests, 84 assertions, 0 failures (1 pre-existing skip: `DrawSonataIntegrationExtensionTest::testServiceDefinition` with null data, "No service to test"). No test expectations needed updating — none of the applied fixes broke or changed any tested behavior.
- PHPStan (level 5, `phpstan.dist.neon`): 19 errors, all verified pre-existing via `git stash` (identical error list with and without the applied fixes — mostly soft-dependency `class.notFound` on `App\*` defaults (M6), `Symfony\Component\Mailer\MailerInterface`, and Config `TreeBuilder` fluent-API inference). No new errors introduced; the empty `phpstan-baseline.neon` was left untouched.
- `markdownlint-cli2 --fix` normalized a missing trailing newline in this file; all Markdown files now lint clean.
- Pre-existing (not introduced here): composer autoload PSR-4 warning for `Tests/DependencyInjection/DrawSonataIntegrationExtensionCronJobEnabledTest.php` (namespace `DependencyInjection` instead of `Draw\Bundle\SonataIntegrationBundle\Tests\DependencyInjection`); the test still runs via PHPUnit directory discovery.
- No additional CODEREVIEW findings were fixed in this pass: the remaining items (H1–H3, M2–M6, L1, L4, L6, L7) live in code with no direct unit-test coverage.

## Overall Assessment

This bundle glues the draw/codraw framework components (console executions, cron jobs, messenger, entity migrator, configuration, user/2FA/lock management) into Sonata Admin. The code is small (~3.3k LOC), well organized by feature, and the dependency-injection layer is cleanly written with per-feature enable flags and soft-dependency detection. However, the review found a cluster of real security-relevant problems in the custom controller actions: several state-changing endpoints perform no Sonata authorization check at all, all of them are plain GET links with no CSRF protection, and the console "Execution" feature's command whitelist can be bypassed because the form relies on the client-side `readonly` HTML attribute. One admin-extension route uses the single-colon controller notation removed in Symfony 5 and appears to be broken. Test coverage is limited almost exclusively to the DI layer; none of the controllers, admins, or security-sensitive flows are tested.

Grade: **C** — good structure, but several genuine authorization/CSRF gaps and a broken route that should be fixed before this is relied on in production.

---

## Findings

### High

#### H1. Console command whitelist bypass — `readonly` attribute is not server-side protection

`Console/Admin/ExecutionAdmin.php:128-134` (form), `:170-192` (prePersist/postPersist)

```php
protected function configureFormFields(FormMapper $form): void
{
    $form
        ->add('command', null, ['attr' => ['readonly' => true]])
        ->add('commandName', null, ['attr' => ['readonly' => true]])
    ;
}
```

`readonly` is only an HTML attribute; the fields are still bound on submit. A user with `create` access on the Execution admin can POST any `commandName` value (dev tools or curl, with the normal CSRF token). `prePersist()` copies the submitted `commandName` into the input array and `postPersist()` executes it synchronously through `Symfony\Bundle\FrameworkBundle\Console\Application` with `--no-interaction` and `-vvv` forced. This defeats the whole point of the `CommandRegistry` whitelist configured in `draw_sonata_integration.console.commands`: any registered console command in the application becomes executable, including destructive ones that are only guarded by an interactive confirmation (e.g. `doctrine:fixtures:load`, which purges the database when `--no-interaction` is passed). Fields should be `disabled => true` (disabled fields are ignored on submit) and/or `prePersist()` should validate the submitted command name against `CommandRegistry`.

#### H2. Missing authorization checks on custom state-changing controller actions

- `Console/Controller/ExecutionController.php:35` — `acknowledgeAction()` mutates the execution state and persists it with no `checkAccess()`/`isGranted()` call.
- `Console/Controller/ExecutionController.php:52` — `reportAction()` (read-only, but still exposes data with no check).
- `CronJob/Controller/CronJobController.php:15` — `queueAction()` queues a cron job for forced execution with no check.
- `CronJob/Controller/CronJobExecutionController.php:14` — `acknowledgeAction()` mutates and flushes with no check.
- `Messenger/Controller/MessageController.php:16` — `retryAction()` dispatches a `RetryFailedMessageMessage` with no check.

Sonata only enforces permissions inside the base CRUD actions; custom actions must call `$this->admin->checkAccess('...')` themselves (compare `ExecutionController::myCreateAction()` at `Console/Controller/ExecutionController.php:20`, which does it correctly, and the User-module actions, which all do it). As written, any authenticated user who can pass the admin firewall can acknowledge executions, force-queue cron jobs (i.e. trigger command execution), and retry failed messenger messages, regardless of their Sonata role grants. Each action needs an access mapping entry (via `getAccessMapping()` / admin `protected $accessMapping`) and an explicit `checkAccess()` call.

#### H3. State-changing operations via GET with no CSRF protection

Templates: `Resources/views/CronJob/CronJob/list__action_queue.html.twig`, `Resources/views/CronJob/CronJobExecution/list__action_acknowledge.html.twig`, `Resources/views/Console/Execution/button_acknowledge.html.twig`, `Resources/views/Messenger/Message/list__action_retry.html.twig`, `Resources/views/User/Buttons/disable-2fa.html.twig`, `unlock_button.html.twig`, `refresh_user_locks_button.html.twig`, `request_password_change_button.html.twig`.

All of these render plain `<a href>` links, and the corresponding controllers (`queueAction`, both `acknowledgeAction`s, `retryAction`, `disable2faAction` in `User/Controller/TwoFactorAuthenticationController.php:104`, `UnlockUserAction`, `RequestPasswordChangeAction`, `RefreshUserLockController::refreshUserLocksAction`) accept GET and perform writes without any CSRF token validation. A third-party page can trigger these with `<img src="https://admin.example.com/admin/.../1/disable-2fa">` against a logged-in admin — notably **disabling another user's 2FA**, unlocking accounts, or force-queuing cron jobs. Sonata's own destructive actions (delete, batch) use POST + CSRF token; these custom actions should do the same (or at minimum validate a token passed in the query string).

#### **[FIXED]** H4. Broken controller reference — single-colon notation removed in Symfony 5

`User/Admin/Extension/RefreshUserLockExtension.php:24`

```php
['_controller' => RefreshUserLockController::class.':refreshUserLocksAction']
```

The `service:method` single-colon controller syntax was deprecated in Symfony 4.1 and removed in 5.0; on Symfony 6.4 (the only supported line per composer.json) the resolver treats the whole string as a service id/class name and throws (`The controller ... does neither exist as service nor as class`), so hitting the `refresh-user-locks` route produces a 500. Every other route in the bundle correctly uses `::` (e.g. `User/Extension/TwoFactorAuthenticationExtension.php:98,104`). Should be `RefreshUserLockController::class.'::refreshUserLocksAction'`. The absence of any test on this route is why it went unnoticed.

#### **[FIXED]** H5. Null dereference on user-supplied `command` query parameter

`Console/Admin/ExecutionAdmin.php:42-45`

```php
if ($commandName = $this->getRequest()->get('command')) {
    $command = $this->commandFactory->getCommand($commandName);
    $execution->setCommand($command->getName());
```

`CommandRegistry::getCommand()` returns `?Command` (`Console/CommandRegistry.php:25-28`). Any request to the create page with an unknown `?command=foo` triggers a fatal `Error: Call to a member function getName() on null` → 500. The parameter is directly user-controlled. A null-check with a 404/flash (or fallback to the selection screen) is needed.

### Medium

#### **[FIXED]** M1. Invalid JSON in ConfigAdmin form causes a 500 instead of a validation error

`Configuration/Admin/ConfigAdmin.php:34-37`

The reverse model transformer uses `json_decode((string) $data, true, 512, \JSON_THROW_ON_ERROR)`. The Form component only converts `TransformationFailedException` into a form error; a `JsonException` propagates and produces an exception page. Typing malformed JSON into a 20-row free-text textarea is entirely normal admin use. Catch `\JsonException` and rethrow as `TransformationFailedException` so `invalid_message` is displayed.

#### M2. Synchronous, unbounded, error-swallowing command execution inside the web request

`Console/Admin/ExecutionAdmin.php:183-192`

`postPersist()` runs the console command in-process via `Application::run()` with a `BufferedOutput` that is discarded and a return code that is ignored (`Application` also catches exceptions internally by default). Long-running commands block the HTTP request until the PHP timeout kills them (potentially leaving the Execution row in a misleading state), failures are invisible to the admin user, and the command runs with the web server's memory/time limits and privileges. Dispatching the execution asynchronously (messenger) — or at minimum checking the exit code, setting a state on failure, and imposing a timeout — would be much safer.

#### M3. 2FA enablement trusts a client-supplied TOTP secret and mutates the managed entity before verification

`User/Controller/TwoFactorAuthenticationController.php:51-63` and `:74-78`

The TOTP secret round-trips through a hidden form field (`User/Form/Enable2faForm.php:31`) and on submit is written to the managed user entity (`$user->setTotpSecret($enable2fa->totpSecret)`) *before* `checkCode()` succeeds. If the code check fails the entity is left dirty with an attacker-chosen secret; any unrelated `flush()` during the remainder of the request (event listener, feed writer, etc.) would persist it. The same in-memory mutation happens on the initial GET (`:75`). Additionally, since the secret is client-supplied, a user with EDIT access on *another* user can install a secret they know on that account via this endpoint (access mapping is `EDIT` — `User/Extension/TwoFactorAuthenticationExtension.php:25`), silently gaining the ability to pass that user's second factor. Prefer storing the pending secret in the session server-side, and only calling `setTotpSecret()` after successful verification.

#### M4. No throttling on the 2FA resend-code endpoint

`User/Action/TwoFactorAuthenticationResendCodeAction.php:19-30`, route `Resources/config/routes.yaml`

`__invoke()` regenerates and emails a new code on every hit with no rate limiter. An attacker in the 2FA phase (password known) can flood the victim's mailbox and keep regenerating codes. Also, if the route is reachable without a (partial) authentication token, `$security->getUser()` returns null and the `LogicException` at `:24` produces a 500 instead of a redirect/403. A rate limiter (e.g. Symfony RateLimiter, as scheb/2fa suggests for resend actions) and a graceful anonymous handling are advisable.

#### M5. Password change without current-password verification

`User/Controller/LoginController.php:95-125`, `User/Form/ChangePasswordForm.php`

`changePasswordAction()` lets any logged-in user set a new password by submitting only the new password twice — no current password, no `UserPassword` constraint, no re-authentication. Combined with the fact that `needPasswordChange()` returns `true` for any user when no `t` query parameter is present (`:133-134`), the form is always reachable. A hijacked session (or an unlocked workstation) can be converted into permanent account takeover. Requiring the current password (`Symfony\Component\Validator\Constraints\UserPassword`) for non-forced changes is the standard mitigation.

#### M6. Bundle configuration defaults reference `App\` application classes

`DependencyInjection/Configuration.php:5-6`, `:150` (`App\Entity\MessengerMessage`), `:167` (`App\Sonata\Admin\UserAdmin`)

A reusable bundle hard-codes application-namespace class names as config defaults. In any app whose classes live elsewhere, the defaults are silently wrong (the messenger admin would be registered against a non-existent entity class; `user_admin_code` would point to a non-existent admin). Because these are `::class` constants on non-loaded classes they don't error at compile time, so the failure surfaces later and confusingly. These defaults should be required config keys (with a clear "you must set this" message) or resolved from the corresponding component's configuration.

### Low

#### L1. Dead loader / unused parameter in the DI extension

`DependencyInjection/DrawSonataIntegrationExtension.php:48` builds a `PhpFileLoader` pointed at `Resources/config`, which contains no PHP service files (only `routes.yaml`); the `$loader` argument is threaded through every `configure*()` method and never used.

#### **[FIXED]** L2. composer.json constraints

`composer.json` — `"php": ">=8.5"` has no upper bound (allows a future PHP 9 with breaking changes); the conventional `^8.5` (or `>=8.5 <9`) is safer. Several classes in `src` hard-reference packages that appear only in `require-dev` (`scheb/2fa-bundle` in `TwoFactorAuthenticationController`, `codraw/messenger`, `codraw/user-bundle`, `codraw/console`, `codraw/cron-job` types across the DI Configuration); that is a deliberate soft-dependency design, but a `suggest` section documenting which optional packages unlock which feature would help consumers.

#### **[FIXED]** L3. Deprecated `Symfony\Component\Routing\Annotation\Route` import

`User/Controller/LoginController.php:18`, `User/Controller/AccountLockedController.php:6` — the `Annotation\Route` alias is deprecated as of Symfony 7; since 6.4 the attribute lives in `Symfony\Component\Routing\Attribute\Route`. Trivial forward-compat fix.

#### L4. Magic sentinel date in the messenger voter

`Messenger/Security/CanShowMessageVoter.php:20` — `'9999-12-31' === $subject->getDeliveredAt()?->format('Y-m-d')` re-encodes the Doctrine-transport "handled/rejected" sentinel as a string literal with no named constant or comment; if the transport's sentinel changes this silently stops matching. Extract a documented constant.

#### **[FIXED]** L5. `UserCountBlock` assumes every filter setting has a `value` key

`User/Block/UserCountBlock.php:31-33` — `$data['value']` raises an undefined-key error for a block configured with a filter entry missing `value`. Config-time input, but a `?? null` would make it robust.

#### L6. Messenger list query semantics worth double-checking

`Messenger/Admin/MessengerMessageAdmin.php:135-149` — `configureQuery()` restricts the list to `expiresAt <= now OR expiresAt IS NULL`, i.e. it *shows* already-expired messages and *hides* messages with a future expiry. If `expiresAt` marks message TTL, the comparison looks inverted; if it is used as an in-flight/redelivery marker this is intentional. Worth a clarifying comment either way, since the reader's first assumption is the opposite.

#### L7. Inconsistent hook signatures

`Console/Admin/ExecutionAdmin.php:152` declares `configureActionButtons(array $buttonList, $action, $object = null)` as `public` with untyped parameters, while the sibling admins (`CronJob/Admin/CronJobAdmin.php:115`, `Messenger/Admin/MessengerMessageAdmin.php:151`) use the proper `protected ... (array, string, ?object)` signature. Harmless today, but the loose variant will break first on a Sonata signature tightening.

---

## Strengths

- **Clean, feature-modular DI layer.** Each integration (configuration, console, cron job, entity migrator, messenger, user/2FA/lock) is independently toggleable, and `Configuration::canBe()` neatly flips between `canBeEnabled`/`canBeDisabled` depending on whether the optional dependency is installed.
- **Fail-fast validation of 2FA prerequisites** at container build time (`DrawSonataIntegrationExtension.php:311-330`): missing SchebTwoFactorBundle or a user entity not implementing `TwoFactorAuthenticationUserInterface` produce clear exceptions instead of runtime failures.
- **The User-module actions get authorization right**: `enable2faAction`/`disable2faAction`, `UnlockUserAction`, `RequestPasswordChangeAction`, and `RefreshUserLockController` all call `checkAccess()` and declare proper access mappings with meaningful permissions (`MASTER`, `UNLOCK`, `REQUEST_PASSWORD_CHANGE`).
- **Sensible admin ergonomics**: default sort orders, choice filters built from entity state constants, conditional action buttons (`retry` only for failed messages, `acknowledge` only when `canBeAcknowledged()`), and the create route removed entirely when no console commands are whitelisted (`ExecutionAdmin::configureRoutes`).
- **Empty phpstan baseline** — no suppressed static-analysis debt.
- Well-organized translations and templates per feature, with translation domains wired through config.

---

## Test Coverage

Coverage is narrow and almost entirely structural:

- **DependencyInjection (good):** `Tests/DependencyInjection/` contains a `ConfigurationTest` plus one extension test per feature flag (configuration, console, cron job, messenger, user, user-lock, 2FA, 2FA+email), verifying which service definitions are registered for each toggle. This is the best-covered area of the bundle.
- **Actions (minimal):** a single unit test, `Tests/User/Action/TwoFactorAuthenticationResendCodeActionTest.php`, covering the happy path of the resend-code action.
- **Untested:** every Sonata Admin class (`ExecutionAdmin` — including the security-sensitive `prePersist`/`postPersist` command execution and `createNewInstance` — `CronJobAdmin`, `CronJobExecutionAdmin`, `MessengerMessageAdmin`, `ConfigAdmin`, `UserLockAdmin`, migrator admins), every controller (`ExecutionController`, `CronJobController`, `CronJobExecutionController`, `MessageController`, `LoginController`, `TwoFactorAuthenticationController`, `RefreshUserLockController`), the `CanShowMessageVoter`, `AdminLoginAuthenticator`/`AdminLoginFactory`, all admin extensions, forms, the Twig runtime, and `UserCountBlock`.

The absence of even smoke-level functional tests on the custom routes is what allowed the broken `refresh-user-locks` controller reference (H4) and the missing access checks (H2) to go unnoticed. Priority additions: a functional test per custom route asserting both behavior and access denial for an under-privileged user, plus unit tests for `ExecutionAdmin::createNewInstance`/`prePersist` and the `ConfigAdmin` JSON transformer.
