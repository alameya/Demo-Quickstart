This quickstart guide will show you the basics of developing with PHPixie and it's components.
If you have some experience with any other PHP framework soon you will feel right at home.

## Installing

First, install Composer if you don't have it already and run the following command:

```
php composer.phar create-project phpixie/project your_project_folder 3.*-dev
```

If you are on Windows create a symlink from the `/bundles/app/web` folder to `/web/bundles/app`.
For Linux and Mac users the symlink should work automatically.

Your HTTP server should point to the `/web` directory. In case you use Nginx you will also require this rule:

```
location / {
            try_files $uri $uri/ /index.php;
}
```

Now visit http://localhost/ in your browser and you should see a greeting.

## Bundles

PHPixie 3 supports organizing your project into bundles to make it easy to reuse your code in other projects.
For example a user system that handles login and registration could be organized into a bundle and used in a 
different project.

When you create PHPixie project it comes preconfigured with a single `app` bundle in the `/bundles/app` folder.
Until your project gets more bundles you will rarely need to do anything outside this one directory.

## Processors

A familiar concept of MVC Controllers has been greatly extended in PHPixie, it now allows nesting, composition and extended flexibility.
But you can still achieve a Controller-like behavior, which is actually used by the `Hello` in the `app` bundle that showed us the greeting.

Let's now create a new `Quickstart` processor to use with the tutorial:

```php?start_inline=1
// bundles/app/src/Project/App/HTTPProcessors/Quickstart.php

namespace Project\App\HTTPProcessors;

use PHPixie\HTTP\Request;

// we extend a class that allows Controller-like behavior
class Quickstart extends \PHPixie\DefaultBundle\Processor\HTTP\Actions
{
    /**
     * The Builder will be used to access
     * various parts of the framework later on
     * @var Project\App\HTTPProcessors\Builder
     */
    protected $builder;
    
    public function __construct($builder)
    {
        $this->builder = $builder;
    }
    
    // This is the default action
    public function defaultAction(Request $request)
    {
        return "Quikcstart tutorial"
    }
    
    //We will be adding methods here in a moment
}
```

And now register it with the bundle:

```php?start_inline=1
// bundles/app/src/Project/App/HTTPProcessors.php

//...
    protected function buildQuickstartProcessor()
    {
        return new HTTPProcessors\Quickstart($this);
    }
//...
```

Accessing http://localhost/quickstart/ now will yield a "Quickstart tutorial" message.

Now we are all set up to try out the framework!

## Routing

A popular use case is to have URLs like `/quickstart/view/4` including a name or an id of some item.
First let's create an appropriate action in our processor:

```php?start_inline=1
// bundles/app/src/Project/App/HTTPProcessors/Quickstart.php

//...
    public function viewAction(Request $request)
    {
        //Output the 'id' parameter
        return $request->attributes()->get('id');
    }
    
//...
}

Now we must also configure a new route to support the `id` parameter.
For that let's first look at the route configuration file:

```php?start_inline=1
// bundles/app/assets/config/routeResolver

return array(
    //this allows us to group routes into one
    'type'      => 'group',
    'resolvers' => array(
        
        //...We will add new routes here..
        
        //The 'default' route
        'default' => array(
        //this type of route does pattern matching
            'type'     => 'pattern',
            
            //brackets mean that the part in them is optional
            'path'     => '(<processor>(/<action>))',
            
            //Default set of parameters to use
            //E.g. if the url is simply /hello
            //The 'action' parameter will default to 'greet'
            'defaults' => array(
                'processor' => 'hello',
                'action'    => 'greet'
            )
        )
    )
);
```

The route we want to add would look like this:
```php?start_inline=1
'view' => array(
    'type'     => 'pattern',
    
    //Since the id parameter is mandatory
    //we don't wrap it in brackets
    'path'     => 'quickstart/view/<id>',
    'defaults' => array(
        'processor' => 'quickstart',
        'action'    => 'view'
    )
)
```

> Routes are tried one by one until a match is found.
> So it is important to put specific routes before more general ones in
> the config file.

Now navigatie to http://localhost/quickstart/view/5 and you should see '5' as a response.

As a quick example of some more advanced routing features here is what you could to to prefix multiple routes
with a common pattern. Don't worry if it looks a bit complicated, you don't need it for now:

```php?start_inline=1
array(
    
    //Define a common prefix for routes
    'type'      => 'prefix',
    'pattern'   => 'user/<userId>/',
    'resolver' => array(
        'type'      => 'group',
        'resolvers' => array(
        
            //would handle /user/5/friends to Friends::userFriends()
            'friends' => array(
                'pattern'  => 'friends',
                'defaults' => array(
                    'processor' => 'friends',
                    'action'    => 'usersFriends'
                )
            ),
            
            //would handle /user/5/friends to Profile::userProfile()
            'profile' => array(
                'pattern'  => 'profile',
                'defaults' => array(
                    'processor' => 'profile',
                    'action'    => 'userProfile'
                )
            )
        )
    )
);
````

Such an approach allows you to avoid specifying redundant parts in your routes and also
makes changing them easier and less error-prone.

## Input and output

As you probably already noticed each action takes a `Request` as a parameter and returns a response.
Accessing request data can be done simply by:

```php?start_inline=1
//$_GET['name'] 
$request->query()->get('name');

//$_POST['name'] 
$request->data()->get('name');

//Getting a routing attribute
$request->attributes()->get('name');
```

And now something more advanced:

```
$data = $request->data();

//Providing a default value
$data->get('name', 'Trixie');

//Throw an exception if 'name' is missing
$data->getRequired('name');

//Accessing a nested field
$data->get('users.pixie.name');

//You can also 'slice' the data to avoid long paths
$pixie = $data->slice('users.pixie');
$pixie->get('name');

//Getting data as array
$data->get();

//Getting all set keys
$data->keys();

//You can also iterate over it directly
foreach($data as $key => $value) {

}

//If you like this syntax take a look at the phpixie/slice library
//You can use it with any other array-like data
```

> JSON requests are also automatically parsed into $request->data()

As for the output it's even simpler:

```php?start_inline=1
//A simple string message
return 'hello';

//To build a properly encoded JSON reponse
//just return any array or object
return array('success' => true);

//Or build custom responses
//Using the HTTP library
$http = $this->builder->components()->http();
$httpResponses = $http->responses();

//Redirect
return $httpResponses->redirect('http://phpixie.com/');

//Set custom status and headers
return $httpResponses->stringResponse('Not found', $headers = array(), 404);

//Initialize a file download
return $httpResponses->downloadFile('pixie.jpg', 'image/png', $filePath);

//Initialize a file download from string
//Usefull for CSVs
return $httpResponses->download('report.csv', 'text/csv', $contents);
```
