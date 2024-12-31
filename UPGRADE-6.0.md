# Backwards compatibility breaking changes

## Removed features
- The Gated Video feature was removed as it was used only in the Legacy builder. See https://github.com/mautic/mautic/pull/14284
- The Froala editor got removed due to security issues of the old version we couldn't update due to licencing issues. It was used in the legacy builder only.

## BC breaks in the code
- Multiple method signatures changed to improve type coverage. Some forced by dependency updates, some in Mautic itself. Run `composer phpstan` when your plugin is installed to get the full list related to your plugin.
- `Mautic\PointBundle\Form\Type\GenericPointSettingsType` was removed. See https://github.com/mautic/mautic/pull/13904
- Changes necessary for https://symfony.com/blog/new-in-symfony-5-3-guard-component-deprecation, see https://github.com/mautic/mautic/pull/14219
    - `Mautic\ApiBundle\DependencyInjection\Factory\ApiFactory` was removed.
    - The `friendsofsymfony/oauth-server-bundle` package was replaced with a maintained fork `klapaudius/oauth-server-bundle`
    - The `lightsaml/sp-bundle` package was replaced with a maintained fork `javer/sp-bundle`
- Deprecated `Mautic\LeadBundle\Model\FieldModel::getUniqueIdentiferFields` and `Mautic\LeadBundle\Model\FieldModel::getUniqueIdentifierFields` were removed. Use `Mautic\LeadBundle\Field\FieldsWithUniqueIdentifier::getFieldsWithUniqueIdentifier` instead.
- The signature for the `Mautic\PluginBundle\Integration\AbstractIntegration::__construct()` had to be changed as the `SessionInterface` service no longer exists in Symfony 6. So it was removed from the constructor and session is being fetched from the `RequestStack` instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getDebugMode` use dependency injection instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getMauticBundles` use BundleHelper instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getPluginBundles` use BundleHelper instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getBundleConfig` use BundleHelper instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getUser` use UserHelper instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getTranslator` use Translator instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getRouter` use Router or UrlGeneratorInterface instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getLocalConfigFile` use dependency injection with KernelInterface, which will retrieve \AppKernel, then invoke getLocalConfigFile().
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getEnvironment` use dependency injection instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getIpAddress` use IpLookupHelper instead.
- Removed `Mautic\CoreBundle\Factory\MauticFactory::getTwig` use DI with the `\Twig\Environment` instead.
- Removed `Mautic\CampaignBundle\Entity::getEventsByChannel()` as unused and buggy. No replacement
- Removed `Mautic\CoreBundle\Test::createAnotherClient()` as unused. No replacement.
- Removed `Mautic\NotificationBundle\Entity::getLeadStats()` as unused and buggy. No replacment
- Removed `Mautic\WebhookBundle\Entity::removeOldLogs()` as it was deprecated. Use `removeLimitExceedLogs()` instead.
- Removed `Mautic\PageBundle\Entity::findByIds()` as unused and buggy. Use Doctrine's `findAllBy(['id' => [1,2]])` instead.
- Removed `Mautic\PluginBundle\Controller::getIntegrationCampaignsAction()` as unused and buggy together with JS function `Mautic.getIntegrationCampaigns`
- Removed `Mautic\CoreBundle\Tests\Functional\Service::class` as unused and testing 3rd party code instead of Mautic.
- Removed `Mautic\CoreBundle\Doctrine\TranslationMigrationTrait` as unused and deprecated.
- Removed `Mautic\CoreBundle\Doctrine\VariantMigrationTrait` as unused and deprecated.
- Removed `Mautic\IntegrationsBundle\Form\Type\NotBlankIfPublishedConstraintTrait` as unused.
- Removed `Mautic\IntegrationsBundle\Form\Type\Auth\BasicAuthKeysTrait` as unused.
- Removed `Mautic\IntegrationsBundle\Form\Type\Auth\Oauth1aTwoLeggedKeysTrait` as unused.
- Removed these services as the authentication system in Symfony 6 has changed and these services were using code that no longer existed.
    - `mautic.user.form_guard_authenticator` (`Mautic\UserBundle\Security\Authenticator\FormAuthenticator::class`)
    - `mautic.user.preauth_authenticator` (`Mautic\UserBundle\Security\Authenticator\PreAuthAuthenticator::class`)
    - `mautic.security.authentication_listener` (`Mautic\UserBundle\Security\Firewall\AuthenticationListener::class`)
- The `GrapesJsData` class was moved from `Mautic\InstallBundle\InstallFixtures\ORM` namespace to `MauticPlugin\GrapesJsBuilderBundle\InstallFixtures\ORM` as plugins should not be coupled with core bundles.


## Most notable changes required by Symfony 6

### Getting a value from request must be scalar

Meaning arrays cannot be returned with the `get()` method. Example of how to resolve it:
```diff
- $asset = $request->request->get('asset') ?? [];
+ $asset = $request->request->all()['asset'] ?? [];
```

### ASC contants replaced with enums in Doctrine
```diff
- $q->orderBy($this->getTableAlias().'.dateAdded', \Doctrine\Common\Collections\Criteria::DESC);
+ $q->orderBy($this->getTableAlias().'.dateAdded', \Doctrine\Common\Collections\Order::Descending->value);
```

### Creating AJAX requests in functional tests
```diff
- $this->client->request(Request::METHOD_POST, '/s/ajax', $payload, [], $this->createAjaxHeaders());
+ $this->setCsrfHeader(); // this is necessary only for the /s/ajax endpoints. Other ajax requests do not need it.
+ $this->client->xmlHttpRequest(Request::METHOD_POST, '/s/ajax', $payload);
```

### Logging in different user in functional tests
```diff
- $user = $this->loginUser('admin');
+ $user = $this->em->getRepository(User::class)->findOneBy(['username' => 'admin']);
+ $this->loginUser($user);
```

### Asserting successful response in functional tests
```diff
$this->client->request('GET', '/s/campaigns/new/');
- $response = $this->client->getResponse();
- Assert::assertTrue($response->isOk(), $response->getContent());
+ $this->assertResponseIsSuccessful();
```

### Session service doesn't exist anymore
Use Request to get the session instead.
```diff
- use Symfony\Component\HttpFoundation\Session\SessionInterface;
+ use Symfony\Component\HttpFoundation\RequestStack;
class NeedsSession
{
-   public function __construct(private SessionInterface $session) {}
+   public function __construct(private RequestStack $requestStack) {}

    public function doStuff()
    {
-       $selected = $this->session->get('mautic.category.type', 'category');
+       $selected = $this->requestStack->getSession()->get('mautic.category.type', 'category');
        // ...
    }
}
```