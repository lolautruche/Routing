# Symfony Chain Routing Bundle [![Build Status](https://secure.travis-ci.org/symfony-cmf/ChainRoutingBundle.png)](http://travis-ci.org/symfony-cmf/ChainRoutingBundle)


This bundle contains a replacement for the default Symfony Router. As the name
implies, the chain router registers a list of routers that it tries by
priority to match and generate routes. One of the routers in that chain can of
course be the default router so you can still use the standard way for some of
your routes.

Additionally, this bundle delivers useful router implementations. Currently,
there is the *DoctrineRouter* that routes based on doctrine database entities
or documents. These services are only made available when explicitly enabled in
the application configuration.


## Installation

If you just use the chain router, this bundle has no dependencies.
For the DoctrineRouter you need a doctrine implementation. Default provided
classes use phpcr-odm which you can install as described in
[symfony-cmf](https://github.com/symfony-cmf/symfony-cmf)


## ChainRouter

The ChainRouter can replace the default symfony routing system with a chain-
enabled implementation. It does not route anything on its own, but only loops
through all chained routers. To handle standard configured symfony routes, the
symfony default router is used.

### Configuration

In your app/config/config.yml, you can specify which router services you want
to use. If you do not specify the routers_by_id map at all, by default the
chain router will just load the built-in symfony router. When you specify the
routers_by_id list, you need to explicitly specify router.default unless you
don't want the configuration based default router.

The format is ```service_name: priority``` - the higher the priority number the
earlier this router service is asked to match a route or to generate a url.

    symfony_cmf_chain_routing:
        chain:
            routers_by_id:
                # enable the DoctrineRouter with high priority to allow overwriting configured routes with content
                symfony_cmf_chain_routing.doctrine_router: 200
                # enable the symfony default router with a lower priority
                router.default: 100
            # whether the chain router should replace the default router. defaults to true
            # if you set this to false, you will need to do somthing else to trigger your router
            # replace_symfony_router: true

### Loading routers with tagging

Your routers can automatically register, just add it as a service tagged with `router` and an optional `priority`.
The higher the priority, the earlier your router will be asked to match the route. If you do not specify the priority,
your router will come last.
If there are several routers with the same priority, the order between them is undetermined.

For example, the cmf router is loaded with

    <service id="my_namespace.my_router" class="%my_namespace.my_router_class%">
        <tag name="router" priority="300" />

See also [Symfony documentation for DependencyInjection tags.](http://symfony.com/doc/2.0/reference/dic_tags.html)


## Doctrine Router

This implementation of a router loads routes from a database. To read data, the
RouteRepositoryInterface is used. It can be easily implemented with doctrine.
The router works with the base UrlMatcher and UrlGenerator classes and only
adds loading routes from the database and the concept of referenced content.

The DoctrineRouter service is set up with a repository. See the configuration
section for how to change the route_repository_service.
This bundle comes with a route repository implementation for PHCPR-ODM. PHPCR
is well suited to the tree nature of the data. If you use PHPCR-ODM with a
route document like the one provided, you can just leave the repository service
at the default.

You will want to configure the controller resolvers that decide what controller
will be used to handle the request, to avoid hardcoding controller names into
your content.

The minimum configuration required to load the doctrine router is to have enabled: true
in your config.yml (if you do nothing about that, the doctrine router service will not
be loaded at all and you can use the chain router with your own routers):

    symfony_cmf_chain_routing:
        doctrine:
            enabled: true

### PHPCR-ODM repository service

The default repository loads the route at the path in the request and all
parent paths to allow for some of the path segments being parameters. If you
need a different way to load routes or for example never use parameters, you
can write your own repository implementation to optimize (see cmf_routing.xml
for how to configure the service).

### Match Process

* Ask the repository for Route documents that could match the requested url
* Build a route collection and let the UrlMatcher find a matching route
* If the defaults do not contain the field _controller, loop through the
    ControllerResolverInterface list to find the controller. If none of the
    resolver finds a controller, throw a ResourceNotFoundException
* If the route implements RouteObjectInterace and returns a non-null content,
    set it as request attribute with the name ``contentDocument``. (Use the
    constant DoctrineRouter::CONTENT_KEY in your code.)

Your controllers should expect the parameter $contentDocument in their
``Action`` methods if they are supposed to work with content referenced by the
routes.
See ``Symfony\Cmf\Bundle\ContentBundle\Controller\ContentController`` for an
example.

### Configuration

To configure the resolvers, you can specify mappings. Presence of each of the
mappings makes the DI container inject the respective resolver into the
DoctrineRouter.

The possible mappings are (in order of precedence):

* (Explicit controller): If there is a _controller set in getRouteDefaults(),
    it is used and no resolver is asked.
* Explicit Template: requires the route document to return a 'template'
    parameter in getRouteDefaults. The configured generic controller is
    returned by the resolver.
* Controller by alias: requires the route document to return a 'type' value in
    getRouteDefaults()
* Controller by class: requires the route document to return an object for
    getRouteContent(). The content document is checked for being instanceof the
    class names in the map and if matched that controller is returned.
    Instanceof is used instead of direct lookup to work with proxy classes.
* Template by class: requires the route document to return an object for
    getRouteContent(). The content document is checked for being instanceof the
    class names in the map and if matched that template will be set as
    'template' in the $defaults and return the configured generic controller.

    symfony_cmf_chain_routing:
        doctrine:
            enabled: true
            generic_controller: symfony_cmf_content.controller:indexAction
            controllers_by_alias:
                editablestatic: sandbox_main.controller:indexAction
            controllers_by_class:
                Symfony\Cmf\Bundle\ContentBundle\Document\StaticContent: symfony_cmf_content.controller::indexAction
            templates_by_class:
                Symfony\Cmf\Bundle\ContentBundle\Document\StaticContent: SymfonyCmfContentBundle:StaticContent:index.html.twig

            # the repository is responsible to load routes
            # for phpcr-odm, we mainly use this because it can map from url to repository path
            # an orm repository might need different logic. look at cmf_routing.xml for an example if you
            # need to define your own service
            route_repository_service: symfony_cmf_chain_routing.phpcrodm_route_repository

            # if you use the default doctrine router service, you can use this to customize
            # the root path for the phpcr-odm RouteRepository
            routing_repositoryroot: /cms/routes

To see some examples, please look at the [cmf-sandbox](https://github.com/symfony-cmf/cmf-sandbox)
and specifically the routing fixtures loading.


### RouteObjectInterface

Routes that implement this interface are linked to a content document. This
bundle provides a default implementation for phpcr-odm.
All routes still need to extend the base class Symfony\Component\Routing\Route

### Redirections

You can build redirections with the RedirectRoute document. It can redirect
either to an absolute URI, or to a named symfony route or to another
Route object that can be referenced in the repository.
The RedirectRoute can be handled by the RedirectController
TODO: see Configuration.php of this bundle. I could not figure out how to set
this mapping as a default mapping. Meanwhile, in order to do redirections, you
need to add an entry to your mapping in config.yml:

    controllers_by_class:
        Symfony\Cmf\Bundle\ChainRoutingBundle\Document\RedirectRoute:  symfony_cmf_chain_routing.redirect_controller:redirectAction

If you need to do something special, you can use your own controller, or also
replace the document with your own, as long as it implements RedirectRouteInterface.

### Routes and locales

You can use the _locale default value in a route to create one route per locale
that all reference the same multilingual content.
The DoctrineRouter respects the _locale when generating routes from content.
When resolving the route, the _locale gets into the request and is picked up
by the symfony locale system.

Routes should never be translatable documents, as one route document represents
one single url, and serving several translations under the same url is not a
good idea.


### Customize

You can add more ControllerResolverInterface implementations if you have a case
not handled by the provided ones.

If you use an odm / orm different to phpcr-odm, you probably need to specify
the class for the route entity (in phpcr-odm, the class is automatically
detected).
You might need to extend DoctrineRouter and overwrite findRouteForUrl to find
route objects by URLs in your database.

## Authors

* Filippo De Santis (p16)
* Henrik Bjornskov (henrikbjorn)
* Claudio Beatrice (omissis)
* Lukas Kahwe Smith (lsmith77)
* David Buchmann (dbu)
* [And others](https://github.com/symfony-cmf/ChainRoutingBundle/contributors)

The original code for the chain router was contributed by Magnus Nordlander.